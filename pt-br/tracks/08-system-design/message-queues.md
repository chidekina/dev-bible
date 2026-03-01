# Message Queues

## Visão Geral

Message queues desacoplam os componentes de um sistema ao permitir que um serviço envie trabalho para outro sem esperar que ele seja concluído. O remetente adiciona uma mensagem à queue e segue em frente; um worker a busca de forma assíncrona. Isso libera o escalonamento horizontal de trabalho em background, resiliência a falhas de serviços downstream e a capacidade de absorver picos de tráfego sem perder dados. Este capítulo cobre os fundamentos de queue, BullMQ na prática, dead-letter queues e padrões comuns.

---

## Pré-requisitos

- Padrões assíncronos em Node.js e TypeScript
- Conhecimento básico de Redis (BullMQ usa Redis como backend)
- Track 03 — Backend: entendimento de padrões API-para-background-job

---

## Conceitos Fundamentais

### Por que usar queues?

**Fluxo síncrono (sem queue):**
```
Usuário → API → envia email → aguarda 2 segundos → responde ao usuário
```

**Fluxo assíncrono (com queue):**
```
Usuário → API → enfileira "send email" → responde imediatamente (rápido)
                    ↓
             Worker → envia email (em background, sem pressão de timeout)
```

Benefícios:
- **Resiliência:** se o serviço de email estiver fora do ar, o job aguarda na queue e é retentado
- **Escalabilidade:** adicione mais workers para processar mais jobs em paralelo
- **Desacoplamento:** a API não sabe nem se importa com a implementação do email
- **Suavização de tráfego:** pico repentino de 10.000 cadastros? A queue absorve o pico; workers processam no próprio ritmo

### Conceitos-chave

| Conceito | Definição |
|----------|----------|
| Producer | O componente que adiciona mensagens à queue |
| Consumer / Worker | O componente que lê e processa mensagens |
| Queue | A lista ordenada de mensagens aguardando processamento |
| Dead-letter queue (DLQ) | Para onde as mensagens vão após esgotar todas as tentativas de retry |
| Acknowledgment | Worker sinaliza "processei esta mensagem" — remove-a da queue |
| Entrega at-least-once | A queue garante que toda mensagem seja entregue ao menos uma vez (pode ser duplicada) |
| Idempotência | Processar a mesma mensagem duas vezes produz o mesmo resultado que processá-la uma vez |

---

## Exemplos Práticos

### Setup do BullMQ

```bash
npm install bullmq
```

```typescript
// src/lib/queue.ts
import { Queue, Worker, QueueEvents } from 'bullmq';
import { redis } from './redis.js';

const connection = { host: process.env.REDIS_HOST, port: parseInt(process.env.REDIS_PORT ?? '6379') };

// Define tipos de job
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

// Cria queues
export const emailQueue = new Queue<EmailJobData>('email', { connection });
export const invoiceQueue = new Queue<InvoiceJobData>('invoice', { connection });
```

### Adicionando jobs (producer)

```typescript
// src/routes/auth.ts
import { emailQueue } from '../lib/queue.js';

fastify.post('/auth/register', async (request, reply) => {
  const { email, password } = RegisterSchema.parse(request.body);

  const user = await db.user.create({
    data: { email, passwordHash: await hashPassword(password) },
  });

  // Fire-and-forget — adiciona à queue e retorna imediatamente
  await emailQueue.add(
    'welcome-email',
    {
      to: user.email,
      subject: 'Welcome!',
      template: 'welcome',
      variables: { name: user.name ?? user.email },
    },
    {
      attempts: 3,               // tenta até 3 vezes em caso de falha
      backoff: { type: 'exponential', delay: 2000 }, // 2s, 4s, 8s
      removeOnComplete: { count: 100 },  // mantém os últimos 100 jobs concluídos
      removeOnFail: { count: 500 },      // mantém os últimos 500 jobs com falha
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
      // Distingue falhas permanentes de transitórias
      if (error instanceof InvalidEmailError) {
        // Não tente novamente — o endereço de email é inválido
        throw new UnrecoverableError(`Invalid email: ${job.data.to}`);
      }
      // Erro transitório — deixa o BullMQ fazer retry com backoff
      throw error;
    }
  },
  {
    connection,
    concurrency: 5, // processa até 5 emails em paralelo
  }
);

emailWorker.on('completed', (job) => {
  logger.info({ jobId: job.id }, 'Email job completed');
});

emailWorker.on('failed', (job, error) => {
  logger.error({ jobId: job?.id, error: error.message }, 'Email job failed');
});
```

### Padrão dead-letter queue

```typescript
// Captura jobs que esgotaram todas as tentativas de retry
import { QueueEvents } from 'bullmq';

const emailQueueEvents = new QueueEvents('email', { connection });

emailQueueEvents.on('failed', async ({ jobId, failedReason }) => {
  const job = await emailQueue.getJob(jobId);
  if (!job) return;

  const maxAttempts = job.opts.attempts ?? 1;
  if (job.attemptsMade >= maxAttempts) {
    // Move para a DLQ
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

### Agendamento de jobs (delayed e recorrentes)

```typescript
// Job com delay — processa no futuro
await emailQueue.add(
  'reminder',
  { to: user.email, subject: '7-day trial ending', template: 'trial-ending', variables: {} },
  {
    delay: 7 * 24 * 60 * 60 * 1000, // 7 dias a partir de agora
    attempts: 3,
  }
);

// Job recorrente — sintaxe cron
import { Queue } from 'bullmq';

