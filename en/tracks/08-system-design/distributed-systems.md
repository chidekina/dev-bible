# Distributed Systems

## Overview

A distributed system is a collection of computers that appear to users as a single system. Every system that runs across more than one process — even just an API and a database — is technically distributed. At scale, the challenges become acute: clocks drift, networks fail, messages are reordered, and partial failures produce states that are neither fully succeeded nor fully failed. This chapter covers the core concepts that every backend engineer must understand: consensus, idempotency, the saga pattern, and clock problems.

---

## Prerequisites

- Track 08: CAP Theorem
- Track 08: Message Queues (saga pattern uses queues)
- Track 03: Database transactions

---

## Core Concepts

### The fundamental problems of distribution

1. **Partial failures:** one component fails while others continue. The system is in a undefined state.
2. **Network unreliability:** messages can be delayed, duplicated, or lost.
3. **Clock skew:** clocks on different machines drift; you cannot assume ordering based on timestamps.
4. **Concurrent access:** multiple processes modify shared state simultaneously.

### Fallacies of distributed computing

Eight assumptions that engineers mistakenly make:
1. The network is reliable
2. Latency is zero
3. Bandwidth is infinite
4. The network is secure
5. Topology does not change
6. There is one administrator
7. Transport cost is zero
8. The network is homogeneous

Every distributed system must be designed assuming all eight fallacies are false.

---

## Hands-On Examples

### Idempotency

The most important distributed systems concept in practice. An operation is idempotent if performing it multiple times produces the same result as performing it once.

**Why it matters:** networks fail mid-request. The client may retry. Without idempotency, retries cause duplicates (double charges, duplicate emails, duplicate orders).

```typescript
// src/services/payment.service.ts
export class PaymentService {
  async charge(params: {
    userId: string;
    orderId: string;
    amount: number;
    currency: string;
    idempotencyKey: string; // caller generates this (usually: userId + orderId)
  }) {
    // Check if this operation was already performed
    const existing = await db.payment.findUnique({
      where: { idempotencyKey: params.idempotencyKey },
    });

    if (existing) {
      // Return the same result — safe to call multiple times
      return existing;
    }

    // Perform the charge
    const charge = await stripe.charges.create(
      {
        amount: params.amount,
        currency: params.currency,
        customer: params.userId,
        metadata: { orderId: params.orderId },
      },
      { idempotencyKey: params.idempotencyKey } // Stripe also stores this
    );

    // Store in our DB with idempotency key
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
// Generating idempotency keys
function paymentIdempotencyKey(userId: string, orderId: string, attempt: number = 1): string {
  return `payment:${userId}:${orderId}:${attempt}`;
}
```

### Saga pattern — managing distributed transactions

A saga is a sequence of local transactions coordinated across services. When one step fails, compensating transactions undo the work done by previous steps.

**Example: placing an order (booking a flight, hotel, and car in sequence)**

```typescript
// Choreography-based saga (event-driven)
// Each service publishes events; other services react

// Step 1: Order service creates order and publishes OrderPlaced
async function placeOrder(data: PlaceOrderDto) {
  const order = await db.order.create({
    data: { ...data, status: 'pending' },
  });

  await eventBus.publish('order.placed', { orderId: order.id, items: data.items });
  return order;
}

// Step 2: Inventory service listens for OrderPlaced
eventBus.subscribe('order.placed', async ({ orderId, items }) => {
  try {
    // Reserve inventory
    await reserveInventory(items);
    await eventBus.publish('inventory.reserved', { orderId });
  } catch {
    // Compensation: cannot reserve → emit failure event
    await eventBus.publish('inventory.reservation_failed', { orderId });
  }
});

// Step 3: Payment service listens for InventoryReserved
eventBus.subscribe('inventory.reserved', async ({ orderId }) => {
  const order = await db.order.findUnique({ where: { id: orderId } });
  try {
    await chargeCustomer(order);
    await eventBus.publish('payment.succeeded', { orderId });
  } catch {
    await eventBus.publish('payment.failed', { orderId });
  }
});

// Compensation: if payment fails, release inventory
eventBus.subscribe('payment.failed', async ({ orderId }) => {
  const order = await db.order.findUnique({ where: { id: orderId } });
  await releaseInventory(order.items); // compensating transaction
  await db.order.update({ where: { id: orderId }, data: { status: 'failed' } });
});
```

**Orchestration-based saga (centralized coordinator):**

```typescript
// src/sagas/place-order.saga.ts
export class PlaceOrderSaga {
  async execute(data: PlaceOrderDto) {
    const sagaId = crypto.randomUUID();
    const steps: Array<() => Promise<void>> = [];
    const compensations: Array<() => Promise<void>> = [];

    try {
      // Step 1: Create order
      const order = await orderService.create(data);
      compensations.push(() => orderService.cancel(order.id));

      // Step 2: Reserve inventory
      const reservation = await inventoryService.reserve(order.items);
      compensations.push(() => inventoryService.release(reservation.id));

      // Step 3: Charge payment
      const payment = await paymentService.charge({
        orderId: order.id,
        amount: order.total,
        idempotencyKey: `${sagaId}:payment`,
      });
      compensations.push(() => paymentService.refund(payment.id));

      // Step 4: Confirm order
      await orderService.confirm(order.id);

      return order;
    } catch (error) {
      // Run compensations in reverse order
      for (const compensate of compensations.reverse()) {
        try {
          await compensate();
        } catch (compensationError) {
          // Log compensation failure — requires manual intervention
          logger.error({ sagaId, error: compensationError }, 'Compensation failed');
        }
      }
      throw error;
    }
  }
}
```

