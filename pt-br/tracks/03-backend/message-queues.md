# Filas de Mensagens

## 1. O Que É e Por Que Usar

Uma fila de mensagens é uma forma de comunicação assíncrona entre serviços. Um produtor publica mensagens em uma fila; um ou mais consumidores as processam independentemente. O produtor não espera o consumidor terminar.

Esse conceito simples tem implicações profundas no design de sistemas:

**Desacoplamento:** O serviço de pedidos não sabe qual serviço de notificação processa seus eventos. Qualquer um pode ser deployado, escalado ou falhar independentemente.

**Nivelamento de carga:** Um pico de 10.000 submissões de pedidos por segundo não sobrecarrega o serviço de e-mail que processa apenas 500/segundo. A fila absorve a rajada; o consumidor processa no próprio ritmo.

**Confiabilidade:** Se o consumidor travar no meio do processamento, a mensagem não é perdida — ela volta à fila e é tentada novamente. Sem uma fila, um handler HTTP travado significa que o trabalho se perde.

**Processamento assíncrono:** Requisições HTTP devem retornar em milissegundos. Enviar um e-mail de boas-vindas, gerar um PDF ou sincronizar dados com uma API de terceiros pode levar segundos — delegue para uma fila e retorne imediatamente.

---

## 2. Conceitos Fundamentais

### Garantias de Entrega

Esta é a principal troca em mensageria:

**No máximo uma vez (at-most-once):** Dispara e esquece. A mensagem é enviada uma vez; se a entrega falhar ou o consumidor travar, a mensagem se perde. Usado para: métricas, notificações não críticas, logging onde perda ocasional é aceitável.

**Pelo menos uma vez (at-least-once):** A mensagem é entregue até o consumidor confirmar (ack). Se o consumidor travar antes de confirmar, a mensagem é reenviada. O consumidor pode receber a mesma mensagem múltiplas vezes. **Esta é a garantia mais comum.** Consumidores devem ser idempotentes.

**Exatamente uma vez (exactly-once):** Cada mensagem é entregue exatamente uma vez — sem perda, sem duplicatas. Requer coordenação entre produtor e consumidor (transação distribuída ou deduplicação). É caro e às vezes impossível por fronteiras de falha. Na prática, "efetivamente exatamente uma vez" é alcançado combinando at-least-once com consumidores idempotentes.

> Adote at-least-once como padrão. Projete consumidores idempotentes. Tentar exactly-once sem entender as implicações leva a sistemas complexos e frágeis.

### Dead Letter Queue (DLQ)

Quando uma mensagem falha no processamento após o número máximo de tentativas, é movida para uma Dead Letter Queue em vez de ser descartada. A DLQ serve para:
- Inspeção manual de mensagens com falha
- Análise da causa raiz
- Reprocessamento após corrigir o bug subjacente
- Alertas: profundidade da DLQ > 0 = alarme

### Consumidores Idempotentes

Um consumidor idempotente produz o mesmo resultado independente de processar uma mensagem uma ou várias vezes.

```typescript
// NÃO IDEMPOTENTE: processar duas vezes envia dois e-mails
async function sendWelcomeEmail(userId: string) {
  const user = await db.users.findUnique({ where: { id: userId } });
  await emailService.send({ to: user.email, subject: 'Bem-vindo!' });
  // Se o consumidor reiniciar aqui, o e-mail é enviado novamente
}

// IDEMPOTENTE: verifica antes de agir
async function sendWelcomeEmail(userId: string, messageId: string) {
  // Verifica se já foi processado usando o ID único da mensagem
  const alreadyProcessed = await db.processedMessages.findUnique({
    where: { messageId },
  });
  if (alreadyProcessed) {
    console.log(`Mensagem ${messageId} já processada — ignorando`);
    return;
  }

  const user = await db.users.findUnique({ where: { id: userId } });
  await emailService.send({ to: user.email, subject: 'Bem-vindo!' });

  // Marca como processado (na mesma transação se possível)
  await db.processedMessages.create({ data: { messageId, processedAt: new Date() } });
}
```

---

## 3. RabbitMQ

RabbitMQ é um broker de mensagens maduro e rico em recursos que implementa o protocolo AMQP.

### Conceitos Principais

