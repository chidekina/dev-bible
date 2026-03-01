# Sistemas Distribuídos

## Visão Geral

Um sistema distribuído é uma coleção de computadores que aparece para os usuários como um único sistema. Todo sistema que roda em mais de um processo — até mesmo uma API e um banco de dados — é tecnicamente distribuído. Em escala, os desafios ficam agudos: relógios derivam, redes falham, mensagens chegam fora de ordem e falhas parciais produzem estados que não são nem totalmente bem-sucedidos nem totalmente falhos. Este capítulo cobre os conceitos fundamentais que todo engenheiro backend deve entender: consenso, idempotência, o padrão saga e problemas com relógio.

---

## Pré-requisitos

- Track 08: Teorema CAP
- Track 08: Message Queues (o padrão saga usa queues)
- Track 03: Transações de banco de dados

---

## Conceitos Fundamentais

### Os problemas fundamentais da distribuição

1. **Falhas parciais:** um componente falha enquanto outros continuam. O sistema está em estado indefinido.
2. **Não confiabilidade de rede:** mensagens podem ser atrasadas, duplicadas ou perdidas.
3. **Clock skew:** relógios em máquinas diferentes derivam; você não pode assumir ordenação com base em timestamps.
4. **Acesso concorrente:** múltiplos processos modificam estado compartilhado simultaneamente.

### Falácias da computação distribuída

Oito suposições que engenheiros fazem erroneamente:
1. A rede é confiável
2. A latência é zero
3. A largura de banda é infinita
4. A rede é segura
5. A topologia não muda
6. Há um único administrador
7. O custo de transporte é zero
8. A rede é homogênea

Todo sistema distribuído deve ser projetado assumindo que todas as oito falácias são falsas.

---

## Exemplos Práticos

### Idempotência

O conceito mais importante de sistemas distribuídos na prática. Uma operação é idempotente se realizá-la múltiplas vezes produz o mesmo resultado que realizá-la uma vez.

**Por que importa:** redes falham no meio de requisições. O cliente pode tentar novamente. Sem idempotência, retries causam duplicatas (cobranças duplas, emails duplicados, pedidos duplicados).

```typescript
// src/services/payment.service.ts
export class PaymentService {
  async charge(params: {
    userId: string;
    orderId: string;
    amount: number;
    currency: string;
    idempotencyKey: string; // gerado pelo chamador (geralmente: userId + orderId)
  }) {
    // Verifica se essa operação já foi realizada
    const existing = await db.payment.findUnique({
      where: { idempotencyKey: params.idempotencyKey },
    });

    if (existing) {
      // Retorna o mesmo resultado — seguro chamar múltiplas vezes
      return existing;
    }

    // Realiza a cobrança
    const charge = await stripe.charges.create(
      {
        amount: params.amount,
        currency: params.currency,
        customer: params.userId,
        metadata: { orderId: params.orderId },
      },
      { idempotencyKey: params.idempotencyKey } // o Stripe também armazena isso
    );

    // Armazena no nosso DB com a idempotency key
    return db.payment.create({
      data: {
        idempotencyKey: params.idempotencyKey,
        orderId: params.orderId,
        userId: params.userId,
        stripeChargeId: charge.id,
        amount: params.amount,
        status: 'succeeded',
      },
    });
  }
}
```

```typescript
// Gerando idempotency keys
function paymentIdempotencyKey(userId: string, orderId: string, attempt: number = 1): string {
  return `payment:${userId}:${orderId}:${attempt}`;
}
```

### Padrão saga — gerenciando transações distribuídas

Uma saga é uma sequência de transações locais coordenadas entre serviços. Quando uma etapa falha, transações compensatórias desfazem o trabalho das etapas anteriores.

**Exemplo: realizar um pedido (reservar voo, hotel e carro em sequência)**

```typescript
// Saga baseada em coreografia (orientada a eventos)
// Cada serviço publica eventos; outros serviços reagem

// Etapa 1: serviço de pedidos cria pedido e publica OrderPlaced
async function placeOrder(data: PlaceOrderDto) {
  const order = await db.order.create({
    data: { ...data, status: 'pending' },
  });

  await eventBus.publish('order.placed', { orderId: order.id, items: data.items });
  return order;
}

// Etapa 2: serviço de estoque escuta OrderPlaced
eventBus.subscribe('order.placed', async ({ orderId, items }) => {
  try {
    // Reserva estoque
    await reserveInventory(items);
    await eventBus.publish('inventory.reserved', { orderId });
  } catch {
    // Compensação: não foi possível reservar → emite evento de falha
    await eventBus.publish('inventory.reservation_failed', { orderId });
  }
});

// Etapa 3: serviço de pagamento escuta InventoryReserved
eventBus.subscribe('inventory.reserved', async ({ orderId }) => {
  const order = await db.order.findUnique({ where: { id: orderId } });
  try {
    await chargeCustomer(order);
    await eventBus.publish('payment.succeeded', { orderId });
  } catch {
    await eventBus.publish('payment.failed', { orderId });
  }
});

// Compensação: se o pagamento falhar, libera o estoque
eventBus.subscribe('payment.failed', async ({ orderId }) => {
  const order = await db.order.findUnique({ where: { id: orderId } });
  await releaseInventory(order.items); // transação compensatória
  await db.order.update({ where: { id: orderId }, data: { status: 'failed' } });
});
```

