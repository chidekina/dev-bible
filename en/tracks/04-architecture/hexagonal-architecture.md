# Hexagonal Architecture

## 1. What & Why

Hexagonal Architecture — also called "Ports and Adapters" — was coined by Alistair Cockburn in 2005. The central idea is to isolate the application core (your business logic) from everything outside it: databases, HTTP, message queues, CLI tools, third-party services.

The name "hexagonal" is metaphorical — the six sides represent multiple ports, not literally six of them. What matters is the shape: a core surrounded by pluggable adapters.

> 💡 The driving insight: if you can run your entire domain logic with zero external dependencies (no database, no HTTP), you have a healthy hexagonal architecture.

Why Hexagonal Architecture?

- **Testability without infrastructure:** Swap real databases for in-memory fakes; test business logic in milliseconds.
- **Technology agnosticism:** The core does not care whether it is driven by HTTP, CLI, or a message queue.
- **Parallel development:** Infrastructure adapters and business logic can be developed concurrently.
- **Explicit boundaries:** Every interaction with the outside world goes through a named port.

Hexagonal Architecture and Clean Architecture solve the same problem. Clean Architecture is more prescriptive about layers; Hexagonal Architecture is more focused on the port/adapter vocabulary.

---

## 2. Core Concepts

### The Three Zones

```
┌─────────────────────────────────────────────────────────┐
│  OUTSIDE                                                │
│  (HTTP, CLI, DB, Email, Queue, Third-party APIs)        │
│                                                         │
│         [Primary Adapter]     [Secondary Adapter]       │
│                │                       │                │
│         Primary Port            Secondary Port          │
│                │                       │                │
│  ┌─────────────▼───────────────────────▼─────────────┐ │
│  │              APPLICATION CORE                      │ │
│  │         (Domain Logic + Use Cases)                 │ │
│  └────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Ports

A **port** is an interface — a contract that defines how the outside world communicates with the core (or the core with the outside).

- **Primary ports (driving ports):** Called *by* external actors to trigger application behavior. Examples: `IOrderService`, `ICheckoutService`. HTTP controllers, CLI commands, and test suites are primary adapters that call these ports.
- **Secondary ports (driven ports):** Called *by* the application core to reach external systems. Examples: `IOrderRepository`, `IPaymentGateway`, `IEmailSender`. Database adapters and third-party integrations implement these.

### Adapters

An **adapter** translates between an external technology and the port interface.

- **Primary adapters:** A Fastify route handler that calls `IOrderService.placeOrder()`. A CLI command that calls the same service. Both are primary adapters for the same primary port.
- **Secondary adapters:** `PrismaOrderRepository implements IOrderRepository`. `StripePaymentAdapter implements IPaymentGateway`. `NodemailerEmailAdapter implements IEmailSender`.

> ⚠️ Ports are defined *inside* the application core. Adapters live *outside* and depend on the core, never the reverse. This is the same Dependency Inversion Principle as in SOLID — applied at the architectural level.

---

## 3. How It Works

Request lifecycle in a Hexagonal application:

```
1. HTTP POST /orders
       │
       ▼
2. FastifyOrderController (Primary Adapter)
   - Validates HTTP request
   - Maps to PlaceOrderCommand
   - Calls IOrderService.placeOrder()
       │
       ▼
3. OrderService (Application Core)
   - Enforces business rules
   - Calls IOrderRepository.findById()
   - Calls IPaymentGateway.charge()
   - Calls INotificationPort.notifyUser()
   - Returns OrderPlacedResult
       │
       ▼
4. Adapters execute:
   - PrismaOrderRepository → PostgreSQL
   - StripePaymentAdapter → Stripe API
   - SESNotificationAdapter → AWS SES
       │
       ▼
5. Result flows back through the layers
6. Controller maps to HTTP response
```

The core (step 3) has no knowledge of Fastify, Prisma, Stripe, or SES. It only knows its own port interfaces.

---

## 4. Code Examples (TypeScript)

### Domain: Order Aggregate

```typescript
// src/core/domain/Order.ts
export interface OrderItem {
  productId: string;
  quantity: number;
  unitPrice: number;
}