**Exchange:** Recebe mensagens dos produtores e as roteia para filas baseado em regras. Quatro tipos:

- **Direct:** Roteia para filas onde a binding key coincide exatamente com a routing key da mensagem.
- **Fanout:** Transmite para todas as filas vinculadas (pub/sub para todos os inscritos).
- **Topic:** Roteia baseado em padrões wildcard (`order.#` casa com `order.created`, `order.updated`, etc.).
- **Headers:** Roteia baseado em atributos do cabeçalho da mensagem (raramente usado).

**Queue:** Armazena mensagens até serem consumidas. Pode ser durável (sobrevive a reinicialização), exclusiva (um consumidor) ou auto-delete (deletada quando o último consumidor desconecta).

**Binding:** Conecta um exchange a uma fila com routing key ou padrão opcional.

```typescript
import amqplib, { Channel, Connection } from 'amqplib';

// Produtor
async function setupProducer(connection: Connection) {
  const channel = await connection.createChannel();

  // Declara exchange (durable = sobrevive a reinicialização do broker)
  await channel.assertExchange('orders', 'topic', { durable: true });

  // Publica uma mensagem
  const message = { orderId: 'order_123', userId: 'user_42', total: 149.99 };
  const published = channel.publish(
    'orders',               // exchange
    'order.created',        // routing key
    Buffer.from(JSON.stringify(message)),
    {
      persistent: true,     // mensagem sobrevive a reinicialização do broker
      contentType: 'application/json',
      messageId: crypto.randomUUID(), // para idempotência
      timestamp: Date.now(),
    }
  );

  if (!published) {
    // Buffer do canal cheio — aguarda evento 'drain'
    await new Promise((r) => channel.once('drain', r));
  }
}

// Consumidor
async function setupConsumer(connection: Connection) {
  const channel = await connection.createChannel();

  // Quantas mensagens não confirmadas buscar por consumidor (controle de fluxo)
  channel.prefetch(10); // QoS: processa no máximo 10 mensagens em paralelo

  // Declara fila
  await channel.assertQueue('email-notifications', {
    durable: true,
    // Dead letter exchange: mensagens com falha vão para cá
    arguments: {
      'x-dead-letter-exchange': 'orders.dlx',
      'x-dead-letter-routing-key': 'email-notifications.failed',
      'x-message-ttl': 86400000, // 24 horas de tempo máximo na fila
    },
  });

  // Vincula fila ao exchange com padrão de routing key
  await channel.bindQueue('email-notifications', 'orders', 'order.created');
  await channel.bindQueue('email-notifications', 'orders', 'order.updated');
  // 'order.#' casaria com todas as routing keys de order

  // Consome
  await channel.consume('email-notifications', async (msg) => {
    if (!msg) return;

    try {
      const payload = JSON.parse(msg.content.toString());
      await processOrderEvent(payload, msg.properties.messageId);

      // Acknowledge: mensagem removida da fila
      channel.ack(msg);
    } catch (err) {
      console.error('Falha no processamento:', err);

      // Nack com requeue: reenviada para esta fila
      // channel.nack(msg, false, true);

      // Nack sem requeue: vai para o DLX (dead letter exchange)
      channel.nack(msg, false, false);
    }
  });
}

// Configuração do Dead Letter Exchange
async function setupDLX(connection: Connection) {
  const channel = await connection.createChannel();
  await channel.assertExchange('orders.dlx', 'direct', { durable: true });
  await channel.assertQueue('email-notifications.failed', { durable: true });
  await channel.bindQueue('email-notifications.failed', 'orders.dlx', 'email-notifications.failed');
}
```

---

## 4. BullMQ (Baseado em Redis)

BullMQ é uma fila de jobs TypeScript-first construída sobre Redis. É a melhor escolha para aplicações Node.js que já usam Redis e precisam de processamento confiável de jobs em background.

