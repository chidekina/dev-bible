# Message Queues

## 1. What & Why

A message queue is a form of asynchronous service-to-service communication. A producer publishes messages to a queue; one or more consumers process them independently. The producer does not wait for the consumer to finish.

This simple concept has profound implications for system design:

**Decoupling:** The order service does not know which notification service processes its events. Either can be deployed, scaled, or failed independently.

**Load leveling:** A traffic spike of 10,000 order submissions per second does not overwhelm the email service that can only process 500/second. The queue absorbs the burst; the consumer processes at its own pace.

**Reliability:** If the consumer crashes mid-processing, the message is not lost — it is requeued and retried. Without a queue, a crashed HTTP handler means the work is gone.

**Async processing:** HTTP requests should return in milliseconds. Sending a welcome email, generating a PDF, or syncing data to a third-party API can take seconds — offload to a queue and return immediately.

---

## 2. Core Concepts

### Delivery Guarantees

This is the fundamental trade-off in messaging:

**At-most-once:** Fire and forget. The message is sent once; if delivery fails or the consumer crashes, the message is lost. Used for: metrics, non-critical notifications, logging where occasional loss is acceptable.

**At-least-once:** The message is delivered until the consumer acknowledges it. If the consumer crashes before acknowledging, the message is redelivered. The consumer may receive the same message multiple times. **This is the most common guarantee.** Consumers must be idempotent.

**Exactly-once:** Every message is delivered exactly once — no loss, no duplicates. This requires coordination between producer and consumer (distributed transaction or deduplication). It is expensive and sometimes impossible across failure boundaries. In practice, "effectively exactly-once" is achieved by combining at-least-once delivery with idempotent consumers.

> ⚠️ Default to at-least-once. Design your consumers to be idempotent. Attempting exactly-once without understanding the implications leads to complex, fragile systems.

### Dead Letter Queue (DLQ)

When a message fails processing after the maximum number of retries, it is moved to a Dead Letter Queue instead of being discarded. The DLQ is for:
- Manual inspection of failed messages
- Root cause analysis
- Replaying messages after fixing the underlying bug
- Alerting: DLQ depth > 0 = alert

### Idempotent Consumers

An idempotent consumer produces the same result whether it processes a message once or many times.

```typescript
// NON-IDEMPOTENT: processing twice sends two emails
async function sendWelcomeEmail(userId: string) {
  const user = await db.users.findUnique({ where: { id: userId } });
  await emailService.send({ to: user.email, subject: 'Welcome!' });
  // If this consumer restarts here, the email is sent again
}

// IDEMPOTENT: check before acting
async function sendWelcomeEmail(userId: string, messageId: string) {
  // Check if already processed using the message's unique ID
  const alreadyProcessed = await db.processedMessages.findUnique({
    where: { messageId },
  });
  if (alreadyProcessed) {
    console.log(`Message ${messageId} already processed — skipping`);
    return;
  }

  const user = await db.users.findUnique({ where: { id: userId } });
  await emailService.send({ to: user.email, subject: 'Welcome!' });

  // Mark as processed (in same transaction if possible)
  await db.processedMessages.create({ data: { messageId, processedAt: new Date() } });
}
```

---

## 3. RabbitMQ

RabbitMQ is a mature, feature-rich message broker implementing the AMQP protocol.

### Core Concepts

**Exchange:** Receives messages from producers and routes them to queues based on routing rules. Four types:

- **Direct:** Routes to queues where the queue's binding key exactly matches the message's routing key.
- **Fanout:** Broadcasts to all bound queues (pub/sub for all subscribers).
- **Topic:** Routes based on wildcard patterns (`order.#` matches `order.created`, `order.updated`, etc.).
- **Headers:** Routes based on message header attributes (rarely used).

**Queue:** Stores messages until consumed. Can be durable (survives restart), exclusive (one consumer), auto-delete (deleted when last consumer disconnects).

**Binding:** Connects an exchange to a queue with an optional routing key or pattern.

