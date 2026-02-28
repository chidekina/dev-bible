# Message Queues

## Overview

Message queues decouple the components of a system by allowing one service to send work to another without waiting for it to complete. The sender adds a message to the queue and moves on; a worker picks it up asynchronously. This unlocks horizontal scaling of background work, resilience to downstream failures, and the ability to absorb traffic spikes without losing data. This chapter covers queue fundamentals, BullMQ in practice, dead-letter queues, and common patterns.

---

## Prerequisites

- Node.js and TypeScript async patterns
- Basic Redis knowledge (BullMQ uses Redis as a backend)
- Track 03 — Backend: understanding of API-to-background-job patterns

---

## Core Concepts

### Why queues?

**Synchronous flow (without queue):**
```
User → API → send email → wait 2 seconds → respond to user
```

**Asynchronous flow (with queue):**
```
User → API → enqueue "send email" → respond immediately (fast)
                    ↓
             Worker → send email (in background, no timeout pressure)
```

Benefits:
- **Resilience:** if the email service is down, the job waits in the queue and is retried
- **Scalability:** add more workers to process more jobs in parallel
- **Decoupling:** the API does not know or care about the email implementation
- **Traffic smoothing:** sudden spike of 10,000 signups? Queue absorbs the spike; workers process at their own rate

### Key concepts

| Concept | Definition |
|---------|-----------|
| Producer | The component that adds messages to the queue |
| Consumer / Worker | The component that reads and processes messages |
| Queue | The ordered list of messages waiting to be processed |
| Dead-letter queue (DLQ) | Where messages go after exhausting all retry attempts |
| Acknowledgment | Worker signals "I processed this message" — removes it from queue |
| At-least-once delivery | The queue guarantees every message is delivered at least once (may be duplicated) |
| Idempotency | Processing the same message twice produces the same result as processing it once |

---

## Hands-On Examples

### BullMQ setup

```bash
npm install bullmq
```

```typescript
// src/lib/queue.ts
import { Queue, Worker, QueueEvents } from 'bullmq';
import { redis } from './redis.js';

const connection = { host: process.env.REDIS_HOST, port: parseInt(process.env.REDIS_PORT ?? '6379') };

// Define job types
export interface EmailJobData {
  to: string;
  subject: string;
  template: string;
  variables: Record<string, string>;
}

export interface InvoiceJobData {
  orderId: string;
  userId: string;
}

// Create queues
export const emailQueue = new Queue<EmailJobData>('email', { connection });
export const invoiceQueue = new Queue<InvoiceJobData>('invoice', { connection });
```

### Adding jobs (producer)

```typescript
// src/routes/auth.ts
import { emailQueue } from '../lib/queue.js';

fastify.post('/auth/register', async (request, reply) => {
  const { email, password } = RegisterSchema.parse(request.body);

  const user = await db.user.create({
    data: { email, passwordHash: await hashPassword(password) },
  });

  // Fire-and-forget — add to queue and return immediately
  await emailQueue.add(
    'welcome-email',
    {
      to: user.email,
      subject: 'Welcome!',
      template: 'welcome',
      variables: { name: user.name ?? user.email },
    },
    {
      attempts: 3,               // retry up to 3 times on failure
      backoff: { type: 'exponential', delay: 2000 }, // 2s, 4s, 8s
      removeOnComplete: { count: 100 },  // keep last 100 completed jobs
      removeOnFail: { count: 500 },      // keep last 500 failed jobs
    }
  );

  return reply.status(201).send({ id: user.id, email: user.email });
});
```

### Worker (consumer)

```typescript
// src/workers/email.worker.ts
import { Worker, UnrecoverableError } from 'bullmq';
import { EmailJobData } from '../lib/queue.js';
import { sendEmail } from '../lib/mailer.js';
import { logger } from '../lib/logger.js';

const connection = { host: process.env.REDIS_HOST, port: parseInt(process.env.REDIS_PORT ?? '6379') };

const emailWorker = new Worker<EmailJobData>(
  'email',
  async (job) => {
    logger.info({ jobId: job.id, to: job.data.to }, 'Processing email job');

    try {
      await sendEmail(job.data);
      logger.info({ jobId: job.id }, 'Email sent successfully');
    } catch (error) {
      // Distinguish permanent failures from transient ones
      if (error instanceof InvalidEmailError) {
        // Don't retry — the email address is invalid
        throw new UnrecoverableError(`Invalid email: ${job.data.to}`);
      }
      // Transient error — let BullMQ retry with backoff
      throw error;
    }
  },
  {
    connection,
    concurrency: 5, // process up to 5 emails in parallel
  }
);

emailWorker.on('completed', (job) => {
  logger.info({ jobId: job.id }, 'Email job completed');
});

emailWorker.on('failed', (job, error) => {
  logger.error({ jobId: job?.id, error: error.message }, 'Email job failed');
});
```

### Dead-letter queue pattern

```typescript
// Capture jobs that have exhausted all retry attempts
import { QueueEvents } from 'bullmq';

const emailQueueEvents = new QueueEvents('email', { connection });

emailQueueEvents.on('failed', async ({ jobId, failedReason }) => {
  const job = await emailQueue.getJob(jobId);
  if (!job) return;

  const maxAttempts = job.opts.attempts ?? 1;
  if (job.attemptsMade >= maxAttempts) {
    // Move to DLQ
    logger.error(
      { jobId, data: job.data, reason: failedReason },
      'Job moved to dead-letter queue'
    );

    await db.failedJob.create({
      data: {
        queueName: 'email',
        jobId,
        data: job.data as Record<string, unknown>,
        error: failedReason,
        failedAt: new Date(),
      },
    });
  }
});
```