**Saga baseada em orquestração (coordenador centralizado):**

```typescript
// src/sagas/place-order.saga.ts
export class PlaceOrderSaga {
  async execute(data: PlaceOrderDto) {
    const sagaId = crypto.randomUUID();
    const steps: Array<() => Promise<void>> = [];
    const compensations: Array<() => Promise<void>> = [];

    try {
      // Etapa 1: Cria pedido
      const order = await orderService.create(data);
      compensations.push(() => orderService.cancel(order.id));

      // Etapa 2: Reserva estoque
      const reservation = await inventoryService.reserve(order.items);
      compensations.push(() => inventoryService.release(reservation.id));

      // Etapa 3: Processa pagamento
      const payment = await paymentService.charge({
        orderId: order.id,
        amount: order.total,
        idempotencyKey: `${sagaId}:payment`,
      });
      compensations.push(() => paymentService.refund(payment.id));

      // Etapa 4: Confirma pedido
      await orderService.confirm(order.id);

      return order;
    } catch (error) {
      // Executa compensações em ordem inversa
      for (const compensate of compensations.reverse()) {
        try {
          await compensate();
        } catch (compensationError) {
          // Loga falha de compensação — requer intervenção manual
          logger.error({ sagaId, error: compensationError }, 'Compensation failed');
        }
      }
      throw error;
    }
  }
}
```

### Locks distribuídos

Previne modificação concorrente de recursos compartilhados:

```typescript
// src/lib/distributed-lock.ts
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

export class DistributedLock {
  private static readonly DEFAULT_TTL = 10000; // 10 segundos

  static async acquire(key: string, ttlMs = this.DEFAULT_TTL): Promise<string | null> {
    const lockValue = crypto.randomUUID();
    // SET key value NX EX — só define se a chave não existir
    const result = await redis.set(`lock:${key}`, lockValue, {
      NX: true,
      PX: ttlMs,
    });
    return result === 'OK' ? lockValue : null;
  }

  static async release(key: string, lockValue: string): Promise<boolean> {
    // Script Lua garante que só deletamos se somos donos do lock (atomicamente)
    const script = `
      if redis.call("GET", KEYS[1]) == ARGV[1] then
        return redis.call("DEL", KEYS[1])
      else
        return 0
      end
    `;
    const result = await redis.eval(script, { keys: [`lock:${key}`], arguments: [lockValue] });
    return result === 1;
  }

  static async withLock<T>(
    key: string,
    fn: () => Promise<T>,
    ttlMs = this.DEFAULT_TTL
  ): Promise<T> {
    const lockValue = await this.acquire(key, ttlMs);
    if (!lockValue) {
      throw new Error(`Could not acquire lock for key: ${key}`);
    }

    try {
      return await fn();
    } finally {
      await this.release(key, lockValue);
    }
  }
}

// Uso: previne reserva concorrente de estoque para o mesmo produto
const result = await DistributedLock.withLock(
  `inventory:${productId}`,
  async () => {
    const product = await db.product.findUnique({ where: { id: productId } });
    if (product.stock < quantity) throw new Error('Insufficient stock');
    return db.product.update({
      where: { id: productId },
      data: { stock: { decrement: quantity } },
    });
  }
);
```

### Clock skew e ordenação com relógios lógicos

Timestamps de relógio de parede de máquinas diferentes não podem ser usados para ordenar eventos:

```typescript
// Problema: Alice e Bob escrevem simultaneamente
// Máquina da Alice: escreve às 10:00:00.001
// Máquina do Bob: escreve às 10:00:00.000 (mas o relógio do Bob está 10ms atrasado)
// Ordenar por timestamp diz que Bob escreveu primeiro — errado

// Solução: Timestamp de Lamport (relógio lógico)
class LamportClock {
  private counter = 0;

  tick(): number {
    return ++this.counter;
  }

  // Ao receber uma mensagem, avança o relógio além do timestamp do remetente
  receive(senderTimestamp: number): number {
    this.counter = Math.max(this.counter, senderTimestamp) + 1;
    return this.counter;
  }
}

// Na prática: use sequences do banco de dados ou ULIDs para ordenação
import { ulid } from 'ulid';

// ULID: Universally Unique Lexicographically Sortable Identifier
// Monotonicamente crescente dentro de um milissegundo → seguro para ordenar
const eventId = ulid(); // '01HX4T6QYWZ9JKXBQ9NZ5Q74VT'
// Eventos podem ser ordenados pelo ULID sem problemas de relógio
```