```typescript
import amqplib, { Channel, Connection } from 'amqplib';

// Producer
async function setupProducer(connection: Connection) {
  const channel = await connection.createChannel();

  // Declare exchange (durable = survives broker restart)
  await channel.assertExchange('orders', 'topic', { durable: true });

  // Publish a message
  const message = { orderId: 'order_123', userId: 'user_42', total: 149.99 };
  const published = channel.publish(
    'orders',               // exchange
    'order.created',        // routing key
    Buffer.from(JSON.stringify(message)),
    {
      persistent: true,     // message survives broker restart
      contentType: 'application/json',
      messageId: crypto.randomUUID(), // for idempotency
      timestamp: Date.now(),
    }
  );

  if (!published) {
    // Channel buffer full — need to wait for 'drain' event
    await new Promise((r) => channel.once('drain', r));
  }
}

// Consumer
async function setupConsumer(connection: Connection) {
  const channel = await connection.createChannel();

  // How many unacked messages to prefetch per consumer (flow control)
  channel.prefetch(10); // QoS: process 10 messages in parallel max

  // Declare queue
  await channel.assertQueue('email-notifications', {
    durable: true,
    // Dead letter exchange: failed messages go here
    arguments: {
      'x-dead-letter-exchange': 'orders.dlx',
      'x-dead-letter-routing-key': 'email-notifications.failed',
      'x-message-ttl': 86400000, // 24 hours max time in queue
    },
  });

  // Bind queue to exchange with routing key pattern
  await channel.bindQueue('email-notifications', 'orders', 'order.created');
  await channel.bindQueue('email-notifications', 'orders', 'order.updated');
  // 'order.#' would match all order routing keys

  // Consume
  await channel.consume('email-notifications', async (msg) => {
    if (!msg) return;

    try {
      const payload = JSON.parse(msg.content.toString());
      await processOrderEvent(payload, msg.properties.messageId);

      // Acknowledge: message removed from queue
      channel.ack(msg);
    } catch (err) {
      console.error('Processing failed:', err);

      // Nack with requeue: redelivered to this queue
      // channel.nack(msg, false, true);

      // Nack without requeue: goes to DLX (dead letter exchange)
      channel.nack(msg, false, false);
    }
  });
}

// Dead Letter Exchange setup
async function setupDLX(connection: Connection) {
  const channel = await connection.createChannel();
  await channel.assertExchange('orders.dlx', 'direct', { durable: true });
  await channel.assertQueue('email-notifications.failed', { durable: true });
  await channel.bindQueue('email-notifications.failed', 'orders.dlx', 'email-notifications.failed');
}
```

---

## 4. BullMQ (Redis-based)

BullMQ is a TypeScript-first job queue built on Redis. It is the best choice for Node.js applications that already use Redis and need reliable background job processing.