### Distributed locks

Prevent concurrent modification of shared resources:

```typescript
// src/lib/distributed-lock.ts
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });

export class DistributedLock {
  private static readonly DEFAULT_TTL = 10000; // 10 seconds

  static async acquire(key: string, ttlMs = this.DEFAULT_TTL): Promise<string | null> {
    const lockValue = crypto.randomUUID();
    // SET key value NX EX — only set if key does not exist
    const result = await redis.set(`lock:${key}`, lockValue, {
      NX: true,
      PX: ttlMs,
    });
    return result === 'OK' ? lockValue : null;
  }

  static async release(key: string, lockValue: string): Promise<boolean> {
    // Lua script ensures we only delete if we own the lock (atomically)
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

// Usage: prevent concurrent inventory reservation for the same product
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

### Clock skew and ordering with logical clocks

Wall-clock timestamps from different machines cannot be used to order events:

```typescript
// Problem: Alice and Bob write simultaneously
// Alice's machine: write at 10:00:00.001
// Bob's machine: write at 10:00:00.000 (but Bob's clock is 10ms behind)
// Ordering by timestamp says Bob wrote first — wrong

// Solution: Lamport timestamp (logical clock)
class LamportClock {
  private counter = 0;

  tick(): number {
    return ++this.counter;
  }

  // When receiving a message, advance clock past the sender's timestamp
  receive(senderTimestamp: number): number {
    this.counter = Math.max(this.counter, senderTimestamp) + 1;
    return this.counter;
  }
}

// In practice: use database sequences or ULIDs for ordering
import { ulid } from 'ulid';

// ULID: Universally Unique Lexicographically Sortable Identifier
// Monotonically increasing within a millisecond → safe to sort
const eventId = ulid(); // '01HX4T6QYWZ9JKXBQ9NZ5Q74VT'
// Events can be ordered by their ULID without clock issues
```

### Circuit breaker pattern

Prevent cascading failures when a downstream service is unhealthy:

```typescript
// src/lib/circuit-breaker.ts
type CircuitState = 'closed' | 'open' | 'half-open';

export class CircuitBreaker {
  private state: CircuitState = 'closed';
  private failureCount = 0;
  private lastFailureTime = 0;

  constructor(
    private readonly failureThreshold = 5,   // open after 5 consecutive failures
    private readonly recoveryTimeMs = 30000    // try again after 30 seconds
  ) {}

  async execute<T>(fn: () => Promise<T>): Promise<T> {
    if (this.state === 'open') {
      const timeSinceFailure = Date.now() - this.lastFailureTime;
      if (timeSinceFailure < this.recoveryTimeMs) {
        throw new Error('Circuit breaker is open — request rejected');
      }
      // Transition to half-open to test the service
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

// Usage
const paymentBreaker = new CircuitBreaker(5, 30000);

async function chargeWithBreaker(params: ChargeParams) {
  return paymentBreaker.execute(() => stripe.charges.create(params));
}
```

---

## Common Patterns & Best Practices

- **Design for idempotency from day one** — assign idempotency keys at the call site, check before acting
- **Use the outbox pattern** — write events to the database in the same transaction as data, then publish from the DB
- **Avoid synchronous chains longer than 2 hops** — each hop multiplies latency and failure probability
- **Set timeouts on all network calls** — never wait indefinitely for a downstream service
- **Use ULIDs or database sequences** for event ordering — not wall-clock timestamps

---

## Anti-Patterns to Avoid

- Using wall-clock timestamps to order distributed events
- Distributed transactions (2PC) — they block on failure; use sagas instead
- Retrying without idempotency — causes duplicates
- Not setting timeouts — one slow downstream service blocks your entire process pool
- Assuming that because a request returned an error, the operation did not happen — it may have half-happened

---

## Debugging & Troubleshooting

**"Users are getting duplicate charges after payment retries"**
Missing idempotency keys. Add an idempotency key to your payment calls and check for existing records before processing.

**"Saga is stuck in a partial state"**
A compensation step failed. Implement a "saga log" table that records each step outcome. Add a background job to find and resolve stuck sagas.

**"Distributed lock is not preventing concurrent access"**
The lock TTL may be shorter than your critical section. Increase the TTL. Or the lock release failed — check your Lua script is atomic.

---

## Further Reading

- [Designing Data-Intensive Applications — Martin Kleppmann](https://dataintensive.net/) — Chapters 8 and 9
- [The Outbox Pattern — Chris Richardson](https://microservices.io/patterns/data/transactional-outbox.html)
- [Saga pattern](https://microservices.io/patterns/data/saga.html)
- [AWS Blog: Idempotency in distributed systems](https://aws.amazon.com/builders-library/making-retries-safe-with-idempotent-APIs/)
- Track 08: [Message Queues](message-queues.md)

---

## Summary

Distributed systems are hard because networks fail, clocks drift, and partial failures leave systems in undefined states. The most practical tools for managing this complexity are: idempotency (makes retries safe), sagas (replaces distributed transactions with compensatable sequences), circuit breakers (prevent cascading failures), distributed locks (prevent race conditions), and logical clocks (provide safe event ordering). Most of these patterns can be implemented in a Node.js/Redis/Postgres stack without specialized infrastructure. The key mental shift is accepting that failures are normal and designing the system to handle them gracefully rather than assuming they will not occur.