```typescript
import { Queue, Worker, QueueEvents, Job, FlowProducer } from 'bullmq';
import { Redis } from 'ioredis';

const connection = new Redis(process.env.REDIS_URL!, { maxRetriesPerRequest: null });

// ---- Queue: adicionando jobs ----
const emailQueue = new Queue('email', { connection });
const reportQueue = new Queue('reports', { connection });

// Adiciona um job imediatamente
await emailQueue.add(
  'welcome-email',           // nome do job
  { userId: 'user_42', email: 'alice@example.com' }, // payload
  {
    jobId: `welcome:user_42`, // chave de deduplicação — se o job existe, ignora
    attempts: 3,              // tenta até 3 vezes
    backoff: {
      type: 'exponential',
      delay: 2000,            // 2s, 4s, 8s
    },
    removeOnComplete: 100,    // mantém os últimos 100 jobs concluídos
    removeOnFail: 500,        // mantém os últimos 500 jobs com falha (para inspeção)
  }
);

// Job com delay: executa após 1 hora
await emailQueue.add(
  'follow-up-email',
  { userId: 'user_42', template: 'onboarding-day1' },
  { delay: 3600_000 }
);

// Job repetitivo (cron)
await reportQueue.add(
  'daily-report',
  { reportType: 'revenue' },
  {
    repeat: { pattern: '0 8 * * *' }, // 8h todo dia
  }
);

// Job com prioridade (número menor = maior prioridade)
await emailQueue.add('urgent-notification', payload, { priority: 1 });
await emailQueue.add('marketing-email', payload, { priority: 10 });

// ---- Worker: processando jobs ----
const emailWorker = new Worker(
  'email',
  async (job: Job) => {
    // job.name, job.data, job.id, job.attemptsMade
    console.log(`Processando ${job.name} tentativa ${job.attemptsMade + 1}`);

    // Atualiza progresso (clientes podem se inscrever)
    await job.updateProgress(0);

    if (job.name === 'welcome-email') {
      const { userId, email } = job.data;

      // Verificação de idempotência
      const sent = await db.sentEmails.findUnique({
        where: { jobId: job.id },
      });
      if (sent) {
        console.log(`E-mail já enviado para o job ${job.id}`);
        return { skipped: true };
      }

      await emailService.send({
        to: email,
        subject: 'Bem-vindo à plataforma!',
        template: 'welcome',
        data: { userId },
      });

      await db.sentEmails.create({ data: { jobId: job.id!, userId } });
      await job.updateProgress(100);

      return { sent: true, email };
    }

    if (job.name === 'follow-up-email') {
      // ... trata diferente
    }
  },
  {
    connection,
    concurrency: 5,      // processa até 5 jobs simultaneamente
    limiter: {
      max: 50,           // máximo de 50 jobs por período
      duration: 1000,    // por 1 segundo (rate limiting)
    },
  }
);

// ---- Tratamento de erros e DLQ ----
emailWorker.on('completed', (job, result) => {
  console.log(`Job ${job.id} concluído:`, result);
});

emailWorker.on('failed', (job, err) => {
  console.error(`Job ${job?.id} falhou após ${job?.attemptsMade} tentativas:`, err.message);
  // Após as tentativas máximas, o job vai para o estado 'failed' (DLQ do BullMQ)
});

emailWorker.on('error', (err) => {
  console.error('Erro no worker:', err);
});

// ---- Inspecionar e reprocessar jobs com falha ----
const failedJobs = await emailQueue.getFailed(0, 49); // primeiros 50 jobs com falha
for (const job of failedJobs) {
  console.log(job.id, job.failedReason, job.data);
}

// Tenta novamente um job específico com falha
const failedJob = await emailQueue.getJob('job-id-123');
await failedJob?.retry();

// Tenta novamente TODOS os jobs com falha
await emailQueue.retryJobs({ state: 'failed' });

// ---- Queue Events: monitoramento em tempo real ----
const queueEvents = new QueueEvents('email', { connection });
queueEvents.on('completed', ({ jobId, returnvalue }) => {
  console.log(`Job ${jobId} concluído:`, returnvalue);
});
queueEvents.on('failed', ({ jobId, failedReason }) => {
  console.error(`Job ${jobId} falhou: ${failedReason}`);
});
queueEvents.on('progress', ({ jobId, data }) => {
  console.log(`Progresso do job ${jobId}: ${data}%`);
});

// ---- Flow: jobs dependentes (pai aguarda filhos) ----
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
  // job pai só executa quando TODOS os filhos concluem
});
```

---

## 5. Apache Kafka

Kafka é uma plataforma de streaming de eventos distribuída, otimizada para logs de eventos com alto throughput, durabilidade e capacidade de replay.

