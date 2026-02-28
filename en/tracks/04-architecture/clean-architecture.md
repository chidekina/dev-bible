# Clean Architecture

## 1. What & Why

Clean Architecture, proposed by Robert C. Martin in his 2017 book of the same name, organizes code into concentric layers where inner layers know nothing about outer layers. It unifies earlier ideas — Hexagonal Architecture, Onion Architecture, and BCE (Boundary-Control-Entity) — under a single, opinionated structure.

The central promise: **your business logic survives technology changes.** Switch from Fastify to Express, from PostgreSQL to MongoDB, from REST to gRPC — the core domain code does not change. The framework is a delivery mechanism, not the foundation.

> 💡 The Dependency Rule is the single most important idea in Clean Architecture: *source code dependencies can only point inward.* Nothing in an inner layer can know anything about something in an outer layer.

Why does this matter?

- **Testability:** Use cases can be tested with zero infrastructure (no database, no HTTP server).
- **Independence from frameworks:** The business logic is not coupled to Fastify, Prisma, or any other tool.
- **Replaceability:** Swapping a database or UI is a mechanical task, not an architectural one.
- **Longevity:** The core domain can outlive multiple generations of frameworks.

---

## 2. Core Concepts

### The Four Layers

```
┌──────────────────────────────────────────────────────┐
│  Frameworks & Drivers (outermost)                    │
│  ┌────────────────────────────────────────────────┐  │
│  │  Interface Adapters                            │  │
│  │  ┌──────────────────────────────────────────┐ │  │
│  │  │  Use Cases (Application Business Rules)  │ │  │
│  │  │  ┌──────────────────────────────────┐    │ │  │
│  │  │  │  Entities (Enterprise Rules)     │    │ │  │
│  │  │  └──────────────────────────────────┘    │ │  │
│  │  └──────────────────────────────────────────┘ │  │
│  └────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────┘
```

| Layer | What lives here | Examples |
|-------|----------------|---------|
| **Entities** | Enterprise-wide business rules, domain objects | `User`, `Order`, `Money`, domain events |
| **Use Cases** | Application-specific business rules, orchestration | `CreateUserUseCase`, `PlaceOrderUseCase` |
| **Interface Adapters** | Convert data between use cases and external formats | Controllers, presenters, repository implementations |
| **Frameworks & Drivers** | All external tools and frameworks | Fastify, Prisma, Redis, S3 SDK |

### The Dependency Rule

Dependencies flow inward only:

```
Frameworks → Interface Adapters → Use Cases → Entities
```

Never the reverse. A `User` entity never imports from Fastify. A `CreateUserUseCase` never imports from Prisma. This is enforced through interfaces defined in inner layers and implemented in outer layers (the Dependency Inversion Principle at architectural scale).

### Crossing Boundaries

Data crossing a boundary is always converted into a simple data structure (plain object or DTO). No ORM entities, no framework request objects, no library types cross a layer boundary inward.

---

## 3. How It Works

The typical request flow through a Clean Architecture application:

```
HTTP Request
    │
    ▼
[Fastify Route Handler]   ← Frameworks & Drivers
    │  (parses request, creates InputDto)
    ▼
[Controller]              ← Interface Adapters
    │  (calls use case with InputDto)
    ▼
[CreateUserUseCase]       ← Use Cases
    │  (calls IUserRepository interface)
    ▼
[IUserRepository]         ← Use Case boundary (interface defined here)
    │
    ▼
[PrismaUserRepository]    ← Interface Adapters (implements the interface)
    │
    ▼
[Prisma / PostgreSQL]     ← Frameworks & Drivers
```

The use case never knows about Prisma. The entity never knows about the use case request structure. Each layer communicates through well-defined input/output data structures.

---

## 4. Code Examples (TypeScript)

### Project Structure

```
src/
├── domain/                    # Entities (innermost)
│   ├── entities/
│   │   └── User.ts
│   └── value-objects/
│       └── Email.ts
│
├── application/               # Use Cases
│   ├── use-cases/
│   │   └── create-user/
│   │       ├── CreateUserUseCase.ts
│   │       ├── CreateUserInput.ts
│   │       └── CreateUserOutput.ts
│   └── repositories/          # Interfaces (defined here, implemented in infra)
│       └── IUserRepository.ts
│
├── infrastructure/            # Interface Adapters + Frameworks
│   ├── repositories/
│   │   └── PrismaUserRepository.ts
│   ├── http/
│   │   └── controllers/
│   │       └── UserController.ts
│   └── database/
│       └── prisma-client.ts
│
└── main.ts                    # Composition root
```

### Layer 1: Entity