export class Order {
  private _items: OrderItem[] = [];
  private _status: 'pending' | 'confirmed' | 'cancelled' = 'pending';

  constructor(
    public readonly id: string,
    public readonly customerId: string,
    private readonly createdAt: Date = new Date(),
  ) {}

  addItem(item: OrderItem): void {
    if (this._status !== 'pending') {
      throw new Error('Cannot modify a confirmed or cancelled order');
    }
    if (item.quantity <= 0) {
      throw new Error('Quantity must be positive');
    }
    this._items.push(item);
  }

  get total(): number {
    return this._items.reduce((sum, i) => sum + i.unitPrice * i.quantity, 0);
  }

  get items(): ReadonlyArray<OrderItem> { return this._items; }
  get status(): string { return this._status; }

  confirm(): void {
    if (this._items.length === 0) throw new Error('Cannot confirm empty order');
    this._status = 'confirmed';
  }
}
```

### Secondary Ports (defined in the core)

```typescript
// src/core/ports/secondary/IOrderRepository.ts
import { Order } from '../domain/Order';

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  save(order: Order): Promise<void>;
}
```

```typescript
// src/core/ports/secondary/IPaymentGateway.ts
export interface PaymentResult {
  transactionId: string;
  status: 'approved' | 'declined';
}

export interface IPaymentGateway {
  charge(customerId: string, amount: number, currency: string): Promise<PaymentResult>;
  refund(transactionId: string, amount: number): Promise<void>;
}
```

```typescript
// src/core/ports/secondary/INotificationPort.ts
export interface INotificationPort {
  notifyOrderConfirmed(customerId: string, orderId: string, total: number): Promise<void>;
}
```

### Primary Port (defined in the core)

```typescript
// src/core/ports/primary/IOrderService.ts
import { Order } from '../domain/Order';

export interface PlaceOrderCommand {
  customerId: string;
  items: Array<{ productId: string; quantity: number; unitPrice: number }>;
}

export interface PlaceOrderResult {
  orderId: string;
  total: number;
  transactionId: string;
}

export interface IOrderService {
  placeOrder(command: PlaceOrderCommand): Promise<PlaceOrderResult>;
  getOrder(orderId: string): Promise<Order | null>;
}
```

### Application Core: OrderService

```typescript
// src/core/OrderService.ts
import { randomUUID } from 'crypto';
import { Order } from './domain/Order';
import { IOrderRepository } from './ports/secondary/IOrderRepository';
import { IPaymentGateway } from './ports/secondary/IPaymentGateway';
import { INotificationPort } from './ports/secondary/INotificationPort';
import { IOrderService, PlaceOrderCommand, PlaceOrderResult } from './ports/primary/IOrderService';

export class OrderService implements IOrderService {
  constructor(
    private readonly orderRepo: IOrderRepository,
    private readonly paymentGateway: IPaymentGateway,
    private readonly notifications: INotificationPort,
  ) {}

  async placeOrder(command: PlaceOrderCommand): Promise<PlaceOrderResult> {
    // 1. Build aggregate
    const order = new Order(randomUUID(), command.customerId);
    for (const item of command.items) {
      order.addItem(item);
    }

    // 2. Business rule: minimum order amount
    if (order.total < 1) {
      throw new Error('Order total must be at least $1.00');
    }

    // 3. Charge via secondary port
    const payment = await this.paymentGateway.charge(
      command.customerId,
      order.total,
      'USD',
    );

    if (payment.status === 'declined') {
      throw new Error('Payment declined');
    }

    // 4. Confirm order
    order.confirm();

    // 5. Persist via secondary port
    await this.orderRepo.save(order);

    // 6. Notify via secondary port
    await this.notifications.notifyOrderConfirmed(
      command.customerId,
      order.id,
      order.total,
    );

    return {
      orderId: order.id,
      total: order.total,
      transactionId: payment.transactionId,
    };
  }

