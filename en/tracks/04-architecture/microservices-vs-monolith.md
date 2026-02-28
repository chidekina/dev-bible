# Microservices vs Monolith

## 1. What & Why

One of the most consequential architectural decisions a team makes is whether to build a monolith or a set of microservices. Both are valid approaches — the right choice depends on team size, domain complexity, traffic patterns, and organizational maturity.

Getting this wrong is expensive: a premature microservices architecture can cripple a team with operational complexity; a monolith that was never designed for modularity becomes a "big ball of mud" that is impossible to maintain.

> 💡 This decision is not binary. Most successful systems start as well-structured monoliths and migrate toward services as the team and domain grow — a path known as the Strangler Fig Pattern.

---

## 2. Core Concepts

### The Monolith

A monolith is a single deployable unit containing all the application's functionality.

```
┌─────────────────────────────────────────┐
│            Monolith Process             │
│                                         │
│  ┌──────────┐  ┌──────────┐  ┌───────┐ │
│  │  Orders  │  │ Payments │  │ Users │ │
│  └──────────┘  └──────────┘  └───────┘ │
│                                         │
│  ┌──────────┐  ┌──────────┐            │
│  │Inventory │  │Shipping  │            │
│  └──────────┘  └──────────┘            │
│                                         │
│         Single Database                 │
└─────────────────────────────────────────┘
```

**Two very different kinds of monolith:**

| Type | Structure | Code quality | Deployability |
|------|-----------|-------------|--------------|
| Modular Monolith | Clear module boundaries, explicit interfaces between modules | High | Easy |
| Big Ball of Mud | No boundaries, everything depends on everything | Low | Dangerous |

A modular monolith is not a failure mode — it is a deliberate, often correct choice.

#### Modular Monolith: What Good Looks Like

```typescript
// src/modules/orders/
//   orders.controller.ts
//   orders.service.ts
//   orders.repository.ts
//   orders.types.ts

// src/modules/payments/
//   payments.controller.ts
//   payments.service.ts
//   ...

// Modules communicate through explicit public APIs, not direct imports of internals
// orders.service.ts
import { PaymentsService } from '../payments/payments.service'; // only the public API

// NOT this:
import { createPaymentRecord } from '../payments/internal/ledger'; // internal detail leaked
```

Use ESLint boundary rules (e.g., `eslint-plugin-boundaries`) to enforce that modules cannot import each other's internals.

#### Big Ball of Mud: What Bad Looks Like

```typescript
// No module structure — everything in one flat directory
// Any file can import any other file
import { sendEmail } from './emailHelper';
import { deductInventory } from './inventoryUtils';
import { chargeCard } from './paymentUtils';
import { createShipment } from './shippingService';

// One function doing everything
async function placeOrder(data: any) {
  // 400 lines of mixed concerns
}
```

---

### Microservices

Microservices decompose the system into independently deployable services, each owning its own data and communicating over the network.

```
┌─────────────┐    ┌─────────────┐    ┌─────────────┐
│  Orders     │    │  Payments   │    │   Users     │
│  Service    │    │  Service    │    │  Service    │
│  :3001      │    │  :3002      │    │  :3003      │
│  Postgres   │    │  Postgres   │    │  Postgres   │
└──────┬──────┘    └──────┬──────┘    └──────┬──────┘
       │                  │                  │
       └──────────────────┼──────────────────┘
                          │
                   Message Bus / API Gateway
```

Key properties:
- Each service owns its own database (no shared DB)
- Services communicate via HTTP/gRPC (synchronous) or events/queues (asynchronous)
- Each service can be deployed, scaled, and restarted independently
- Each service can use the language/framework best suited for its job

---

## 3. Pros and Cons

### Monolith

| Pros | Cons |
|------|------|
| Simple to develop locally | Can become hard to maintain if not modular |
| Easy to test (single process) | Cannot scale individual components independently |
| No network latency between modules | A bug in any module can crash the entire service |
| Easy to refactor across module boundaries | Deployment of one feature requires full redeployment |
| Low operational overhead | Technology lock-in (all modules use same stack) |

### Microservices

| Pros | Cons |
|------|------|
| Independent deployment per service | Distributed systems complexity (network, latency, partial failure) |
| Independent scaling per service | Data consistency is hard (no ACID across services) |
| Technology flexibility per service | Harder to test end-to-end |
| Fault isolation (one service fails, others continue) | Higher operational overhead (logging, tracing, service discovery) |
| Aligns with team ownership boundaries | Over-decomposition can create a "micro-monolith" |

---

## 4. Service Boundaries

### Align with Bounded Contexts