### Padrão circuit breaker

Previne falhas em cascata quando um serviço downstream está com problemas:

```typescript
// src/lib/circuit-breaker.ts
type CircuitState = 'closed' | 'open' | 'half-open';

export class CircuitBreaker {
  private state: CircuitState = 'closed';
  private failureCount = 0;
  private lastFailureTime = 0;

  constructor(
    private readonly failureThreshold = 5,   // abre após 5 falhas consecutivas
    private readonly recoveryTimeMs = 30000    // tenta novamente após 30 segundos
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      const timeSinceFailure = Date.now() - this.lastFailureTime;
      if (timeSinceFailure < this.recoveryTimeMs) {
        throw new Error('Circuit breaker is open — request rejected');
      }
      // Transiciona para half-open para testar o serviço
      this.state = 'half-open';
    }

    try {
      const result = await fn();
      this.onSuccess();
      return result;
    } catch (error) {
      this.onFailure();
      throw error;
    }
  }

  private onSuccess() {
    this.failureCount = 0;
    this.state = 'closed';
  }

  private onFailure() {
    this.failureCount++;
    this.lastFailureTime = Date.now();
    if (this.failureCount >= this.failureThreshold) {
      this.state = 'open';
    }
  }
}

// Uso
const paymentBreaker = new CircuitBreaker(5, 30000);

async function chargeWithBreaker(params: ChargeParams) {
  return paymentBreaker.execute(() => stripe.charges.create(params));
}
```

---

## Padrões e Boas Práticas

- **Projete para idempotência desde o início** — atribua idempotency keys no ponto de chamada, verifique antes de agir
- **Use o padrão outbox** — escreva eventos no banco de dados na mesma transação que os dados, então publique a partir do DB
- **Evite cadeias síncronas com mais de 2 hops** — cada hop multiplica latência e probabilidade de falha
- **Defina timeouts em todas as chamadas de rede** — nunca espere indefinidamente por um serviço downstream
- **Use ULIDs ou sequences do banco** para ordenação de eventos — não timestamps de relógio de parede

---

## Anti-Padrões a Evitar

- Usar timestamps de relógio de parede para ordenar eventos distribuídos
- Transações distribuídas (2PC) — bloqueiam em caso de falha; use sagas em vez disso
- Fazer retry sem idempotência — causa duplicatas
- Não definir timeouts — um serviço downstream lento bloqueia todo o seu pool de processos
- Assumir que porque uma requisição retornou erro, a operação não aconteceu — ela pode ter acontecido parcialmente

---

## Debugging e Troubleshooting

**"Usuários estão sendo cobrados duas vezes após retries de pagamento"**
Idempotency keys ausentes. Adicione uma idempotency key às suas chamadas de pagamento e verifique registros existentes antes de processar.

**"Saga está presa em estado parcial"**
Uma etapa de compensação falhou. Implemente uma tabela "saga log" que registra o resultado de cada etapa. Adicione um job em background para encontrar e resolver sagas travadas.

**"Lock distribuído não está prevenindo acesso concorrente"**
O TTL do lock pode ser menor que sua seção crítica. Aumente o TTL. Ou a liberação do lock falhou — verifique se seu script Lua é atômico.

---

## Leitura Complementar

- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) — Capítulos 8 e 9
- [The Outbox Pattern — Chris Richardson](https://microservices.io/patterns/data/transactional-outbox.html)
- [Padrão Saga](https://microservices.io/patterns/data/saga.html)
- [AWS Blog: Idempotência em sistemas distribuídos](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- Track 08: [Message Queues](message-queues.md)

---

## Resumo

Sistemas distribuídos são difíceis porque redes falham, relógios derivam e falhas parciais deixam sistemas em estados indefinidos. As ferramentas mais práticas para gerenciar essa complexidade são: idempotência (torna retries seguros), sagas (substitui transações distribuídas por sequências compensáveis), circuit breakers (previnem falhas em cascata), locks distribuídos (previnem race conditions) e relógios lógicos (fornecem ordenação segura de eventos). A maioria desses padrões pode ser implementada em uma stack Node.js/Redis/Postgres sem infraestrutura especializada. A mudança mental fundamental é aceitar que falhas são normais e projetar o sistema para tratá-las graciosamente em vez de assumir que não vão ocorrer.