  async getOrder(orderId: string): Promise<Order | null> {
    return this.orderRepo.findById(orderId);
  }
}
```

### Secondary Adapters (infrastructure)

```typescript
// src/adapters/secondary/StripePaymentAdapter.ts
import Stripe from 'stripe';
import { IPaymentGateway, PaymentResult } from '../../core/ports/secondary/IPaymentGateway';

export class StripePaymentAdapter implements IPaymentGateway {
  constructor(private readonly stripe: Stripe) {}

  async charge(customerId: string, amount: number, currency: string): Promise<PaymentResult> {
    const intent = await this.stripe.paymentIntents.create({
      amount: Math.round(amount * 100), // Stripe uses cents
      currency,
      customer: customerId,
      confirm: true,
    });
    return {
      transactionId: intent.id,
      status: intent.status === 'succeeded' ? 'approved' : 'declined',
    };
  }

  async refund(transactionId: string, amount: number): Promise<void> {
    await this.stripe.refunds.create({
      payment_intent: transactionId,
      amount: Math.round(amount * 100),
    });
  }
}
```

```typescript
// src/adapters/secondary/PrismaOrderRepository.ts
import { PrismaClient } from '@prisma/client';
import { Order } from '../../core/domain/Order';
import { IOrderRepository } from '../../core/ports/secondary/IOrderRepository';

export class PrismaOrderRepository implements IOrderRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<Order | null> {
    const row = await this.prisma.order.findUnique({
      where: { id },
      include: { items: true },
    });
    if (!row) return null;

    const order = new Order(row.id, row.customerId, row.createdAt);
    for (const item of row.items) {
      order.addItem({
        productId: item.productId,
        quantity: item.quantity,
        unitPrice: Number(item.unitPrice),
      });
    }
    return order;
  }

  async save(order: Order): Promise<void> {
    await this.prisma.order.upsert({
      where: { id: order.id },
      create: {
        id: order.id,
        customerId: order.customerId,
        status: order.status,
        items: {
          create: order.items.map(i => ({
            productId: i.productId,
            quantity: i.quantity,
            unitPrice: i.unitPrice,
          })),
        },
      },
      update: { status: order.status },
    });
  }
}
```

### Primary Adapter: Fastify Controller

```typescript
// src/adapters/primary/http/OrderController.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { IOrderService } from '../../../core/ports/primary/IOrderService';

export class OrderController {
  constructor(private readonly orderService: IOrderService) {}

  async placeOrder(req: FastifyRequest, reply: FastifyReply): Promise<void> {
    const body = req.body as { customerId: string; items: unknown[] };
    try {
      const result = await this.orderService.placeOrder({
        customerId: body.customerId,
        items: body.items as any,
      });
      reply.status(201).send(result);
    } catch (err) {
      if (err instanceof Error) {
        reply.status(400).send({ error: err.message });
      }
    }
  }
}
```

### Testing: Swap Adapters for Fakes — No Mocking Framework Needed

```typescript
// tests/OrderService.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { Order } from '../src/core/domain/Order';
import { OrderService } from '../src/core/OrderService';
import { IOrderRepository } from '../src/core/ports/secondary/IOrderRepository';
import { IPaymentGateway, PaymentResult } from '../src/core/ports/secondary/IPaymentGateway';
import { INotificationPort } from '../src/core/ports/secondary/INotificationPort';

// In-memory fakes — no mocking framework
class FakeOrderRepository implements IOrderRepository {
  private store = new Map<string, Order>();
  async findById(id: string) { return this.store.get(id) ?? null; }
  async save(order: Order) { this.store.set(order.id, order); }
}

class FakePaymentGateway implements IPaymentGateway {
  shouldDecline = false;

  async charge(): Promise<PaymentResult> {
    if (this.shouldDecline) return { transactionId: '', status: 'declined' };
    return { transactionId: 'txn_test_123', status: 'approved' };
  }

