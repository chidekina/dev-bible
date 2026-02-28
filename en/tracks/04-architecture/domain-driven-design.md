# Domain-Driven Design

## 1. What & Why

Domain-Driven Design (DDD) is an approach to software development introduced by Eric Evans in his 2003 book *Domain-Driven Design: Tackling Complexity in the Heart of Software*. It centers the design process on the business domain — the rules, language, and concepts that make the business work — rather than on technology.

DDD is not a framework or a set of folder structures. It is a way of thinking and communicating. Its tools are divided into two groups:

- **Strategic DDD:** How to break a large system into meaningful pieces and how teams should communicate.
- **Tactical DDD:** How to model the business domain in code — the building blocks.

> 💡 The most powerful idea in DDD is not aggregates or repositories — it is *ubiquitous language*. A shared vocabulary between developers and domain experts, used consistently in conversation, code, tests, and documentation, eliminates entire categories of misunderstandings.

When does DDD pay off?

- Complex domains with non-trivial business rules
- Multiple teams working on the same large system
- Systems that need to evolve over years as the business changes

When does DDD add unnecessary complexity?

- Simple CRUD applications with no real business logic
- Reporting or analytics systems (data flows, not domain models)
- Early-stage startups where the domain is still being discovered

---

## 2. Core Concepts

### Strategic DDD

#### Ubiquitous Language

A rigorous, shared language that is used by domain experts and developers alike, in conversation, in code, and in documentation. If the domain expert says "invoice" and the developer writes `Bill`, there is a gap that will cause bugs.

```typescript
// BAD: developer-invented language
class Bill { items: LineItem[] }
class LineItem { productCode: string; qty: number; }

// GOOD: ubiquitous language from the domain
class Invoice { lineItems: InvoiceLine[] }
class InvoiceLine { sku: string; quantity: number; unitPrice: Money; }
```

#### Bounded Context

A boundary within which a particular domain model applies. The same concept (e.g., "Customer") can mean different things in different bounded contexts:

- In *Sales*: a prospect with contact details and a lead score
- In *Billing*: an account with payment methods and invoices
- In *Shipping*: a recipient with a delivery address

Each bounded context has its own model. Trying to create a single unified `Customer` entity for all contexts produces a bloated, coupled mess.

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Sales BC      │     │   Billing BC    │     │  Shipping BC    │
│                 │     │                 │     │                 │
│  Customer       │     │  BillingAccount │     │  Recipient      │
│  (prospect)     │     │  (payments)     │     │  (address)      │
└─────────────────┘     └─────────────────┘     └─────────────────┘
```

#### Context Map

A diagram showing how bounded contexts relate and integrate:

- **Shared Kernel:** Two contexts share a small subset of the domain model
- **Customer/Supplier:** One context's output feeds another's input
- **Anti-Corruption Layer (ACL):** A translation layer that protects one context from another's model
- **Conformist:** A downstream context adopts the upstream model as-is
- **Open Host Service:** A context exposes a well-defined API for others

### Tactical DDD

| Building Block | Description | Identity? |
|---------------|-------------|-----------|
| **Entity** | An object defined by identity, not attributes | Yes (stable ID) |
| **Value Object** | An object defined by its attributes; immutable | No |
| **Aggregate** | A cluster of entities and value objects with a single root | Root has identity |
| **Domain Event** | An immutable record that something happened | N/A |
| **Repository** | Abstraction for persisting and retrieving aggregates | N/A |
| **Domain Service** | Stateless logic that doesn't naturally belong to an entity | N/A |
| **Application Service** | Orchestrates use cases; thin layer above domain | N/A |

---

## 3. How It Works

A domain model in DDD is a collaboration of entities, value objects, and aggregates that enforce business invariants. The aggregate is the key building block:

- The **aggregate root** is the single entry point to the aggregate — all changes go through it.
- The aggregate root enforces **invariants** (business rules that must always hold).
- Aggregates communicate with each other only through **domain events** or **aggregate IDs** — never direct object references.
- **Repositories** load and save complete aggregates as a unit.

The flow:

```
Application Service (thin orchestration)
    │
    ├── loads aggregate via Repository
    │
    ├── calls methods on aggregate root (business logic executes here)
    │
    ├── aggregate raises Domain Events
    │
    ├── saves aggregate via Repository
    │
    └── publishes Domain Events (to event bus, handlers, other BCs)