```typescript
import { Queue, Worker, QueueEvents, Job, FlowProducer } from 'bullmq';
import { Redis } from 'ioredis';

const connection = new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null });

// ---- Queue: adding jobs ----
const emailQueue = new Queue('email', { connection });
const reportQueue = new Queue('reports', { connection });

// Add a job immediately
await emailQueue.add(
  'welcome-email',           // job name
  { userId: 'user_42', email: 'alice@example.com' }, // payload
  {
    jobId: `welcome:user_42`, // deduplication key — if job exists, skip
    attempts: 3,              // retry up to 3 times
    backoff: {
      type: 'exponential',
      delay: 2000,            // 2s, 4s, 8s
    },
    removeOnComplete: 100,    // keep last 100 completed jobs
    removeOnFail: 500,        // keep last 500 failed jobs (for inspection)
  }
);

// Delayed job: run after 1 hour
await emailQueue.add(
  'follow-up-email',
  { userId: 'user_42', template: 'onboarding-day1' },
  { delay: 3600_000 }
);

// Repeating job (cron)
await reportQueue.add(
  'daily-report',
  { reportType: 'revenue' },
  {
    repeat: { pattern: '0 8 * * *' }, // 8am every day
  }
);

// Job with priority (lower number = higher priority)
await emailQueue.add('urgent-notification', payload, { priority: 1 });
await emailQueue.add('marketing-email', payload, { priority: 10 });

// ---- Worker: processing jobs ----
const emailWorker = new Worker(
  'email',
  async (job: Job) => {
    // job.name, job.data, job.id, job.attemptsMade
    console.log(`Processing ${job.name} attempt ${job.attemptsMade + 1}`);

    // Update progress (clients can subscribe to this)
    await job.updateProgress(0);

    if (job.name === 'welcome-email') {
      const { userId, email } = job.data;

      // Idempotency check
      const sent = await db.sentEmails.findUnique({
        where: { jobId: job.id },
      });
      if (sent) {
        console.log(`Email already sent for job ${job.id}`);
        return { skipped: true };
      }

      await emailService.send({
        to: email,
        subject: 'Welcome to the platform!',
        template: 'welcome',
        data: { userId },
      });

      await db.sentEmails.create({ data: { jobId: job.id!, userId } });
      await job.updateProgress(100);

      return { sent: true, email };
    }

    if (job.name === 'follow-up-email') {
      // ... handle differently
    }
  },
  {
    connection,
    concurrency: 5,      // process up to 5 jobs simultaneously
    limiter: {
      max: 50,           // max 50 jobs per period
      duration: 1000,    // per 1 second (rate limiting)
    },
  }
);

// ---- Error handling and DLQ ----
emailWorker.on('completed', (job, result) => {
  console.log(`Job ${job.id} completed:`, result);
});

emailWorker.on('failed', (job, err) => {
  console.error(`Job ${job?.id} failed after ${job?.attemptsMade} attempts:`, err.message);
  // After max attempts, job goes to 'failed' state (BullMQ's DLQ)
});

emailWorker.on('error', (err) => {
  console.error('Worker error:', err);
});

// ---- Inspect and replay failed jobs ----
const failedJobs = await emailQueue.getFailed(0, 49); // first 50 failed jobs
for (const job of failedJobs) {
  console.log(job.id, job.failedReason, job.data);
}

// Retry a specific failed job
const failedJob = await emailQueue.getJob('job-id-123');
await failedJob?.retry();

// Retry ALL failed jobs
await emailQueue.retryJobs({ state: 'failed' });

// ---- Queue Events: real-time monitoring ----
const queueEvents = new QueueEvents('email', { connection });
queueEvents.on('completed', ({ jobId, returnvalue }) => {
  console.log(`Job ${jobId} done:`, returnvalue);
});
queueEvents.on('failed', ({ jobId, failedReason }) => {
  console.error(`Job ${jobId} failed: ${failedReason}`);
});
queueEvents.on('progress', ({ jobId, data }) => {
  console.log(`Job ${jobId} progress: ${data}%`);
});

// ---- Flow: dependent jobs (parent waits for children) ----
const flowProducer = new FlowProducer({ connection });

await flowProducer.add({
  name: 'generate-invoice',
  queueName: 'documents',
  data: { orderId: 'order_123' },
  children: [
    {
      name: 'fetch-order-data',
      queueName: 'data-fetch',
      data: { orderId: 'order_123' },
    },
    {
      name: 'fetch-user-data',
      queueName: 'data-fetch',
      data: { userId: 'user_42' },
    },
  ],
  // parent job only runs when ALL children complete
});
```

---

## 5. Apache Kafka

Kafka is a distributed event streaming platform optimized for high-throughput, durable, replayable event logs.

### Core Concepts

**Topic:** A named log of events. Topics are partitioned and replicated.

**Partition:** The unit of parallelism. Events in a partition are strictly ordered. Different partitions process in parallel. The partition key (a field of the event) determines which partition receives the event — use a stable key (e.g., `userId`) so related events always go to the same partition and stay ordered.

**Offset:** Each event in a partition has an incrementing offset. Consumers track their offset. A consumer can replay events by seeking to an earlier offset.

**Consumer Group:** A set of consumers sharing a topic's partitions. Each partition is assigned to exactly one consumer in the group. If you have 10 partitions and 5 consumers, each consumer gets 2 partitions. Adding consumers scales reads (up to the number of partitions). Multiple independent consumer groups each get a full copy of all events.