### Conceitos Principais

**Topic:** Log nomeado de eventos. Topics são particionados e replicados.

**Partition:** A unidade de paralelismo. Eventos em uma partição são estritamente ordenados. Partições diferentes processam em paralelo. A chave de partição (um campo do evento) determina qual partição recebe o evento — use uma chave estável (ex: `userId`) para que eventos relacionados sempre vão para a mesma partição e permaneçam em ordem.

**Offset:** Cada evento em uma partição tem um offset incremental. Consumidores rastreiam seu offset. Um consumidor pode fazer replay de eventos buscando um offset anterior.

**Consumer Group:** Conjunto de consumidores compartilhando as partições de um topic. Cada partição é atribuída a exatamente um consumidor no grupo. Com 10 partições e 5 consumidores, cada consumidor recebe 2 partições. Adicionar consumidores escala leituras (até o número de partições). Múltiplos consumer groups independentes recebem uma cópia completa de todos os eventos.

**Retenção:** Kafka mantém eventos em disco por um tempo configurável (padrão 7 dias) ou até um limite de tamanho. Topics compactados mantêm apenas o último evento por chave.

```
Topic: "order-events" (3 partições)

Partition 0: [offset 0: order.created userId=1] [offset 1: order.updated userId=3]
Partition 1: [offset 0: order.created userId=2] [offset 1: order.paid userId=2]
Partition 2: [offset 0: order.created userId=3] [offset 1: order.shipped userId=3]

Consumer Group A (serviço de e-mail): 3 consumidores → 1 partição cada
Consumer Group B (analytics):          1 consumidor  → lê as 3 partições
```

```typescript
import { Kafka, Partitioners, CompressionTypes } from 'kafkajs';

const kafka = new Kafka({
  clientId: 'order-service',
  brokers: [process.env.KAFKA_BROKER!],
  ssl: true,
  sasl: { mechanism: 'plain', username: process.env.KAFKA_USER!, password: process.env.KAFKA_PASS! },
});

// ---- Produtor ----
const producer = kafka.producer({
  createPartitioner: Partitioners.DefaultPartitioner,
  idempotent: true,           // garantia exatamente-uma-vez no produtor (retries não duplicam)
  transactionTimeout: 30000,
});
await producer.connect();

// Publica um evento
await producer.send({
  topic: 'order-events',
  messages: [
    {
      key: 'order_123',        // chave de partição — mesma chave sempre na mesma partição
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

// ---- Consumidor ----
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
        // Faz commit do offset apenas após processamento bem-sucedido
        await resolveOffset(offset);
        await heartbeat(); // evita timeout de sessão durante processamento lento
      } catch (err) {
        console.error(`Falha ao processar offset ${offset}:`, err);
        // Decisão: tentar novamente, ignorar ou enviar para um topic de erro separado
        // Na maioria dos casos: lança para parar e tentar o batch novamente
        throw err;
      }
    }

    await commitOffsetsIfNecessary();
  },
});

// ---- Seek / Replay (volta o consumidor a um offset específico) ----
consumer.on(consumer.events.GROUP_JOIN, async () => {
  // Volta ao início da partição 0 para fazer replay de todos os eventos
  await consumer.seek({
    topic: 'order-events',
    partition: 0,
    offset: '0',
  });
});
```

---

## 6. Kafka vs RabbitMQ vs BullMQ

| Fator | Kafka | RabbitMQ | BullMQ |
|-------|-------|----------|--------|
| Caso de uso | Streaming de eventos com alto throughput, replay, audit log | Roteamento complexo, filas de tarefas com baixa latência | Filas de jobs Node.js |
| Throughput | Milhões de mensagens/seg | Milhares-100k/seg | Milhares/seg por worker |
| Replay de mensagens | Sim (por offset) | Não (consumida = sumiu) | Limitado (só jobs com falha) |
| Roteamento | Chave de partição (simples) | Exchanges + bindings (complexo) | Nome da fila (simples) |
| Ordenação | Estrita por partição | Por fila com ressalvas | FIFO por fila |
| Retenção | Por tempo ou tamanho (dias) | Até consumir (ou TTL) | Até remoção |
| Complexidade de setup | Alta (Zookeeper ou KRaft) | Média | Baixa (só Redis) |
| Multi-consumidor | Consumer groups | Competição ou pub/sub | Workers compartilham fila |
| Suporte a linguagens | Todas | Todas (clientes AMQP) | Só Node.js |
| Melhor para | Event sourcing, stream processing, audit | Comunicação entre microserviços, roteamento de tarefas | Jobs em background em apps Node.js |