```

---

## 4. Code Examples (TypeScript)

### Value Object: Money

```typescript
// src/domain/value-objects/Money.ts
export class Money {
  private constructor(
    private readonly _amount: number,   // in cents to avoid floating point
    private readonly _currency: string,
  ) {
    if (_amount < 0) throw new Error('Money amount cannot be negative');
    if (!_currency || _currency.length !== 3) throw new Error('Invalid currency code');
  }

  static of(amount: number, currency: string): Money {
    return new Money(Math.round(amount * 100), currency.toUpperCase());
  }

  static zero(currency: string): Money {
    return new Money(0, currency.toUpperCase());
  }

  get amount(): number { return this._amount / 100; }
  get currency(): string { return this._currency; }

  add(other: Money): Money {
    this.assertSameCurrency(other);
    return new Money(this._amount + other._amount, this._currency);
  }

  multiply(factor: number): Money {
    if (factor < 0) throw new Error('Factor must be non-negative');
    return new Money(Math.round(this._amount * factor), this._currency);
  }

  equals(other: Money): boolean {
    return this._amount === other._amount && this._currency === other._currency;
  }

  isGreaterThan(other: Money): boolean {
    this.assertSameCurrency(other);
    return this._amount > other._amount;
  }

  toString(): string {
    return `${this.currency} ${this.amount.toFixed(2)}`;
  }

  private assertSameCurrency(other: Money): void {
    if (this._currency !== other._currency) {
      throw new Error(`Currency mismatch: ${this._currency} vs ${other._currency}`);
    }
  }
}
```

### Value Object: OrderItem

```typescript
// src/domain/value-objects/OrderItem.ts
import { Money } from './Money';

export class OrderItem {
  private constructor(
    public readonly productId: string,
    public readonly productName: string,
    public readonly quantity: number,
    public readonly unitPrice: Money,
  ) {
    if (quantity <= 0) throw new Error('Quantity must be positive');
    if (!productId) throw new Error('Product ID is required');
  }

  static create(
    productId: string,
    productName: string,
    quantity: number,
    unitPrice: Money,
  ): OrderItem {
    return new OrderItem(productId, productName, quantity, unitPrice);
  }

  get subtotal(): Money {
    return this.unitPrice.multiply(this.quantity);
  }

  equals(other: OrderItem): boolean {
    return this.productId === other.productId && this.quantity === other.quantity;
  }
}
```

### Domain Event

```typescript
// src/domain/events/OrderPlaced.ts
export class OrderPlaced {
  public readonly occurredAt: Date;

  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly total: number,
    public readonly currency: string,
  ) {
    this.occurredAt = new Date();
  }
}
```

```typescript
// src/domain/events/OrderCancelled.ts
export class OrderCancelled {
  public readonly occurredAt: Date;

  constructor(
    public readonly orderId: string,
    public readonly customerId: string,
    public readonly reason: string,
  ) {
    this.occurredAt = new Date();
  }
}
```

### Entity: Order (Aggregate Root)

```typescript
// src/domain/entities/Order.ts
import { OrderItem } from '../value-objects/OrderItem';
import { Money } from '../value-objects/Money';
import { OrderPlaced } from '../events/OrderPlaced';
import { OrderCancelled } from '../events/OrderCancelled';

export type OrderStatus = 'draft' | 'placed' | 'confirmed' | 'shipped' | 'delivered' | 'cancelled';

type DomainEvent = OrderPlaced | OrderCancelled;

export class Order {
  private _items: OrderItem[] = [];
  private _status: OrderStatus = 'draft';
  private _domainEvents: DomainEvent[] = [];

  private constructor(
    private readonly _id: string,
    private readonly _customerId: string,
    private readonly _currency: string,
    private readonly _placedAt?: Date,
  ) {}

  // Factory method for new orders
  static create(id: string, customerId: string, currency: string): Order {
    if (!customerId) throw new Error('Customer ID is required');
    return new Order(id, customerId, currency);
  }

  // Factory method for reconstitution from persistence
  static reconstitute(props: {
    id: string;
    customerId: string;
    currency: string;
    status: OrderStatus;
    items: OrderItem[];
    placedAt?: Date;
  }): Order {
    const order = new Order(props.id, props.customerId, props.currency, props.placedAt);
    order._status = props.status;
    order._items = props.items;
    return order;
  }

  // --- Getters ---
  get id(): string { return this._id; }
  get customerId(): string { return this._customerId; }
  get status(): OrderStatus { return this._status; }
  get items(): ReadonlyArray<OrderItem> { return this._items; }
  get placedAt(): Date | undefined { return this._placedAt; }