**Retention:** Kafka keeps events on disk for a configurable time (default 7 days) or until a size limit. After retention expires, events are deleted. Compacted topics keep only the latest event per key.

```
Topic: "order-events" (3 partitions)

Partition 0: [offset 0: order.created userId=1] [offset 1: order.updated userId=3]
Partition 1: [offset 0: order.created userId=2] [offset 1: order.paid userId=2]
Partition 2: [offset 0: order.created userId=3] [offset 1: order.shipped userId=3]

Consumer Group A (email service): 3 consumers → 1 partition each
Consumer Group B (analytics):     1 consumer  → reads all 3 partitions (slower, different use)
```

```typescript
import { Kafka, Partitioners, CompressionTypes } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: [process.env.KAFKA_BROKER!],
  ssl: true,
  sasl: { mechanism: 'plain', username: process.env.KAFKA_USER!, password: process.env.KAFKA_PASS! },
});

// ---- Producer ----
const producer = kafka.producer({
  createPartitioner: Partitioners.DefaultPartitioner,
  idempotent: true,           // exactly-once producer guarantee (retries won't duplicate)
  transactionTimeout: 30000,
});
await producer.connect();

// Publish an event
await producer.send({
  topic: 'order-events',
  messages: [
    {
      key: 'order_123',        // partition key — same key always same partition
      value: JSON.stringify({
        type: 'order.created',
        orderId: 'order_123',
        userId: 'user_42',
        total: 149.99,
        timestamp: new Date().toISOString(),
      }),
      headers: {
        'content-type': 'application/json',
        'event-version': '1',
      },
    },
  ],
  compression: CompressionTypes.GZIP,
});

// ---- Consumer ----
const consumer = kafka.consumer({
  groupId: 'email-notification-service',
  sessionTimeout: 30000,
  rebalanceTimeout: 60000,
  heartbeatInterval: 3000,
});
await consumer.connect();
await consumer.subscribe({ topics: ['order-events'], fromBeginning: false });

await consumer.run({
  eachBatchAutoResolve: true,
  eachBatch: async ({ batch, resolveOffset, heartbeat, commitOffsetsIfNecessary }) => {
    for (const message of batch.messages) {
      const event = JSON.parse(message.value!.toString());
      const key = message.key?.toString();
      const offset = message.offset;

      try {
        await processOrderEvent(event);
        // Commit offset only after successful processing
        await resolveOffset(offset);
        await heartbeat(); // prevent session timeout during slow processing
      } catch (err) {
        console.error(`Failed to process offset ${offset}:`, err);
        // Decide: retry, skip, or send to a separate error topic
        // For most cases: throw to stop and retry the whole batch
        throw err;
      }
    }

    await commitOffsetsIfNecessary();
  },
});

// ---- Seek / Replay (rewind consumer to a specific offset) ----
consumer.on(consumer.events.GROUP_JOIN, async () => {
  // Reset to beginning of partition 0 to replay all events
  await consumer.seek({
    topic: 'order-events',
    partition: 0,
    offset: '0',
  });
});
```

---

## 6. Kafka vs RabbitMQ vs BullMQ

| Factor | Kafka | RabbitMQ | BullMQ |
|--------|-------|----------|--------|
| Use case | High-throughput event streaming, replay, audit log | Complex routing, low-latency task queues | Node.js job queues |
| Throughput | Millions of messages/sec | Thousands-100k/sec | Thousands/sec per worker |
| Message replay | Yes (by offset) | No (once consumed, gone) | Limited (failed jobs only) |
| Routing | Partition key (simple) | Exchanges + bindings (complex) | Queue name (simple) |
| Message ordering | Per-partition strict ordering | Per-queue with caveats | Per-queue FIFO |
| Retention | Time or size based (days) | Until consumed (or TTL) | Until removed |
| Setup complexity | High (Zookeeper or KRaft) | Medium | Low (just Redis) |
| Multi-consumer | Consumer groups | Compete or pub/sub | Workers share queue |
| Language support | All | All (AMQP clients) | Node.js only |
| Best for | Event sourcing, stream processing, audit | Microservice communication, task routing | Background jobs in Node.js apps |