The best boundaries for microservices are the same as bounded contexts in DDD. Each service:
- Has its own ubiquitous language
- Owns its own data model
- Exposes a well-defined API
- Is owned by one team (Conway's Law)

```
Bounded Context → Microservice (when complexity justifies it)

Sales BC        → Order Service
Billing BC      → Payment Service
Fulfillment BC  → Shipping Service
Identity BC     → User Service
```

### Conway's Law

> Organizations which design systems... are constrained to produce designs which are copies of the communication structures of those organizations. — Melvin Conway (1968)

If your organization has 3 teams, your system will naturally evolve toward 3 subsystems. Use Conway's Law as a tool: design team structures and service boundaries together. A service owned by two teams will become a coordination bottleneck.

> 💡 Inverse Conway's Maneuver: deliberately design your team structure to match the desired architecture, rather than letting the architecture mirror the existing org chart.

---

## 5. Inter-Service Communication

### Synchronous: REST and gRPC

```typescript
// Order Service calls Payment Service via HTTP
class OrderService {
  constructor(private readonly paymentClient: PaymentServiceClient) {}

  async placeOrder(order: Order): Promise<PlacedOrder> {
    // Synchronous call — Order Service waits for response
    const paymentResult = await this.paymentClient.charge({
      customerId: order.customerId,
      amount: order.total,
      currency: order.currency,
    });

    if (paymentResult.status === 'declined') {
      throw new OrderPaymentDeclinedError(paymentResult.errorCode);
    }

    return this.finalizeOrder(order, paymentResult.transactionId);
  }
}
```

**Trade-offs of synchronous communication:**
- Simple to reason about (request/response)
- Coupling: if Payment Service is down, Order Service fails too
- Latency compounds across service hops
- Good for: queries, user-facing operations that need immediate confirmation

### Asynchronous: Events and Message Queues

```typescript
// Order Service publishes an event; Payment Service consumes it
// No direct dependency between services

// In Order Service
class OrderService {
  constructor(private readonly eventBus: IEventBus) {}

  async createOrder(input: CreateOrderInput): Promise<string> {
    const order = Order.create(input);
    await this.orderRepo.save(order);

    // Fire and continue — no waiting
    await this.eventBus.publish('order.placed', {
      orderId: order.id,
      customerId: order.customerId,
      total: order.total,
      currency: order.currency,
      items: order.items,
    });

    return order.id;
  }
}

// In Payment Service (separate process, separate deployment)
eventBus.subscribe('order.placed', async (event) => {
  const result = await paymentGateway.charge(event.customerId, event.total, event.currency);
  if (result.status === 'approved') {
    await eventBus.publish('payment.completed', { orderId: event.orderId, ...result });
  } else {
    await eventBus.publish('payment.failed', { orderId: event.orderId, reason: result.errorCode });
  }
});
```

**Trade-offs of asynchronous communication:**
- Decoupled: services don't know about each other
- Higher resilience: producer continues even if consumer is down
- Eventual consistency: no immediate confirmation
- Harder to debug: events scattered across logs
- Good for: notifications, background processing, integration between bounded contexts

---

## 6. Data Management: Saga Pattern

When a business operation spans multiple services, you cannot use a single database transaction. The Saga pattern coordinates multi-step, cross-service workflows.

### Choreography-Based Saga

No central coordinator — each service reacts to events and publishes its own.

```
Order Service    →  order.placed    →  Payment Service
Payment Service  →  payment.done    →  Inventory Service
Inventory Service→  stock.reserved  →  Shipping Service
Shipping Service →  shipment.created→  Order Service (mark complete)

Compensation (on failure):
Payment Service  →  payment.failed  →  Order Service  (cancel order)
```

```typescript
// Each service is autonomous — no shared state
// Payment Service
eventBus.subscribe('order.placed', async (event) => {
  try {
    const result = await charge(event);
    await eventBus.publish('payment.completed', { orderId: event.orderId });
  } catch {
    await eventBus.publish('payment.failed', { orderId: event.orderId });
  }
});

// Order Service
eventBus.subscribe('payment.failed', async (event) => {
  await orderService.cancel(event.orderId, 'Payment failed');
});
```

### Orchestration-Based Saga

A central orchestrator (Saga coordinator) drives the workflow explicitly.

```typescript
class PlaceOrderSaga {
  async run(orderId: string): Promise<void> {
    try {
      // Step 1
      const payment = await paymentService.charge(orderId);

      // Step 2
      const reservation = await inventoryService.reserve(orderId);

      // Step 3
      await shippingService.schedule(orderId);

      await orderService.markCompleted(orderId);
    } catch (error) {
      // Compensate in reverse order
      await this.compensate(orderId, error);
    }
  }

  private async compensate(orderId: string, error: unknown): Promise<void> {
    await inventoryService.releaseReservation(orderId).catch(() => {});
    await paymentService.refund(orderId).catch(() => {});
    await orderService.markFailed(orderId, String(error));
  }
}
```

| Aspect | Choreography | Orchestration |
|--------|-------------|--------------|
| Coupling | Low — services don't know each other | Higher — orchestrator knows all steps |
| Visibility | Hard to see the full flow | Easy — orchestrator defines it |
| Failure handling | Distributed compensation events | Centralized rollback logic |
| Best for | Simple flows with few steps | Complex, conditional multi-step workflows |

---

## 7. Strangler Fig Pattern

The Strangler Fig Pattern is the recommended way to migrate from a monolith to microservices incrementally, without a risky big-bang rewrite.

```
Phase 1: Monolith serves all traffic
┌─────────────────────────┐
│   Monolith              │ ← all requests
│   (Orders, Payments,    │
│   Users, Shipping)      │
└─────────────────────────┘

Phase 2: Extract the first service; route some traffic
       ┌────────────────┐
       │  API Gateway   │
       └───────┬────────┘
               │
    ┌──────────┴──────────┐
    │                     │
┌───▼────┐          ┌─────▼───────────┐
│ User   │          │   Monolith      │
│Service │          │   (Orders,      │
│(new)   │          │   Payments,     │
└────────┘          │   Shipping)     │
                    └─────────────────┘

Phase N: Monolith is replaced
┌────────────────────────────────────────────┐
│                 API Gateway                │
└───┬───────────┬───────────┬───────────┬───┘
    │           │           │           │
┌───▼───┐ ┌────▼───┐ ┌─────▼──┐ ┌──────▼──┐
│Orders │ │Payments│ │ Users  │ │Shipping │
└───────┘ └────────┘ └────────┘ └─────────┘
```

Key principles of Strangler Fig:
1. Never rewrite from scratch — migrate incrementally
2. Use an API Gateway or reverse proxy to route traffic
3. Extract one service at a time, starting with the most valuable or most independent piece
4. Run old and new in parallel until confidence is high; then remove the old code
5. Data migration is the hardest part — consider dual-writes and backfills

---

## 8. When to Choose Each

### Choose a Monolith when:
- Team is small (under ~8 engineers on the same product)
- The domain is not yet fully understood (avoid premature decomposition)
- Low traffic — independent scaling is not needed
- Fast development speed is critical (MVP, startup)
- Operations are minimal — you cannot afford Kubernetes, service meshes, distributed tracing

### Choose Microservices when:
- Multiple large teams own distinct parts of the system
- Services have very different scaling requirements (e.g., the video transcoding service needs 10x the CPU of the user auth service)
- Strong fault isolation is required (payments must not affect catalog)
- Different services need different technology stacks
- Regulatory separation is required (PCI-DSS, HIPAA — isolate sensitive data processing)

### Common Mistakes with Microservices

1. **Too fine-grained services:** A "User Preferences" microservice with one endpoint serving one team is overhead with no benefit. Services should be large enough to justify the operational cost.

2. **Shared database:** If two services share a database, they are not truly independent. Schema changes in the shared DB require coordination between teams — the main benefit of microservices (independent deployment) is lost.

3. **No distributed tracing:** Without trace IDs propagated across services, debugging a failed request that touched 5 services is a multi-hour debugging session. Instrument with OpenTelemetry from day one.

4. **Synchronous calls for everything:** Chains of synchronous HTTP calls create cascading failures. If Service A calls B calls C and C is slow, all three are blocked.

5. **No contract testing:** Without contract tests, API changes in Service A break Service B in production, discovered at deployment. Use Pact or schema registries.

> ⚠️ The distributed monolith is the worst outcome: all the operational complexity of microservices (network calls, independent deployments) with none of the independence (tightly coupled data, shared database, must deploy together). It happens when teams split services along technical layers (frontend service, backend service, database service) instead of business capabilities.

---

## 9. Real-World Scenario

An e-commerce startup (3 developers, 1 product) launches as a Fastify monolith with Postgres. It is a modular monolith from day one:

```
src/
  modules/
    orders/
    payments/
    users/
    inventory/
```

After 18 months: 50,000 daily users, 2 teams (product + checkout). The checkout team moves at a different pace from the product team. Black Friday requires scaling the order processing independently.

**Migration plan (Strangler Fig):**
1. Add an API Gateway (Nginx/Kong) in front of the monolith
2. Extract the `payments` module — it has the most regulatory pressure and highest value
3. Migrate payments database (dual-write for 4 weeks, then cut over)
4. Route `/api/payments/*` to the new service
5. Over 6 months, extract `orders` and `inventory`

The `users` module stays in the monolith — low traffic, high change rate, no scaling need.

---

## 10. Interview Questions

**Q1: When is a monolith the right choice, even for a large company?**

A: When a bounded context is small, stable, and owned by one team with no independent scaling needs. A modular monolith for a well-defined business capability is almost always simpler to develop and operate than a microservice. The size of the company does not dictate the architecture — the size of the team and the variability of the business domain do.

---

**Q2: What is Conway's Law and how does it affect architecture decisions?**

A: Conway's Law states that an organization's system architecture tends to mirror its communication structure. A company with 3 siloed teams will produce 3 tightly coupled subsystems. The practical implication: design your team structure and service boundaries together. Services owned by two teams become coordination bottlenecks. Services owned by one team with a clear charter can move fast.

---

**Q3: What is the Strangler Fig Pattern and why is it preferred over a full rewrite?**

A: The Strangler Fig Pattern is an incremental migration strategy: extract pieces of the monolith into new services one at a time, routing traffic gradually from the monolith to the new service. It is preferred because it eliminates the risk of a "big bang" rewrite — you never have a point where the system is non-functional. You can validate each extracted service before removing the old code.

---

**Q4: Explain the difference between choreography and orchestration in the Saga pattern.**

A: In choreography, each service publishes events and reacts to events from other services — no central coordinator. It is loosely coupled but hard to understand as a whole flow. In orchestration, a central saga coordinator calls each service in sequence and handles compensating transactions on failure. Orchestration is easier to understand and debug but introduces a coordination coupling point.

---

**Q5: Why is a shared database between microservices considered an anti-pattern?**

A: It defeats the primary benefit of microservices — independent deployability. If two services share a database, a schema change (adding a column, renaming a table) must be coordinated between both services and deployed simultaneously. This creates deployment coupling, meaning you lose the ability to deploy Service A without coordinating with Service B. Data ownership is a first-class architectural concern.

---

**Q6: What is a distributed monolith, and how do you recognize one?**

A: A distributed monolith has the operational complexity of microservices (separate processes, network calls, independent deployments) but none of the independence (shared database, tightly coupled APIs that must be deployed together, no fault isolation). Signs: you must deploy all services simultaneously for any change, services share a database, a failure in one service takes down all others.

---

**Q7: How does distributed tracing work, and why is it essential in microservices?**

A: Distributed tracing propagates a unique trace ID across all services that handle a single request. Each service adds a span (a timed operation) to the trace. Tools like Jaeger, Zipkin, or OpenTelemetry aggregate these spans into a trace visualization. Without it, debugging a slow or failed request that touched 5 services requires manually correlating logs across 5 systems — a process that can take hours. With tracing, you see the entire request lifecycle in one timeline.

---

**Q8: What does "own your data" mean for microservices, and how do you handle cross-service queries?**

A: Each service should be the single source of truth for its own domain data and should never allow another service to query its database directly. Cross-service queries are handled through: (1) API calls (synchronous, real-time), (2) event-driven data replication (each service maintains a local read model of data it needs from others), or (3) dedicated read-optimized services (CQRS read side). The trade-off is eventual consistency vs. immediate consistency.

---

## 11. Exercises

**Exercise 1: Modular Monolith design**

Take a simple blog application (users, posts, comments, tags) and design a modular monolith structure. Define the module boundaries, what each module's public API looks like, and which modules are allowed to import from which others.

*Hint: Think in terms of bounded contexts. Can the Tags module exist independently? Does Comments need to know the internals of Posts, or just the post ID?*

---

**Exercise 2: Saga — order placement**

Design a choreography-based saga for the following flow: Place Order → Reserve Inventory → Charge Payment → Create Shipment. Write the events and compensation events for each failure scenario.

*Hint: What happens if inventory is reserved but payment fails? What if payment succeeds but shipment creation fails?*

---

**Exercise 3: Strangler Fig — migrate authentication**

Given a monolith with an authentication module, design the migration plan using the Strangler Fig Pattern. What are the steps? How do you handle session data that currently lives in the monolith's database? How do you route traffic?

*Hint: Think about the API Gateway routing rules, the dual-write period for session data, and the rollback plan.*

---

## 12. Further Reading

- **Building Microservices** — Sam Newman (2nd edition, 2021, the definitive book)
- **Monolith to Microservices** — Sam Newman (focused on migration patterns)
- **[Microservices](https://martinfowler.com/articles/microservices.html)** — Martin Fowler's original article (2014)
- **[Strangler Fig Application](https://martinfowler.com/bliki/StranglerFigApplication.html)** — Martin Fowler
- **[The Pattern: Saga](https://microservices.io/patterns/data/saga.html)** — microservices.io patterns catalog
- **[Conway's Law](https://en.wikipedia.org/wiki/Conway%27s_law)** — original paper and modern interpretation
- **OpenTelemetry** — [opentelemetry.io](https://opentelemetry.io/) — standard for distributed tracing