  get total(): Money {
    return this._items.reduce(
      (sum, item) => sum.add(item.subtotal),
      Money.zero(this._currency),
    );
  }

  get domainEvents(): ReadonlyArray<DomainEvent> { return this._domainEvents; }

  clearEvents(): void { this._domainEvents = []; }

  // --- Business behavior (invariants enforced here) ---
  addItem(item: OrderItem): void {
    if (this._status !== 'draft') {
      throw new Error(`Cannot add items to an order in status: ${this._status}`);
    }

    const existing = this._items.find(i => i.productId === item.productId);
    if (existing) {
      // Replace the existing item (aggregate maintains consistency)
      this._items = this._items.filter(i => i.productId !== item.productId);
    }

    this._items.push(item);
  }

  removeItem(productId: string): void {
    if (this._status !== 'draft') {
      throw new Error('Cannot remove items from a non-draft order');
    }
    this._items = this._items.filter(i => i.productId !== productId);
  }

  place(): void {
    if (this._status !== 'draft') {
      throw new Error('Order has already been placed');
    }
    if (this._items.length === 0) {
      throw new Error('Cannot place an empty order');
    }
    if (this.total.isGreaterThan(Money.zero(this._currency)) === false) {
      throw new Error('Order total must be greater than zero');
    }

    this._status = 'placed';

    this._domainEvents.push(
      new OrderPlaced(this._id, this._customerId, this.total.amount, this._currency)
    );
  }

  cancel(reason: string): void {
    const cancellableStatuses: OrderStatus[] = ['draft', 'placed', 'confirmed'];
    if (!cancellableStatuses.includes(this._status)) {
      throw new Error(`Cannot cancel order in status: ${this._status}`);
    }

    this._status = 'cancelled';

    this._domainEvents.push(
      new OrderCancelled(this._id, this._customerId, reason)
    );
  }

  confirm(): void {
    if (this._status !== 'placed') {
      throw new Error('Only placed orders can be confirmed');
    }
    this._status = 'confirmed';
  }

  ship(): void {
    if (this._status !== 'confirmed') {
      throw new Error('Only confirmed orders can be shipped');
    }
    this._status = 'shipped';
  }

  deliver(): void {
    if (this._status !== 'shipped') {
      throw new Error('Only shipped orders can be delivered');
    }
    this._status = 'delivered';
  }
}
```

### Repository Interface

```typescript
// src/domain/repositories/IOrderRepository.ts
import { Order } from '../entities/Order';

export interface IOrderRepository {
  findById(id: string): Promise<Order | null>;
  findByCustomerId(customerId: string): Promise<Order[]>;
  save(order: Order): Promise<void>;
  delete(id: string): Promise<void>;
}
```

### Application Service

```typescript
// src/application/PlaceOrderService.ts
import { randomUUID } from 'crypto';
import { Order } from '../domain/entities/Order';
import { OrderItem } from '../domain/value-objects/OrderItem';
import { Money } from '../domain/value-objects/Money';
import { IOrderRepository } from '../domain/repositories/IOrderRepository';

export interface PlaceOrderInput {
  customerId: string;
  currency: string;
  items: Array<{
    productId: string;
    productName: string;
    quantity: number;
    unitPrice: number;
  }>;
}

export interface PlaceOrderOutput {
  orderId: string;
  total: string;
  status: string;
}

// Application services are thin — they orchestrate, not decide
export class PlaceOrderService {
  constructor(
    private readonly orderRepo: IOrderRepository,
    private readonly eventBus: { publish(events: unknown[]): Promise<void> },
  ) {}

  async execute(input: PlaceOrderInput): Promise<PlaceOrderOutput> {
    const order = Order.create(randomUUID(), input.customerId, input.currency);

    for (const i of input.items) {
      const item = OrderItem.create(
        i.productId,
        i.productName,
        i.quantity,
        Money.of(i.unitPrice, input.currency),
      );
      order.addItem(item);
    }

    order.place(); // domain logic + event raised

    await this.orderRepo.save(order);
    await this.eventBus.publish(order.domainEvents);
    order.clearEvents();

    return {
      orderId: order.id,
      total: order.total.toString(),
      status: order.status,
    };
  }
}
```

### Domain Service: TaxCalculator

When a business operation doesn't naturally belong to a single entity, extract a Domain Service:

```typescript
// src/domain/services/TaxCalculator.ts
import { Money } from '../value-objects/Money';
import { Order } from '../entities/Order';