> 💡 Rule of thumb: If you're building a Node.js app and need background jobs (emails, reports, image processing), start with BullMQ. If you need durable event streaming with replay across services, use Kafka. If you need complex message routing with per-message routing rules, use RabbitMQ.

---

## 7. Patterns

### Outbox Pattern (Transactional Messaging)

The outbox pattern ensures a database write and a message publication are atomic — without a distributed transaction.

```typescript
// Problem: writing to DB and publishing to Kafka are two separate operations
// If the app crashes between them, the DB write succeeds but the message is lost

// Solution: write message to DB (in same transaction), then relay to Kafka

// In the same DB transaction:
async function createOrderWithOutbox(db: PrismaClient, orderData: CreateOrderInput) {
  return db.$transaction(async (tx) => {
    const order = await tx.order.create({ data: orderData });

    // Write event to outbox table IN THE SAME TRANSACTION
    await tx.outboxEvent.create({
      data: {
        eventType: 'order.created',
        aggregateId: order.id,
        payload: JSON.stringify({ orderId: order.id, total: order.total }),
        status: 'PENDING',
      },
    });

    return order;
  });
  // If transaction commits: both order AND outbox event are saved atomically
  // If transaction rolls back: neither is saved
}

// Separate relay process: reads outbox and publishes to Kafka
async function relayOutboxEvents(db: PrismaClient, producer: Producer) {
  const events = await db.outboxEvent.findMany({
    where: { status: 'PENDING' },
    orderBy: { createdAt: 'asc' },
    take: 100,
  });

  for (const event of events) {
    await producer.send({
      topic: 'order-events',
      messages: [{ key: event.aggregateId, value: event.payload }],
    });

    await db.outboxEvent.update({
      where: { id: event.id },
      data: { status: 'PUBLISHED', publishedAt: new Date() },
    });
  }
}
```

### Saga Pattern for Distributed Transactions

```typescript
// Order processing saga: coordinates multiple services without a 2PC transaction
// Each step has a compensating action (rollback equivalent)

class OrderSaga {
  constructor(
    private queue: Queue,
    private db: PrismaClient
  ) {}

  async execute(orderId: string): Promise<void> {
    const saga = await this.db.saga.create({
      data: { orderId, status: 'STARTED', currentStep: 'RESERVE_INVENTORY' }
    });

    await this.queue.add('saga-step', {
      sagaId: saga.id,
      step: 'RESERVE_INVENTORY',
      orderId,
    });
  }
}

// Worker processes saga steps
const sagaWorker = new Worker('saga-step', async (job: Job) => {
  const { sagaId, step, orderId } = job.data;

  switch (step) {
    case 'RESERVE_INVENTORY':
      try {
        await inventoryService.reserve(orderId);
        await sagaQueue.add('saga-step', { sagaId, step: 'CHARGE_PAYMENT', orderId });
      } catch {
        // Compensate: nothing to undo (reservation failed)
        await markSagaFailed(sagaId, 'RESERVE_INVENTORY');
      }
      break;

    case 'CHARGE_PAYMENT':
      try {
        await paymentService.charge(orderId);
        await sagaQueue.add('saga-step', { sagaId, step: 'SEND_CONFIRMATION', orderId });
      } catch {
        // Compensate: release inventory reservation
        await inventoryService.release(orderId);
        await markSagaFailed(sagaId, 'CHARGE_PAYMENT');
      }
      break;

    case 'SEND_CONFIRMATION':
      await emailService.sendOrderConfirmation(orderId);
      await markSagaComplete(sagaId);
      break;
  }
});
```

---

## 8. Common Mistakes & Pitfalls

**Non-idempotent consumers:**

```typescript
// WRONG: adding 10 credits twice if message is redelivered
async function processReward(userId: string) {
  await db.users.update({
    where: { id: userId },
    data: { credits: { increment: 10 } },
  });
}

// CORRECT: idempotent using message ID
async function processReward(userId: string, messageId: string) {
  await db.$transaction(async (tx) => {
    const existing = await tx.processedMessages.findUnique({ where: { messageId } });
    if (existing) return;

    await tx.users.update({ where: { id: userId }, data: { credits: { increment: 10 } } });
    await tx.processedMessages.create({ data: { messageId } });
  });
}
```