### Job scheduling (delayed and recurring jobs)

```typescript
// Delayed job — process in the future
await emailQueue.add(
  'reminder',
  { to: user.email, subject: '7-day trial ending', template: 'trial-ending', variables: {} },
  {
    delay: 7 * 24 * 60 * 60 * 1000, // 7 days from now
    attempts: 3,
  }
);

// Recurring job — cron syntax
import { Queue } from 'bullmq';

const reportQueue = new Queue('reports', { connection });

// Generate weekly report every Monday at 8am
await reportQueue.add(
  'weekly-report',
  { type: 'weekly' },
  {
    repeat: { cron: '0 8 * * 1' }, // cron: minute hour day month weekday
    attempts: 3,
  }
);
```

### Idempotent job processing

Because queues deliver at-least-once, always write idempotent handlers:

```typescript
// Bad — not idempotent. If this runs twice, user gets charged twice.
async function processPayment(job: Job<PaymentJobData>) {
  await stripe.charges.create({ amount: job.data.amount, source: job.data.token });
}

// Good — idempotent using an idempotency key
async function processPayment(job: Job<PaymentJobData>) {
  const idempotencyKey = `payment:${job.data.orderId}`;

  // Check if we already processed this
  const existing = await db.payment.findUnique({
    where: { orderId: job.data.orderId },
  });

  if (existing) {
    logger.info({ orderId: job.data.orderId }, 'Payment already processed — skipping');
    return existing;
  }

  // Pass idempotency key to Stripe — prevents double charge even if Stripe is called twice
  const charge = await stripe.charges.create(
    { amount: job.data.amount, source: job.data.token },
    { idempotencyKey }
  );

  return db.payment.create({
    data: { orderId: job.data.orderId, stripeChargeId: charge.id, amount: job.data.amount },
  });
}
```

### BullMQ Dashboard setup (Bull Board)

```typescript
import { createBullBoard } from '@bull-board/api';
import { BullMQAdapter } from '@bull-board/api/bullMQAdapter.js';
import { FastifyAdapter } from '@bull-board/fastify';

const serverAdapter = new FastifyAdapter();

createBullBoard({
  queues: [new BullMQAdapter(emailQueue), new BullMQAdapter(invoiceQueue)],
  serverAdapter,
});

serverAdapter.setBasePath('/admin/queues');

await fastify.register(serverAdapter.registerPlugin(), { prefix: '/admin/queues' });
// Visit http://localhost:3000/admin/queues to see job status
```

---

## Common Patterns & Best Practices

- **Keep job payloads small** — store references (IDs) in the queue, fetch data in the worker
- **Make all workers idempotent** — assume any job can run twice
- **Use separate queues for different priorities** — email notifications vs. payment processing
- **Set appropriate concurrency per queue** — CPU-bound: 1 per core; I/O-bound: 10–50
- **Monitor queue depth** — a growing queue means workers cannot keep up; add more workers or optimize handlers
- **Use `UnrecoverableError`** for permanent failures — stops BullMQ from retrying indefinitely
- **Log job start, success, and failure** — critical for debugging production issues

---

## Anti-Patterns to Avoid

- Doing heavy work synchronously in an HTTP handler — use a queue
- Not handling failures — if the worker throws, the job should retry with backoff
- Storing large objects (files, images) directly in queue payloads — store in S3/blob storage, pass the URL
- Not setting `removeOnComplete`/`removeOnFail` — completed jobs accumulate in Redis, wasting memory
- Missing idempotency in payment and notification handlers — at-least-once delivery causes duplicate actions

---

## Debugging & Troubleshooting

**"Jobs are piling up in the queue"**
Workers are too slow or there are too few of them. Check worker concurrency setting. Scale workers horizontally (start more worker processes pointing to the same Redis queue).

**"Same email is being sent twice"**
Your job handler is not idempotent. Add a database-level uniqueness check or use Stripe/Postmark idempotency keys.

**"Workers are not picking up jobs after a server restart"**
Workers are not running. Check your process manager (PM2, Docker restart policy). BullMQ jobs persist in Redis until processed — they won't be lost, but they won't be processed until a worker starts.

---

## Real-World Scenarios

**Scenario: E-commerce order processing pipeline**

```
Order placed (API)
  → enqueue: send-confirmation-email
  → enqueue: reserve-inventory
  → enqueue: notify-fulfillment-warehouse

Workers run in parallel:
  email-worker → send confirmation email
  inventory-worker → reserve stock in DB
  fulfillment-worker → POST to warehouse API

If warehouse API is down:
  → job retries with exponential backoff (2s, 4s, 8s)
  → after 3 failures → move to DLQ
  → alert on-call engineer
  → manually replay DLQ jobs after warehouse recovers
```

---

## Further Reading

- [BullMQ documentation](https://docs.bullmq.io/)
- [Bull Board dashboard](https://github.com/felixmosh/bull-board)
- [RabbitMQ vs Redis for queuing](https://www.cloudamqp.com/blog/part1-rabbitmq-best-practice.html)
- Track 08: [Distributed Systems](distributed-systems.md)

---

## Summary

Message queues are the foundation of resilient, scalable asynchronous architectures. BullMQ with Redis is the pragmatic choice for Node.js applications — well-supported, battle-tested, and simple to operate. The critical disciplines are: keep handlers idempotent (at-least-once delivery is the reality), use retry backoff for transient failures, use `UnrecoverableError` for permanent failures, and monitor queue depth as an early warning of worker capacity issues. Queues turn fragile synchronous dependencies into resilient asynchronous work, and they are the key to absorbing traffic spikes without data loss.