```typescript
// src/domain/value-objects/Email.ts
export class Email {
  private readonly value: string;

  private constructor(value: string) {
    this.value = value;
  }

  static create(raw: string): Email {
    const normalized = raw.trim().toLowerCase();
    if (!normalized.includes('@')) {
      throw new Error(`Invalid email: ${raw}`);
    }
    return new Email(normalized);
  }

  toString(): string { return this.value; }

  equals(other: Email): boolean { return this.value === other.value; }
}
```

```typescript
// src/domain/entities/User.ts
import { Email } from '../value-objects/Email';

interface UserProps {
  id: string;
  name: string;
  email: Email;
  createdAt: Date;
  isActive: boolean;
}

export class User {
  private readonly props: UserProps;

  private constructor(props: UserProps) {
    this.props = props;
  }

  static create(props: Omit<UserProps, 'createdAt' | 'isActive'>): User {
    return new User({ ...props, createdAt: new Date(), isActive: true });
  }

  static reconstitute(props: UserProps): User {
    return new User(props);
  }

  get id(): string { return this.props.id; }
  get name(): string { return this.props.name; }
  get email(): Email { return this.props.email; }
  get createdAt(): Date { return this.props.createdAt; }
  get isActive(): boolean { return this.props.isActive; }

  deactivate(): void {
    if (!this.props.isActive) throw new Error('User already inactive');
    (this.props as UserProps).isActive = false;
  }
}
```

### Layer 2: Repository Interface and Use Case

```typescript
// src/application/repositories/IUserRepository.ts
import { User } from '../../domain/entities/User';

export interface IUserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  save(user: User): Promise<void>;
}
```

```typescript
// src/application/use-cases/create-user/CreateUserUseCase.ts
import { randomUUID } from 'crypto';
import { User } from '../../../domain/entities/User';
import { Email } from '../../../domain/value-objects/Email';
import { IUserRepository } from '../../repositories/IUserRepository';

export interface CreateUserInput {
  name: string;
  email: string;
}

export interface CreateUserOutput {
  id: string;
  name: string;
  email: string;
  createdAt: string;
}

export class CreateUserUseCase {
  constructor(private readonly userRepo: IUserRepository) {}

  async execute(input: CreateUserInput): Promise<CreateUserOutput> {
    // 1. Validate domain rules
    const email = Email.create(input.email);

    // 2. Business rule: email must be unique
    const existing = await this.userRepo.findByEmail(email.toString());
    if (existing) throw new Error('Email already in use');

    // 3. Create entity
    const user = User.create({ id: randomUUID(), name: input.name.trim(), email });

    // 4. Persist via abstraction
    await this.userRepo.save(user);

    // 5. Return plain DTO — no entity leaks out
    return {
      id: user.id,
      name: user.name,
      email: user.email.toString(),
      createdAt: user.createdAt.toISOString(),
    };
  }
}
```

> 💡 Notice: the use case has zero imports from Fastify, Prisma, or any infrastructure library. It is pure orchestration of domain logic and repository calls.

### Layer 3: Repository Implementation

```typescript
// src/infrastructure/repositories/PrismaUserRepository.ts
import { PrismaClient } from '@prisma/client';
import { User } from '../../domain/entities/User';
import { Email } from '../../domain/value-objects/Email';
import { IUserRepository } from '../../application/repositories/IUserRepository';

export class PrismaUserRepository implements IUserRepository {
  constructor(private readonly prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.prisma.user.findUnique({ where: { id } });
    if (!row) return null;
    return User.reconstitute({
      id: row.id,
      name: row.name,
      email: Email.create(row.email),
      createdAt: row.createdAt,
      isActive: row.isActive,
    });
  }

  async findByEmail(email: string): Promise<User | null> {
    const row = await this.prisma.user.findUnique({ where: { email } });
    if (!row) return null;
    return User.reconstitute({
      id: row.id,
      name: row.name,
      email: Email.create(row.email),
      createdAt: row.createdAt,
      isActive: row.isActive,
    });
  }

  async save(user: User): Promise<void> {
    await this.prisma.user.upsert({
      where: { id: user.id },
      create: {
        id: user.id,
        name: user.name,
        email: user.email.toString(),
        createdAt: user.createdAt,
        isActive: user.isActive,
      },
      update: {
        name: user.name,
        email: user.email.toString(),
        isActive: user.isActive,
      },
    });
  }
}
```

### Layer 3: Controller