**Poison pill messages:**

```typescript
// A poison pill is a message that always fails (bad data, unhandled type)
// Without protection, it blocks the queue indefinitely (retried forever)

// BullMQ: set max attempts
await queue.add('process', badData, {
  attempts: 5,     // after 5 failures, moves to 'failed' state
  backoff: { type: 'exponential', delay: 1000 },
});

// RabbitMQ: check attemptCount and send to DLX after max retries
channel.consume('my-queue', async (msg) => {
  const deathCount = (msg?.properties.headers?.['x-death']?.[0]?.count ?? 0) as number;
  if (deathCount >= 5) {
    // Send to poison pill queue for manual inspection
    channel.sendToQueue('poison-pills', msg!.content, {
      headers: { 'original-routing-key': msg!.fields.routingKey },
    });
    channel.ack(msg!); // acknowledge to remove from main queue
    return;
  }
  // ... process
});
```

**Long-running consumers blocking queue processing:**

```typescript
// WRONG: one slow job blocks all others in the worker
const worker = new Worker('reports', async (job) => {
  const data = await generateHugeReport(job.data); // 5 minutes!
  // While this runs, no other jobs are processed
});

// CORRECT: set concurrency and configure timeouts
const worker = new Worker('reports', processReport, {
  connection,
  concurrency: 3,    // process 3 jobs in parallel
  lockDuration: 600_000, // 10 min lock (must exceed max job duration)
  lockRenewTime: 300_000, // renew lock every 5 minutes
});
```

---

## 9. Monitoring and Observability

```typescript
// BullMQ: expose queue metrics
app.get('/metrics/queues', async (req, reply) => {
  const queues = ['email', 'reports', 'notifications'];
  const metrics = await Promise.all(
    queues.map(async (name) => {
      const queue = new Queue(name, { connection });
      const [waiting, active, completed, failed, delayed] = await Promise.all([
        queue.getWaitingCount(),
        queue.getActiveCount(),
        queue.getCompletedCount(),
        queue.getFailedCount(),
        queue.getDelayedCount(),
      ]);
      return { queue: name, waiting, active, completed, failed, delayed };
    })
  );
  return reply.send(metrics);
});

// Alert if DLQ (failed jobs) grows
setInterval(async () => {
  const failedCount = await emailQueue.getFailedCount();
  if (failedCount > 10) {
    await alerting.send(`Email queue has ${failedCount} failed jobs — check DLQ`);
  }
}, 60_000);
```

---

## 10. When to Use / Not Use

**Use message queues when:**
- Processing takes longer than an acceptable HTTP response time (> 500ms)
- You need to absorb traffic spikes (load leveling)
- Tasks must survive service restarts (reliability)
- Multiple services need to react to the same event (pub/sub)
- You need guaranteed delivery + retry on failure
- Background jobs: emails, reports, image processing, data sync

**You may NOT need a message queue when:**
- The task is fast and synchronous is fine
- You only have one service and do not need decoupling
- The task does not need to survive crashes (ephemeral)
- You already have a database and can use it as a job store (Postgres-backed job queues like pg-boss)
- Simplicity is paramount and the overhead of a broker is not justified

---

## 11. Real-World Scenario

**Problem:** An e-commerce platform sends order confirmation emails synchronously in the `/checkout` HTTP handler. During Black Friday, the email provider's rate limit causes email sends to take 8 seconds. All checkout requests are waiting for the email service. The entire checkout flow is blocked — users are stuck on the loading screen for 8 seconds and abandoning.

**Solution: Async email via BullMQ**