> Regra de ouro: se você está construindo um app Node.js e precisa de jobs em background (e-mails, relatórios, processamento de imagens), comece com BullMQ. Se precisar de streaming de eventos duráveis com replay entre serviços, use Kafka. Se precisar de roteamento complexo de mensagens com regras por mensagem, use RabbitMQ.

---

## 7. Padrões

### Padrão Outbox (Mensageria Transacional)

O padrão outbox garante que uma escrita no banco e uma publicação de mensagem sejam atômicas — sem transação distribuída.

```typescript
// Problema: escrever no banco e publicar no Kafka são duas operações separadas
// Se o app travar entre elas, a escrita no banco tem sucesso mas a mensagem se perde

// Solução: escreve a mensagem no banco (na mesma transação), depois relay para o Kafka

// Na mesma transação do banco:
async function createOrderWithOutbox(db: PrismaClient, orderData: CreateOrderInput) {
  return db.$transaction(async (tx) => {
    const order = await tx.order.create({ data: orderData });

    // Escreve evento na tabela outbox NA MESMA TRANSAÇÃO
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
  // Se a transação faz commit: tanto o pedido quanto o evento outbox são salvos atomicamente
  // Se a transação faz rollback: nenhum dos dois é salvo
}

// Processo de relay separado: lê do outbox e publica no Kafka
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

### Padrão Saga para Transações Distribuídas

```typescript
// Saga de processamento de pedido: coordena múltiplos serviços sem transação 2PC
// Cada passo tem uma ação compensatória (equivalente ao rollback)

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