// Domain Service: stateless, domain-aware logic
export class TaxCalculator {
  // Tax rules are domain knowledge, not infrastructure
  calculate(order: Order, taxRate: number): Money {
    if (taxRate < 0 || taxRate > 1) {
      throw new Error('Tax rate must be between 0 and 1');
    }
    return order.total.multiply(taxRate);
  }
}
```

### Anti-Corruption Layer

When integrating with a legacy system or external service, an ACL translates their model into your domain's language:

```typescript
// src/infrastructure/acl/LegacyOrderACL.ts
// Legacy system uses 'order_no', 'cust_id', 'prod_code', 'amt'
interface LegacyOrderRecord {
  order_no: string;
  cust_id: string;
  prod_code: string;
  qty: number;
  amt: string; // "123.45"
}

import { Order } from '../../domain/entities/Order';
import { OrderItem } from '../../domain/value-objects/OrderItem';
import { Money } from '../../domain/value-objects/Money';

export class LegacyOrderACL {
  toDomain(legacy: LegacyOrderRecord): Order {
    const order = Order.create(legacy.order_no, legacy.cust_id, 'USD');
    const item = OrderItem.create(
      legacy.prod_code,
      legacy.prod_code, // legacy has no product name
      legacy.qty,
      Money.of(parseFloat(legacy.amt), 'USD'),
    );
    order.addItem(item);
    return order;
  }