const reportQueue = new Queue('reports', { connection });

// Gera relatório semanal toda segunda às 8h
await reportQueue.add(
  'weekly-report',
  { type: 'weekly' },
  {
    repeat: { cron: '0 8 * * 1' }, // cron: minuto hora dia mês dia-da-semana
    attempts: 3,
  }
);
```

### Processamento idempotente de jobs

Como queues entregam at-least-once, sempre escreva handlers idempotentes:

```typescript
// Ruim — não é idempotente. Se rodar duas vezes, o usuário é cobrado duas vezes.
async function processPayment(job: Job<PaymentJobData>) {
  await stripe.charges.create({ amount: job.data.amount, source: job.data.token });
}

// Bom — idempotente usando uma idempotency key
async function processPayment(job: Job<PaymentJobData>) {
  const idempotencyKey = `payment:${job.data.orderId}`;

  // Verifica se já processamos isso
  const existing = await db.payment.findUnique({
    where: { orderId: job.data.orderId },
  });

  if (existing) {
    logger.info({ orderId: job.data.orderId }, 'Payment already processed — skipping');
    return existing;
  }

  // Passa a idempotency key para o Stripe — evita cobrança dupla mesmo se o Stripe for chamado duas vezes
  const charge = await stripe.charges.create(
    { amount: job.data.amount, source: job.data.token },
    { idempotencyKey }
  );

  return db.payment.create({
    data: { orderId: job.data.orderId, stripeChargeId: charge.id, amount: job.data.amount },
  });
}
```

### Setup do BullMQ Dashboard (Bull Board)

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
// Acesse http://localhost:3000/admin/queues para ver o status dos jobs
```

---

## Padrões e Boas Práticas

- **Mantenha os payloads de job pequenos** — armazene referências (IDs) na queue, busque os dados no worker
- **Faça todos os workers idempotentes** — assuma que qualquer job pode rodar duas vezes
- **Use queues separadas para diferentes prioridades** — notificações por email vs. processamento de pagamentos
- **Defina concorrência adequada por queue** — CPU-bound: 1 por core; I/O-bound: 10–50
- **Monitore a profundidade da queue** — uma queue crescendo significa que os workers não conseguem acompanhar; adicione mais workers ou otimize os handlers
- **Use `UnrecoverableError`** para falhas permanentes — impede que o BullMQ faça retry indefinidamente
- **Logue início, sucesso e falha do job** — crítico para debugging de problemas em produção

---

## Anti-Padrões a Evitar

- Fazer trabalho pesado de forma síncrona em um handler HTTP — use uma queue
- Não tratar falhas — se o worker lançar uma exceção, o job deve fazer retry com backoff
- Armazenar objetos grandes (arquivos, imagens) diretamente nos payloads de queue — armazene no S3/blob storage, passe a URL
- Não definir `removeOnComplete`/`removeOnFail` — jobs concluídos acumulam no Redis, desperdiçando memória
- Idempotência ausente em handlers de pagamento e notificação — entrega at-least-once causa ações duplicadas

---

## Debugging e Troubleshooting

**"Jobs estão se acumulando na queue"**
Os workers estão lentos ou há poucos deles. Verifique a configuração de concorrência dos workers. Escale workers horizontalmente (inicie mais processos de worker apontando para a mesma queue Redis).

**"O mesmo email está sendo enviado duas vezes"**
Seu handler de job não é idempotente. Adicione uma verificação de unicidade em nível de banco de dados ou use idempotency keys do Stripe/Postmark.

**"Workers não estão pegando jobs após reinicialização do servidor"**
Os workers não estão rodando. Verifique seu gerenciador de processos (PM2, Docker restart policy). Jobs do BullMQ persistem no Redis até serem processados — eles não serão perdidos, mas não serão processados até um worker iniciar.

---

## Cenários do Mundo Real

**Cenário: Pipeline de processamento de pedidos em e-commerce**

```
Pedido realizado (API)
  → enfileira: send-confirmation-email
  → enfileira: reserve-inventory
  → enfileira: notify-fulfillment-warehouse

Workers rodam em paralelo:
  email-worker → envia email de confirmação
  inventory-worker → reserva estoque no DB
  fulfillment-worker → POST para API do depósito

Se a API do depósito estiver fora:
  → job tenta novamente com backoff exponencial (2s, 4s, 8s)
  → após 3 falhas → move para DLQ
  → alerta o engenheiro de plantão
  → replay manual dos jobs da DLQ após o depósito se recuperar
```

---

## Leitura Complementar

- [Documentação do BullMQ](https://docs.bullmq.io/)
- [Bull Board dashboard](https://github.com/felixmosh/bull-board)
- [RabbitMQ vs Redis para queuing](https://www.cloudamqp.com/blog/part1-rabbitmq-best-practice.html)
- Track 08: [Sistemas Distribuídos](distributed-systems.md)

---

## Resumo

Message queues são a base de arquiteturas assíncronas resilientes e escaláveis. BullMQ com Redis é a escolha pragmática para aplicações Node.js — bem suportada, battle-tested e simples de operar. As disciplinas críticas são: mantenha handlers idempotentes (entrega at-least-once é a realidade), use retry com backoff para falhas transitórias, use `UnrecoverableError` para falhas permanentes e monitore a profundidade da queue como aviso antecipado de problemas de capacidade dos workers. Queues transformam dependências síncronas frágeis em trabalho assíncrono resiliente, e são a chave para absorver picos de tráfego sem perda de dados.