// Worker processa passos da saga
const sagaWorker = new Worker('saga-step', async (job: Job) => {
  const { sagaId, step, orderId } = job.data;

  switch (step) {
    case 'RESERVE_INVENTORY':
      try {
        await inventoryService.reserve(orderId);
        await sagaQueue.add('saga-step', { sagaId, step: 'CHARGE_PAYMENT', orderId });
      } catch {
        // Compensa: nada a desfazer (reserva falhou)
        await markSagaFailed(sagaId, 'RESERVE_INVENTORY');
      }
      break;

    case 'CHARGE_PAYMENT':
      try {
        await paymentService.charge(orderId);
        await sagaQueue.add('saga-step', { sagaId, step: 'SEND_CONFIRMATION', orderId });
      } catch {
        // Compensa: libera reserva de estoque
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

## 8. Erros Comuns e Armadilhas

**Consumidores não idempotentes:**

```typescript
// ERRADO: adicionar 10 créditos duas vezes se a mensagem for reenviada
async function processReward(userId: string) {
  await db.users.update({
    where: { id: userId },
    data: { credits: { increment: 10 } },
  });
}

// CORRETO: idempotente usando o ID da mensagem
async function processReward(userId: string, messageId: string) {
  await db.$transaction(async (tx) => {
    const existing = await tx.processedMessages.findUnique({ where: { messageId } });
    if (existing) return;

    await tx.users.update({ where: { id: userId }, data: { credits: { increment: 10 } } });
    await tx.processedMessages.create({ data: { messageId } });
  });
}
```

**Mensagens poison pill:**

```typescript
// Uma poison pill é uma mensagem que sempre falha (dados inválidos, tipo não tratado)
// Sem proteção, ela bloqueia a fila indefinidamente (tentada para sempre)

// BullMQ: define número máximo de tentativas
await queue.add('process', badData, {
  attempts: 5,     // após 5 falhas, vai para o estado 'failed'
  backoff: { type: 'exponential', delay: 1000 },
});

// RabbitMQ: verifica contagem de tentativas e envia para DLX após máximo de retries
channel.consume('my-queue', async (msg) => {
  const deathCount = (msg?.properties.headers?.['x-death']?.[0]?.count ?? 0) as number;
  if (deathCount >= 5) {
    // Envia para fila de poison pills para inspeção manual
    channel.sendToQueue('poison-pills', msg!.content, {
      headers: { 'original-routing-key': msg!.fields.routingKey },
    });
    channel.ack(msg!); // confirma para remover da fila principal
    return;
  }
  // ... processa
});
```

**Consumidores de longa duração bloqueando o processamento:**

```typescript
// ERRADO: um job lento bloqueia todos os outros no worker
const worker = new Worker('reports', async (job) => {
  const data = await generateHugeReport(job.data); // 5 minutos!
  // Enquanto isso roda, nenhum outro job é processado
});

// CORRETO: define concorrência e configura timeouts
const worker = new Worker('reports', processReport, {
  connection,
  concurrency: 3,    // processa 3 jobs em paralelo
  lockDuration: 600_000, // lock de 10 min (deve exceder a duração máxima do job)
  lockRenewTime: 300_000, // renova o lock a cada 5 minutos
});
```

---

## 9. Monitoramento e Observabilidade

```typescript
// BullMQ: expõe métricas das filas
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

// Alerta se a DLQ (jobs com falha) crescer
setInterval(async () => {
  const failedCount = await emailQueue.getFailedCount();
  if (failedCount > 10) {
    await alerting.send(`Fila de e-mail tem ${failedCount} jobs com falha — verifique a DLQ`);
  }
}, 60_000);
```

---

## 10. Quando Usar / Não Usar

**Use filas de mensagens quando:**
- O processamento leva mais tempo que o aceitável para uma resposta HTTP (> 500ms)
- Você precisa absorver picos de tráfego (nivelamento de carga)
- Tarefas devem sobreviver a reinicializações do serviço (confiabilidade)
- Múltiplos serviços precisam reagir ao mesmo evento (pub/sub)
- Você precisa de entrega garantida + retry em caso de falha
- Jobs em background: e-mails, relatórios, processamento de imagens, sincronização de dados

**Talvez você NÃO precise de uma fila de mensagens quando:**
- A tarefa é rápida e síncrona está bem
- Você tem apenas um serviço e não precisa de desacoplamento
- A tarefa não precisa sobreviver a falhas (efêmera)
- Você já tem um banco e pode usá-lo como job store (filas baseadas em Postgres como pg-boss)
- Simplicidade é primordial e o overhead de um broker não se justifica

---

## 11. Cenário Real

**Problema:** Uma plataforma de e-commerce envia e-mails de confirmação de pedido de forma síncrona no handler HTTP `/checkout`. Durante a Black Friday, o limite de taxa do provedor de e-mail faz os envios demorarem 8 segundos. Todas as requisições de checkout ficam aguardando o serviço de e-mail. O fluxo de checkout inteiro está bloqueado — usuários ficam presos na tela de carregamento por 8 segundos e abandonam.

**Solução: E-mail assíncrono via BullMQ**

```typescript
// Antes: síncrono, bloqueante
app.post('/checkout', async (req, reply) => {
  const order = await orderService.create(req.body);
  await emailService.sendOrderConfirmation(order); // 8 SEGUNDOS bloqueando!
  return reply.status(201).send({ orderId: order.id });
});

// Depois: assíncrono, resposta instantânea
const emailQueue = new Queue('transactional-emails', { connection });

app.post('/checkout', async (req, reply) => {
  const order = await orderService.create(req.body);

  // Adiciona à fila — retorna em < 5ms
  await emailQueue.add(
    'order-confirmation',
    { orderId: order.id, userEmail: req.body.email },
    {
      jobId: `order-confirmation:${order.id}`, // deduplicação
      attempts: 5,
      backoff: { type: 'exponential', delay: 2000 },
    }
  );

  // Retorna imediatamente
  return reply.status(201).send({ orderId: order.id });
});

// Worker processa e-mails no próprio ritmo (respeita rate limits graciosamente)
const emailWorker = new Worker(
  'transactional-emails',
  async (job: Job) => {
    const { orderId, userEmail } = job.data;
    await emailService.sendOrderConfirmation({ orderId, userEmail });
  },
  {
    connection,
    concurrency: 10,
    limiter: { max: 50, duration: 1000 }, // respeita o rate limit: 50 e-mails/segundo
  }
);
```

**Resultado:** Tempo de resposta do checkout cai de 8 segundos para 120ms. A entrega de e-mails continua em background na taxa que o provedor permite. Nenhum e-mail é perdido — a fila tenta novamente automaticamente se o provedor estiver indisponível. Usuários completam o checkout instantaneamente.

---

## 12. Perguntas de Entrevista

**P1: Qual a diferença entre entrega at-least-once e exactly-once?**

At-least-once: o broker tenta a entrega até o consumidor confirmar. Se o consumidor processa mas trava antes de confirmar, a mensagem é reenviada — possíveis duplicatas. Consumidores devem ser idempotentes. A maioria dos sistemas de mensageria usa este padrão. Exactly-once: cada mensagem é entregue e processada exatamente uma vez — sem duplicatas, sem perda. Requer coordenação: Kafka consegue com produtores idempotentes + consumidores transacionais, com custo de performance significativo. Na prática: use at-least-once + consumidores idempotentes — dá semântica "efetivamente exactly-once" com infraestrutura mais simples.

**P2: O que é uma Dead Letter Queue e como usá-la?**

Uma DLQ recebe mensagens que falham no processamento após o número máximo de tentativas. Em vez de descartar mensagens com falha, elas são roteadas para a DLQ para inspeção e replay manual. No RabbitMQ, configure `x-dead-letter-exchange` na fila. No BullMQ, jobs com falha (após tentativas máximas) ficam no estado `failed` e são acessíveis via `queue.getFailed()`. A DLQ permite: análise da causa raiz (veja o payload + erro), replay manual após corrigir o bug, e alertas (profundidade da DLQ > 0 dispara alerta).

**P3: Como funcionam partições e consumer groups no Kafka?**

Um topic Kafka é dividido em N partições. Dentro de uma partição, eventos são estritamente ordenados e cada um tem um offset incremental. Um consumer group é um conjunto de consumidores compartilhando uma assinatura de um topic — partições são divididas entre os membros (uma partição → um consumidor por vez). Com 6 partições e 3 consumidores, cada consumidor recebe 2 partições — o paralelismo escala linearmente. Múltiplos consumer groups independentes recebem todos os eventos de forma independente. A implicação: para garantir que eventos relacionados (ex: todos os eventos do mesmo pedido) sejam processados em ordem, use o ID do pedido como chave de partição para que sempre caiam na mesma partição.

**P4: Quais são os tipos de exchange do RabbitMQ e quando usar cada um?**

Direct: roteia mensagens para filas cuja binding key coincide exatamente com a routing key. Use para filas de tarefas onde você direciona para uma fila específica. Fanout: transmite para todas as filas vinculadas, independente da routing key. Use para notificações onde cada inscrito deve receber cada mensagem. Topic: roteia baseado em padrões wildcard (`order.*` casa com `order.created`, `order.updated`; `#` casa com zero ou mais segmentos). Use quando inscritos precisam filtrar por padrões de tipo de evento. Headers: roteia baseado em cabeçalhos em vez de routing key. Raramente usado — complexo e lento em comparação com topic routing.

**P5: Como o BullMQ trata retries e falhas?**

Configure `attempts` (contagem máxima de tentativas) e `backoff` (estratégia de delay: fixo ou exponencial) ao adicionar um job. Em caso de falha, BullMQ tenta novamente o job após o delay de backoff. Se o job falhar `attempts` vezes, vai para o estado `failed` (DLQ do BullMQ). Inspecione jobs com falha via `queue.getFailed()`, veja a mensagem de erro e chame `job.retry()` para replay. Use `backoff: { type: 'exponential', delay: 2000 }` na maioria dos casos — evita sobrecarregar um serviço se recuperando. Use `removeOnFail: 500` para manter apenas as últimas 500 falhas (evita crescimento da memória Redis).

**P6: Por que consumidores precisam ser idempotentes?**

Porque a entrega at-least-once garante que mensagens podem chegar mais de uma vez: consumidor trava antes de confirmar, problemas de rede causam reentrega, replay manual da DLQ. Um consumidor idempotente produz o mesmo resultado independente de quantas vezes a mesma mensagem é processada. Padrões de implementação: (1) Verifica tabela de deduplicação no banco antes de processar (insere `messageId` atomicamente com o efeito colateral). (2) Usa atualizações condicionais (`WHERE processed = FALSE`). (3) Para operações financeiras: usa chaves de idempotência que previnem cobranças duplas. Consumidores não idempotentes com entrega at-least-once = estado incorreto da aplicação.

**P7: Quando você escolheria Kafka em vez de RabbitMQ?**

Escolha Kafka quando: você precisa de alto throughput (milhões de eventos/segundo), precisa de replay de eventos (consumidores podem buscar eventos passados), precisa de múltiplos consumer groups independentes processando o mesmo stream de eventos, seu caso de uso é event sourcing ou CQRS, ou precisa de retenção de eventos a longo prazo. Escolha RabbitMQ quando: você precisa de roteamento complexo (topic exchanges, header routing), baixa latência de entrega é crítica, a mensagem precisa ser consumida uma vez e sumida, ou precisa de decisões de roteamento por mensagem. Escolha BullMQ quando: você está em Node.js, já usa Redis e precisa de processamento de jobs em background sem a complexidade operacional do Kafka ou RabbitMQ.

---

## 13. Exercícios

**Exercício 1: Workflow BullMQ**

Construa um pipeline de processamento de documentos usando BullMQ que:
- Aceita uploads de arquivos PDF
- Passo 1 da fila: extrai texto (lento, intensivo em CPU)
- Passo 2 da fila: executa análise de sentimento no texto extraído (depende do passo 1)
- Passo 3 da fila: armazena resultados no banco e notifica o usuário via WebSocket (depende do passo 2)
- Use BullMQ Flows para dependências entre passos
- Exponha endpoint `GET /jobs/:jobId/status` retornando progresso e passo atual

*Dica:* Use `FlowProducer` para dependências pai/filho. Use `job.updateProgress()` em cada etapa.

**Exercício 2: Processamento idempotente de pedido**

Implemente um consumidor de pagamento de pedido totalmente idempotente que:
- Recebe `{ orderId, userId, amount, messageId }` de uma fila
- NÃO deve cobrar o cliente duas vezes se a mensagem for entregue mais de uma vez
- Deve lidar com: banco indisponível, provedor de pagamento indisponível, reentrega de mensagem após sucesso parcial
- Escreva testes que simulam entrega de mensagem duplicada

*Dica:* Use uma tabela `payment_idempotency` com `(messageId, status)`. Encapsule a verificação+cobrança+registro em uma transação de banco.

**Exercício 3: Consumidor Kafka com gerenciamento de offset**

Construa um consumidor Kafka que:
- Lê de um topic `user-events` (3 partições)
- Processa eventos em ordem por userId (mesmo userId sempre na mesma partição via chave)
- Faz commit de offsets manualmente somente após escrita bem-sucedida no banco (sem auto-commit)
- Trata rebalanceamento graciosamente (salva e restaura offsets)
- Expõe endpoint de health mostrando atribuições de partição atuais e últimos offsets confirmados

*Dica:* Use modo `eachBatch` com resolução manual de offset. Inscreva-se em `consumer.events.GROUP_JOIN` para eventos de rebalanceamento.

---

## 14. Leituras Complementares

- [Documentação BullMQ](https://docs.bullmq.io/)
- [Tutoriais RabbitMQ](https://www.rabbitmq.com/getstarted.html)
- [Documentação Apache Kafka](https://kafka.apache.org/documentation/)
- [KafkaJS — cliente Kafka para Node.js](https://kafka.js.org/)
- [Enterprise Integration Patterns (Gregor Hohpe)](https://www.enterpriseintegrationpatterns.com/)
- [Especificação do modelo AMQP 0-9-1](https://www.rabbitmq.com/tutorials/amqp-concepts)
- [The Log: O que todo engenheiro de software deveria saber sobre dados em tempo real (Jay Kreps)](https://engineering.linkedin.com/distributed-systems/log-what-every-software-engineer-should-know-about-real-time-datas-unifying)
- [Padrão Outbox — Microservices.io](https://microservices.io/patterns/data/transactional-outbox.html)
- [Padrão Saga — Microservices.io](https://microservices.io/patterns/data/saga.html)
- Livro: *Designing Data-Intensive Applications* por Martin Kleppmann (capítulos sobre mensageria)