  fromDomain(order: Order): LegacyOrderRecord {
    const firstItem = order.items[0];
    return {
      order_no: order.id,
      cust_id: order.customerId,
      prod_code: firstItem?.productId ?? '',
      qty: firstItem?.quantity ?? 0,
      amt: order.total.amount.toFixed(2),
    };
  }
}
```

---

## 5. Common Mistakes & Pitfalls

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Anemic domain model | Business logic scattered in services; no invariant enforcement | Put behavior in entities and value objects |
| Sharing entities across bounded contexts | Tight coupling; the entity grows to satisfy every context | Define separate models per context |
| Aggregates that are too large | Poor concurrency; all writes are serialized through one root | Identify the true consistency boundary; split if possible |
| Aggregates that are too small | Cross-aggregate consistency issues | Ask: "what must always be consistent together?" |
| Calling repositories inside domain entities | Entities depend on infrastructure | Keep entities pure; orchestrate in application services |
| Treating DDD as a folder structure | Missing the conceptual shift | Focus on ubiquitous language and bounded contexts first |

> ⚠️ The biggest DDD mistake is skipping ubiquitous language and jumping straight to aggregates and repositories. Without shared language, even perfect tactical patterns produce code that the domain experts cannot verify.

---

## 6. When DDD Is Worth It vs. Overkill

**DDD is worth it when:**
- The domain has complex, non-obvious rules that change independently from the technology
- Multiple teams work on the same large system and need explicit boundaries
- The business can participate in design sessions (access to domain experts)
- The system will be maintained for many years with evolving requirements

**DDD is overkill when:**
- The application is a simple CRUD over a database (no real business rules)
- The domain is trivial and well-understood (e.g., a blog, a to-do list)
- The team has no access to domain experts and no shared language can be built
- The project is a short-lived throwaway or proof-of-concept

> 💡 A useful test: if you can describe every feature as "the user types data, we store it, we show it back," DDD adds complexity without value. If you catch yourself saying "but wait, an order can only be cancelled if it hasn't shipped unless it was a pre-order and the warehouse confirmed..." — DDD is for you.

---

## 7. Real-World Scenario

An e-commerce platform discovers that the `Order` domain is getting complex: different cancellation rules by status, tax calculations that vary by region, and a need to publish events to a warehouse system.

With DDD:

- The `Order` aggregate enforces all state transitions — no invalid status can be set.
- `Money` value object prevents currency mismatches and floating-point bugs.
- `OrderPlaced` domain event decouples the warehouse notification from the order service.
- An Anti-Corruption Layer translates between the modern order model and the legacy warehouse API.

The business analyst can read the domain code and verify the rules without needing to understand database schemas or HTTP handlers.

---

## 8. Interview Questions

**Q1: What is the difference between an Entity and a Value Object?**

A: An Entity is defined by identity — two entities with the same attributes but different IDs are different objects. A Value Object is defined by its attributes — two value objects with identical attributes are equal, and they are immutable. Use value objects for concepts like Money, Address, and DateRange.

---

**Q2: What is an Aggregate and why is the aggregate root important?**

A: An Aggregate is a cluster of domain objects (entities + value objects) treated as a single unit for data changes. The aggregate root is the only entry point — all external code interacts only with the root, which enforces all invariants. This ensures the aggregate is always in a valid state.

---

**Q3: What is Ubiquitous Language and why is it the most important DDD concept?**

A: Ubiquitous Language is a shared vocabulary developed by developers and domain experts together, used consistently in code, tests, documentation, and conversation. It is the most important concept because it eliminates translation errors between what the business needs and what the code does. When the code uses the same terms the business uses, misunderstandings become visible immediately.

---

**Q4: How do Bounded Contexts help manage complexity in large systems?**

A: Bounded Contexts explicitly define where a particular domain model applies. The same concept (e.g., "Customer") can have different models in different contexts. This prevents the big-ball-of-mud anti-pattern where a single `Customer` entity grows to serve 10 different contexts and becomes unmaintainable.

---

**Q5: What is an Anti-Corruption Layer and when do you need one?**

A: An ACL is a translation layer between your domain model and an external model (legacy system, third-party API, or another bounded context). You need one when the external model uses different concepts, language, or structure than your domain — the ACL prevents the external model from "corrupting" your clean domain model.

---

**Q6: What is the difference between a Domain Service and an Application Service?**

A: A Domain Service contains business logic that doesn't naturally belong to a single entity (e.g., cross-aggregate calculations, complex tax rules). It is stateless and uses domain vocabulary. An Application Service is a thin orchestration layer — it loads aggregates via repositories, calls domain logic, saves, and publishes events. Application Services have no business logic themselves.

---

**Q7: How do Domain Events enable loose coupling between bounded contexts?**

A: A Domain Event is an immutable record that something happened in the domain (e.g., `OrderPlaced`). Bounded contexts subscribe to events from other contexts without depending on their models directly. The `Shipping` context subscribes to `OrderPlaced` from the `Sales` context and creates a `Shipment` in its own model — no direct dependency.

---

**Q8: When should you split one Aggregate into two?**

A: Split when: (1) two groups of objects don't need to be consistent within the same transaction, (2) one root has so many children that concurrent writes are frequently blocked, or (3) you find yourself loading a huge graph of objects just to change one attribute. The rule of thumb: aggregates should be as small as possible while still enforcing all their invariants.

---

## 9. Exercises

**Exercise 1: Complete the Order aggregate**

The `Order` aggregate above is missing `confirm()`, `ship()`, and `deliver()` domain events. Add `OrderConfirmed`, `OrderShipped`, and `OrderDelivered` domain events and raise them from the corresponding methods.

*Hint: Follow the same pattern as `OrderPlaced` — the event carries only the data the consumer needs.*

---

**Exercise 2: Implement a discount Value Object**

Create a `Discount` value object that represents a percentage or fixed discount. Add an `applyTo(price: Money): Money` method. Ensure it validates: percentage must be 0-100, fixed must be positive.

*Hint: Use an immutable class with a private constructor and a static factory method.*

---

**Exercise 3: Model a shopping cart as a separate aggregate**

Design a `Cart` aggregate that allows adding and removing items before placing an order. A `Cart` can be converted to an `Order`. Decide: should `Cart` and `Order` share the same `OrderItem` value object, or have separate ones?

*Hint: They likely share the same product and pricing concepts but may have different validation rules.*

---

**Exercise 4: Identify bounded contexts**

For a hotel booking system, identify at least 3 bounded contexts and what "Customer" means in each. Draw a simple context map showing how they relate.

*Hint: Think about the contexts of Reservations, Billing, Housekeeping, and Loyalty.*

---

## 10. Further Reading

- **Domain-Driven Design: Tackling Complexity in the Heart of Software** — Eric Evans (2003, the original)
- **Implementing Domain-Driven Design** — Vaughn Vernon (2013, more practical and accessible)
- **Domain-Driven Design Distilled** — Vaughn Vernon (2016, concise introduction)
- **[DDD Reference](https://www.domainlanguage.com/ddd/reference/)** — Eric Evans's official summary (free PDF)
- **[Khalil Stemmler's DDD Series](https://khalilstemmler.com/articles/domain-driven-design-intro/)** — excellent TypeScript-focused articles
- **[Martin Fowler's DDD articles](https://martinfowler.com/tags/domain%20driven%20design.html)** — patterns explained clearly