  async refund(): Promise<void> {}
}

class FakeNotifications implements INotificationPort {
  sent: Array<{ customerId: string; orderId: string; total: number }> = [];

  async notifyOrderConfirmed(customerId: string, orderId: string, total: number): Promise<void> {
    this.sent.push({ customerId, orderId, total });
  }
}

describe('OrderService', () => {
  let service: OrderService;
  let payment: FakePaymentGateway;
  let notifications: FakeNotifications;

  beforeEach(() => {
    payment = new FakePaymentGateway();
    notifications = new FakeNotifications();
    service = new OrderService(new FakeOrderRepository(), payment, notifications);
  });

  it('places order and sends notification', async () => {
    const result = await service.placeOrder({
      customerId: 'cust_1',
      items: [{ productId: 'p1', quantity: 2, unitPrice: 25 }],
    });

    expect(result.total).toBe(50);
    expect(result.transactionId).toBe('txn_test_123');
    expect(notifications.sent).toHaveLength(1);
    expect(notifications.sent[0].total).toBe(50);
  });

  it('throws when payment is declined', async () => {
    payment.shouldDecline = true;

    await expect(
      service.placeOrder({
        customerId: 'cust_1',
        items: [{ productId: 'p1', quantity: 1, unitPrice: 10 }],
      })
    ).rejects.toThrow('Payment declined');

    expect(notifications.sent).toHaveLength(0);
  });
});
```

> 💡 These tests run in under 50ms with zero network calls. The fakes are dead simple to write because ports have lean interfaces (ISP applied at the architectural level).

---

## 5. Common Mistakes & Pitfalls

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Defining ports in the infrastructure layer | Inverts the dependency — core depends on infra | Always define ports inside the core |
| Making ports too wide (all CRUD methods) | Forces fakes to implement irrelevant methods | One interface per capability cluster |
| Skipping primary ports and calling the service directly | The service becomes coupled to the calling technology | Always define a primary port interface |
| Putting validation in adapters | Business validation is duplicated or missed | Validate in the core; adapters parse only HTTP/protocol concerns |
| Using the same DTO inside and outside the core | Boundary becomes leaky | Define separate input/output types per port |

> ⚠️ The most common mistake: developers put the port interfaces in the same package as the adapters. This makes the core depend on the infrastructure directory, defeating the whole purpose.

---

## 6. When to Use / Not Use

**Use Hexagonal Architecture when:**
- The application has multiple entry points (HTTP + CLI + event-driven)
- Infrastructure tends to change (migrating from MongoDB to PostgreSQL, swapping payment providers)
- Fast, isolated unit tests of business logic are a team priority
- The business domain is complex enough to justify explicit boundary design

**Simplify or skip when:**
- Simple CRUD API with a single entry point and stable infrastructure
- Prototype or early MVP where the domain is still being discovered
- Team is unfamiliar with the pattern — the vocabulary overhead can slow initial delivery

> 💡 You can apply Hexagonal Architecture incrementally. Start with a secondary port for just the most volatile dependency (e.g., the payment gateway) and expand as the need for testability and flexibility grows.

---

## 7. Relationship to Clean Architecture

| Aspect | Hexagonal | Clean Architecture |
|--------|-----------|-------------------|
| Core concept | Ports and Adapters | Layers with the Dependency Rule |
| Terminology | Primary/secondary ports, primary/secondary adapters | Entities, use cases, interface adapters, frameworks |
| Prescription | Less prescriptive about internal layers | More prescriptive (explicit layer names) |
| Origin | Alistair Cockburn, 2005 | Robert C. Martin, 2012 |
| Focus | External boundary design | Full layer hierarchy including entity rules |

Both say: *inner layers must not depend on outer layers.* They are complementary, not competing. Many teams use Clean Architecture's layer vocabulary with Hexagonal's port/adapter language simultaneously.

---

## 8. Interview Questions

**Q1: What is the difference between a port and an adapter?**

A: A port is an interface — a named contract in the application core that defines how external interaction happens. An adapter is a concrete class that implements a port using a specific technology (Prisma, Stripe, SES). Ports live inside the core; adapters live outside.

---

**Q2: What is the difference between a primary port and a secondary port?**

A: Primary (driving) ports are called by external actors to trigger application behavior — they represent the application's API. Secondary (driven) ports are called by the application core to reach external systems — they represent the application's dependencies. Primary adapters (HTTP, CLI) call primary ports; secondary adapters (DB, email) implement secondary ports.

---

**Q3: Why are in-memory fakes better than mocking frameworks in hexagonal architectures?**

A: Fakes implement the port interface directly and can be stateful, making tests readable and realistic without the brittleness of mock expectations. Mocks check that specific methods were called in specific ways — fakes check that the outcome is correct. Fakes are also easier to maintain because they track real state changes.

---

**Q4: How does Hexagonal Architecture make it easier to add a new delivery mechanism?**

A: Because the application core only knows its port interfaces, a new primary adapter (e.g., a CLI or a gRPC server) just implements the primary port and calls the service. The existing HTTP adapter, all tests, and all business logic are untouched.

---

**Q5: Can you have multiple primary adapters for the same primary port?**

A: Yes — this is one of the key benefits. An `IOrderService` primary port can be called by an HTTP controller, a CLI command, a scheduled job, and a test suite simultaneously. Each is a different adapter for the same port, and they share no code.

---

**Q6: How do you handle configuration and infrastructure setup without polluting the core?**

A: Infrastructure bootstrapping (Prisma client, Stripe SDK, SMTP connection) lives entirely in the adapters and composition root (`main.ts`). The core only receives instances of its port interfaces — it never reads environment variables or creates database connections.

---

## 9. Exercises

**Exercise 1: Add an email notification secondary adapter**

Implement `NodemailerNotificationAdapter implements INotificationPort` that sends a real email. The core `OrderService` should not change at all.

*Hint: The adapter reads SMTP credentials from environment variables. Test it with a `FakeNotifications` in unit tests and the real adapter in integration tests.*

---

**Exercise 2: Add a CLI primary adapter**

Write a CLI script (`cli/place-order.ts`) that reads order data from `stdin` as JSON, calls `OrderService.placeOrder()`, and prints the result. Use the same service that the HTTP adapter uses.

*Hint: Wire up the same repository and gateway adapters as in `main.ts`. The service code is identical.*

---

**Exercise 3: Test payment failure scenarios**

Using `FakePaymentGateway`, extend the test suite to cover:
- Insufficient funds (payment declined)
- Empty order (no items)
- Order with a single item at zero price

*Hint: Add a `declineReason` field to `FakePaymentGateway` to simulate different failure modes.*

---

**Exercise 4: Swap the payment gateway**

Implement a `PayPalPaymentAdapter implements IPaymentGateway` (stub implementation is fine). Wire it into the composition root by changing a single line. Verify that all existing tests still pass without modification.

*Hint: The tests use `FakePaymentGateway`, not the real one, so they are unaffected by the swap.*

---

## 10. Further Reading

- **[Ports and Adapters (Hexagonal Architecture)](https://alistair.cockburn.us/hexagonal-architecture/)** — Alistair Cockburn's original article (2005)
- **[DDD, Hexagonal, Onion, Clean, CQRS — How I put it all together](https://herbertograca.com/2017/11/16/explicit-architecture-01-ddd-hexagonal-onion-clean-cqrs-how-i-put-it-all-together/)** — herbertograca (comprehensive comparison)
- **Growing Object-Oriented Software, Guided by Tests** — Steve Freeman & Nat Pryce (fakes-over-mocks philosophy)
- **[Hexagonal Architecture in Node.js](https://www.freecodecamp.org/news/implementing-a-hexagonal-architecture/)** — freeCodeCamp
- **Clean Architecture** — Robert C. Martin (Chapter 22: The Clean Architecture)