```typescript
// Before: synchronous, blocking
app.post('/checkout', async (req, reply) => {
  const order = await orderService.create(req.body);
  await emailService.sendOrderConfirmation(order); // 8 SECONDS blocking!
  return reply.status(201).send({ orderId: order.id });
});

// After: async, instant response
const emailQueue = new Queue('transactional-emails', { connection });

app.post('/checkout', async (req, reply) => {
  const order = await orderService.create(req.body);

  // Add to queue — returns in < 5ms
  await emailQueue.add(
    'order-confirmation',
    { orderId: order.id, userEmail: req.body.email },
    {
      jobId: `order-confirmation:${order.id}`, // deduplication
      attempts: 5,
      backoff: { type: 'exponential', delay: 2000 },
    }
  );

  // Return immediately
  return reply.status(201).send({ orderId: order.id });
});

// Worker handles emails at its own pace (respects rate limits gracefully)
const emailWorker = new Worker(
  'transactional-emails',
  async (job: Job) => {
    const { orderId, userEmail } = job.data;
    await emailService.sendOrderConfirmation({ orderId, userEmail });
  },
  {
    connection,
    concurrency: 10,
    limiter: { max: 50, duration: 1000 }, // respect rate limit: 50 emails/second
  }
);
```

**Result:** Checkout response time drops from 8 seconds to 120ms. Email delivery continues in the background at the rate the provider allows. No emails are lost — the queue retries automatically if the provider is down. Users complete checkout instantly.

---

## 12. Interview Questions

**Q1: What is the difference between at-least-once and exactly-once delivery?**

At-least-once: the broker retries delivery until the consumer acknowledges. If the consumer processes but crashes before acknowledging, the message is redelivered — possible duplicates. Consumers must be idempotent. Most messaging systems default to this. Exactly-once: each message is delivered and processed exactly once — no duplicates, no loss. Requires coordination: Kafka achieves this with idempotent producers + transactional consumers, at significant performance cost. RabbitMQ cannot natively guarantee exactly-once across failures. In practice: use at-least-once + idempotent consumers — it gives you "effectively exactly-once" semantics with simpler infrastructure.

**Q2: What is a Dead Letter Queue and how do you use it?**

A DLQ (Dead Letter Queue) receives messages that fail processing after the maximum number of retries. Instead of discarding failed messages, they are routed to the DLQ for inspection and manual replay. In RabbitMQ, configure `x-dead-letter-exchange` on the queue. In BullMQ, failed jobs (after max attempts) remain in the `failed` state and are accessible via `queue.getFailed()`. The DLQ enables: root cause analysis (see the failing payload + error), manual replay after fixing the bug, and alerting (DLQ depth > 0 triggers an alert).

**Q3: How do Kafka partitions and consumer groups work?**

A Kafka topic is split into N partitions. Within a partition, events are strictly ordered and each has an incrementing offset. A consumer group is a set of consumers sharing a subscription to a topic — partitions are divided among group members (one partition → one consumer at a time). With 6 partitions and 3 consumers, each consumer gets 2 partitions — parallelism scales linearly. Multiple independent consumer groups each receive all events independently. The implication: to ensure related events (e.g., all events for the same order) are processed in order, use the order ID as the partition key so they always land in the same partition.

**Q4: What are RabbitMQ exchange types and when do you use each?**

Direct: routes messages to queues whose binding key exactly matches the message routing key. Use for task queues where you target a specific queue. Fanout: broadcasts to all bound queues regardless of routing key. Use for notifications where every subscriber should receive every message. Topic: routes based on wildcard patterns (`order.*` matches `order.created`, `order.updated`; `#` matches zero or more segments). Use when subscribers need to filter by event type patterns. Headers: routes based on message headers instead of routing key. Rarely used — complex and slow compared to topic routing.

**Q5: How does BullMQ handle retries and failures?**