```typescript
// src/infrastructure/http/controllers/UserController.ts
import { FastifyRequest, FastifyReply } from 'fastify';
import { CreateUserUseCase } from '../../../application/use-cases/create-user/CreateUserUseCase';

interface CreateUserBody { name: string; email: string; }

export class UserController {
  constructor(private readonly createUser: CreateUserUseCase) {}

  async create(req: FastifyRequest<{ Body: CreateUserBody }>, reply: FastifyReply): Promise<void> {
    try {
      const output = await this.createUser.execute(req.body);
      reply.status(201).send(output);
    } catch (err) {
      if (err instanceof Error && err.message === 'Email already in use') {
        reply.status(409).send({ error: err.message });
        return;
      }
      reply.status(500).send({ error: 'Internal server error' });
    }
  }
}
```

### Composition Root

```typescript
// src/main.ts
import Fastify from 'fastify';
import { PrismaClient } from '@prisma/client';
import { PrismaUserRepository } from './infrastructure/repositories/PrismaUserRepository';
import { CreateUserUseCase } from './application/use-cases/create-user/CreateUserUseCase';
import { UserController } from './infrastructure/http/controllers/UserController';

const prisma = new PrismaClient();
const userRepo = new PrismaUserRepository(prisma);
const createUserUseCase = new CreateUserUseCase(userRepo);
const userController = new UserController(createUserUseCase);

const app = Fastify();
app.post('/users', (req, reply) => userController.create(req, reply));
app.listen({ port: 3000 });
```

### Testing Without Infrastructure

```typescript
// tests/create-user.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { CreateUserUseCase } from '../src/application/use-cases/create-user/CreateUserUseCase';
import { IUserRepository } from '../src/application/repositories/IUserRepository';
import { User } from '../src/domain/entities/User';

class InMemoryUserRepository implements IUserRepository {
  private store = new Map<string, User>();

  async findById(id: string): Promise<User | null> { return this.store.get(id) ?? null; }

  async findByEmail(email: string): Promise<User | null> {
    return [...this.store.values()].find(u => u.email.toString() === email) ?? null;
  }

  async save(user: User): Promise<void> { this.store.set(user.id, user); }
}

describe('CreateUserUseCase', () => {
  let useCase: CreateUserUseCase;

  beforeEach(() => {
    useCase = new CreateUserUseCase(new InMemoryUserRepository());
  });

  it('creates a user and returns output DTO', async () => {
    const output = await useCase.execute({ name: 'Alice', email: 'alice@example.com' });
    expect(output.name).toBe('Alice');
    expect(output.email).toBe('alice@example.com');
    expect(output.id).toBeDefined();
  });

  it('rejects duplicate emails', async () => {
    await useCase.execute({ name: 'Alice', email: 'alice@example.com' });
    await expect(
      useCase.execute({ name: 'Alice Clone', email: 'alice@example.com' })
    ).rejects.toThrow('Email already in use');
  });
});
```

---

## 5. Common Mistakes & Pitfalls

| Mistake | Consequence | Fix |
|---------|-------------|-----|
| Importing Prisma types into entities | Entities become coupled to the ORM | Use plain value objects; map in the repository |
| Putting business logic in controllers | Logic is untestable without HTTP setup | Move it to the use case |
| Sharing DTOs across layers | Any layer change ripples everywhere | Define separate input/output types per use case |
| Anemic entities (just data bags) | Business logic leaks into use cases | Entities should enforce their own invariants |
| Calling use cases from other use cases | Hidden coupling and ordering dependencies | Use domain events or application services to coordinate |
| Forcing Clean Architecture on every project | Over-engineering on simple CRUD | Apply where domain complexity justifies overhead |

> ⚠️ The biggest mistake is mapping Clean Architecture layers to framework folder names. A `controllers/` folder in an Express app is not automatically "Interface Adapters." The concept is about dependency direction, not file naming.

---

## 6. When to Use / Not Use

**Use Clean Architecture when:**
- The domain is complex with real business rules (billing, inventory, compliance)
- Multiple delivery mechanisms exist or are planned (REST, CLI, gRPC, background jobs)
- The team needs to unit test business logic without running infrastructure
- The system is expected to live for 5+ years with technology changes along the way

**Skip or simplify when:**
- The project is a thin CRUD API with no real business logic
- It is a prototype or MVP where speed matters more than structure
- The team is small (1-2 devs) and the overhead of layer mapping exceeds the benefit
- The "domain" is just reading and writing data with no decisions

> 💡 A useful heuristic: if you can replace every use case with a single database query and no logic, Clean Architecture adds structure without benefit. The sweet spot is when use cases contain actual decisions.

---

## 7. Real-World Scenario

An e-commerce platform starts as a Fastify + Prisma monolith. After 18 months, the team needs to:

1. Add a CLI tool that processes refunds from a CSV file
2. Expose the same business logic via gRPC for an internal settlement service

With Clean Architecture, both additions are mechanical:

```typescript
// The use case is reused exactly — zero changes
const processRefund = new ProcessRefundUseCase(orderRepo, paymentGateway);

// CLI adapter
const csvRows = parseCsv(file);
for (const row of csvRows) {
  await processRefund.execute({ orderId: row.order_id, amount: row.amount });
}

// gRPC adapter
server.addService(RefundService, {
  processRefund: async (call, callback) => {
    const output = await processRefund.execute(call.request);
    callback(null, output);
  },
});
```

The use case was written once. CLI and gRPC are just new adapters. No business logic was duplicated.

---

## 8. Interview Questions

**Q1: What is the Dependency Rule and why is it the core of Clean Architecture?**

A: The Dependency Rule states that all source code dependencies must point inward — toward higher-level policy. Outer layers (frameworks, UI, DB) depend on inner layers (use cases, entities), never the reverse. This ensures business logic is isolated from technology details, making it independently testable and replaceable.

---

**Q2: Where do you define repository interfaces in Clean Architecture?**

A: In the Use Case (application) layer, not in the infrastructure layer. The use case defines what it needs (the interface), and infrastructure provides the implementation. This keeps the dependency pointing inward — the use case does not know about Prisma or any specific database.

---

**Q3: What is an anemic domain model and why is it a problem?**

A: An anemic domain model is when entities are just data containers with no behavior. Business logic lives in use cases or services instead. The problem is that entities no longer enforce their own invariants — invalid state becomes possible from anywhere. Entities should contain enterprise-wide business rules and enforce them.

---

**Q4: How do you handle cross-cutting concerns like logging without polluting use cases?**

A: Use the Decorator pattern or middleware applied at the interface adapter layer. A `LoggedCreateUserUseCase` wraps `CreateUserUseCase` and adds logging without modifying the use case. Transaction boundaries are managed in the repository implementation or via a Unit of Work pattern in the infrastructure layer.

---

**Q5: What data format should cross a layer boundary?**

A: Plain, serializable data structures — DTOs — not domain entities, ORM models, or framework-specific types. This prevents inner layers from knowing about outer layer types and keeps boundaries clean.

---

**Q6: Is Clean Architecture the same as Hexagonal Architecture?**

A: They share the same core idea (isolate business logic from infrastructure via abstractions), but differ in terminology. Hexagonal Architecture focuses on Ports and Adapters with a primary/secondary distinction. Clean Architecture adds a more explicit four-layer model and is more prescriptive about what goes where.

---

**Q7: How do you test the full system while still benefiting from Clean Architecture's testability?**

A: Use a testing pyramid. Unit test use cases with in-memory fakes — fast and numerous. Integration test repository implementations against a real but ephemeral database. End-to-end test the HTTP layer against a real server instance. Failures in unit tests point to business logic; failures in integration tests point to infrastructure; failures in e2e tests point to wiring or API contracts.

---

## 9. Exercises

**Exercise 1: Add a GetUserByIdUseCase**

Following the project structure shown above, implement a `GetUserByIdUseCase` that accepts `{ id: string }`, returns the user DTO or throws if not found, and has a test using `InMemoryUserRepository`.

*Hint: The repository interface already has `findById`. The use case is thin — validate existence, map to output DTO.*

---

**Exercise 2: Add a CLI delivery mechanism**

Write a simple CLI script (`cli/create-user.ts`) that reads `name` and `email` from `process.argv`, wires up `PrismaUserRepository` and `CreateUserUseCase`, calls the use case, and prints the result.

*Hint: The use case code does not change at all — only the wiring in the CLI entry point differs from `main.ts`.*

---

**Exercise 3: Refactor an anemic controller**

Given this controller that mixes concerns:

```typescript
app.post('/users', async (req, reply) => {
  const { name, email } = req.body;
  const existing = await prisma.user.findUnique({ where: { email } });
  if (existing) return reply.status(409).send({ error: 'Email taken' });
  const user = await prisma.user.create({ data: { name, email } });
  reply.status(201).send(user);
});
```

Extract a proper `CreateUserUseCase` with an `IUserRepository`. The controller should only parse the request and call the use case.

*Hint: The controller should not know what "email taken" means — that is domain knowledge.*

---

## 10. Further Reading

- **Clean Architecture: A Craftsman's Guide to Software Structure and Design** — Robert C. Martin (2017)
- **[The Clean Architecture](https://blog.cleancoder.com/uncle-bob/2012/08/13/the-clean-architecture.html)** — Uncle Bob's original blog post
- **Implementing Domain-Driven Design** — Vaughn Vernon (complementary at the entity layer)
- **[Clean Architecture with TypeScript](https://khalilstemmler.com/articles/software-design-architecture/organizing-app-logic/)** — Khalil Stemmler's series
- **Dependency Injection Principles, Practices, and Patterns** — Mark Seemann & Steven van Deursen