Configure `attempts` (max retry count) and `backoff` (delay strategy: fixed or exponential) when adding a job. On failure, BullMQ retries the job after the backoff delay. If the job fails `attempts` times, it moves to the `failed` state (BullMQ's DLQ). You can inspect failed jobs with `queue.getFailed()`, view the error message, and call `job.retry()` to replay. Use `backoff: { type: 'exponential', delay: 2000 }` for most cases — it avoids hammering a recovering service. Use `removeOnFail: 500` to keep only the last 500 failures (prevents Redis memory growth).

**Q6: Why do consumers need to be idempotent?**

Because at-least-once delivery guarantees that messages may arrive more than once: consumer crashes before acknowledging, network issues cause redelivery, manual replay from DLQ. An idempotent consumer produces the same result regardless of how many times the same message is processed. Implementation patterns: (1) Check a deduplication table in DB before processing (insert `messageId` atomically with the side effect). (2) Use conditional updates (`WHERE processed = FALSE`). (3) For financial operations: use idempotency keys that prevent double charges. Non-idempotent consumers with at-least-once delivery = incorrect application state.

**Q7: When would you choose Kafka over RabbitMQ?**

Choose Kafka when: you need high throughput (millions of events/second), you need event replay (consumers can seek back in time), you need multiple independent consumer groups to independently process the same event stream, your use case is event sourcing or CQRS, or you need long-term event retention. Choose RabbitMQ when: you need complex routing (topic exchanges, header routing), low-latency delivery is critical, you need the message to be consumed once and gone, or you need per-message routing decisions. Choose BullMQ when: you're in Node.js, already use Redis, and need background job processing without the operational complexity of Kafka or RabbitMQ.

**Q8: How do you handle poison pill messages?**

A poison pill is a malformed or unprocessable message that fails on every attempt. Without protection, it can block a queue or consume retry budget indefinitely. Strategies: (1) **Max retries + DLQ** — after N failures, route to DLQ for manual inspection. (2) **Try-catch with inspection** — catch the error, log the message content and error, then acknowledge (or send to DLQ manually) rather than retrying indefinitely. (3) **Schema validation** — validate message schema before processing; immediately DLQ if schema is invalid (never retry a structurally broken message). (4) **Circuit breaker** — if a high percentage of messages are failing (suggesting a systemic bug), pause the consumer and alert rather than hammering the DLQ.

---

## 13. Exercises

**Exercise 1: BullMQ workflow**

Build a document processing pipeline using BullMQ that:
- Accepts PDF file uploads
- Queue step 1: extract text (slow, CPU-intensive)
- Queue step 2: run sentiment analysis on extracted text (depends on step 1)
- Queue step 3: store results in DB and notify user via WebSocket (depends on step 2)
- Use BullMQ Flows for step dependencies
- Expose `GET /jobs/:jobId/status` endpoint returning progress and current step

*Hint:* Use `FlowProducer` for parent/child job dependencies. Use `job.updateProgress()` at each stage.

**Exercise 2: Idempotent order processing**

Implement an order payment consumer that is fully idempotent:
- Receives `{ orderId, userId, amount, messageId }` from a queue
- Should NOT charge the customer twice if the message is delivered more than once
- Should handle: DB down, payment provider down, message redelivery after partial success
- Write tests that simulate duplicate message delivery

*Hint:* Use a `payment_idempotency` table with `(messageId, status)`. Wrap the check+charge+record in a DB transaction.

**Exercise 3: Kafka consumer with offset management**

Build a Kafka consumer that:
- Reads from a `user-events` topic (3 partitions)
- Processes events in order per userId (same userId always in same partition via key)
- Manually commits offsets only after successful DB write (no auto-commit)
- Handles rebalancing gracefully (save and restore offsets)
- Exposes a health endpoint showing current partition assignments and latest committed offsets

*Hint:* Use `eachBatch` mode with manual offset resolution. Subscribe to `consumer.events.GROUP_JOIN` for rebalancing events.

---

## 14. Further Reading

- [BullMQ documentation](https://docs.bullmq.io/)
- [RabbitMQ tutorials](https://www.rabbitmq.com/getstarted.html)
- [Apache Kafka documentation](https://kafka.apache.org/documentation/)
- [KafkaJS — Node.js Kafka client](https://kafka.js.org/)
- [Enterprise Integration Patterns (Gregor Hohpe)](https://www.enterpriseintegrationpatterns.com/)
- [AMQP 0-9-1 model specification](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [The Log: What every software engineer should know about real-time data (Jay Kreps)](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [Outbox Pattern — Microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [Saga Pattern — Microservices.io](https://microservices.io/patterns/data/saga.html)
- Book: *Designing Data-Intensive Applications* by Martin Kleppmann (chapters on messaging)
