# ORM Patterns

## 1. What & Why

An ORM (Object-Relational Mapper) bridges the gap between the object-oriented world of your application code and the relational world of your database. Instead of writing SQL strings and manually mapping result rows to objects, you work with typed models and method calls.

But ORMs are not just convenience tools — they embody design patterns with real implications for testability, maintainability, and performance. The Active Record pattern couples your domain models to database concerns. The Data Mapper pattern separates them. The Repository pattern adds a further abstraction layer. Understanding these patterns helps you make deliberate choices rather than accepting defaults.

This file covers the two main patterns (Active Record and Data Mapper), the two leading TypeScript ORMs (Prisma and Drizzle), common performance pitfalls (N+1), and the Repository pattern for building testable data access layers.

---

## 2. Core Concepts

### Active Record Pattern

In the Active Record pattern, a model object knows how to save itself to the database. The model contains both domain data and persistence logic.

```typescript
// Active Record: the model IS the database row
class User extends ActiveRecord {
  id: string;
  name: string;
  email: string;

  // Domain behavior + persistence mixed together
  async save(): Promise<void> {
    await db.query('UPDATE users SET name=$1, email=$2 WHERE id=$3', [this.name, this.email, this.id]);
  }

  static async findByEmail(email: string): Promise<User | null> {
    const row = await db.query('SELECT * FROM users WHERE email=$1', [email]);
    return row ? Object.assign(new User(), row) : null;
  }

  async posts(): Promise<Post[]> {
    return Post.findByUserId(this.id); // also queries the DB
  }
}

// Usage
const user = await User.findByEmail('alice@example.com');
user.name = 'Alice Updated';
await user.save();
```

**Pros:**
- Simple and fast to write for CRUD-heavy applications.
- Model and persistence are co-located — easy to find.
- Good for rapid prototyping.

**Cons:**
- Hard to unit test — every model method requires a real database connection.
- Business logic ends up in the wrong layer (models, not services).
- Models become "fat" — mixing concerns.
- Difficult to swap the persistence mechanism.

---

### Data Mapper Pattern

In the Data Mapper pattern, domain objects are pure data structures (POJOs) with no database knowledge. A separate "mapper" layer handles all persistence.

```typescript
// Pure domain object — knows nothing about databases
class User {
  constructor(
    public readonly id: string,
    public name: string,
    public email: string,
    public readonly createdAt: Date
  ) {}

  // Pure domain behavior — no DB queries
  rename(newName: string): void {
    if (!newName.trim()) throw new Error('Name cannot be empty');
    this.name = newName;
  }

  isAdmin(): boolean {
    return this.email.endsWith('@internal.example.com');
  }
}

// Mapper: knows about both the domain object and the database
class UserMapper {
  constructor(private db: Pool) {}

  async findById(id: string): Promise<User | null> {
    const row = await this.db.query('SELECT * FROM users WHERE id=$1', [id]);
    if (!row.rows[0]) return null;
    return this.toDomain(row.rows[0]);
  }

  async save(user: User): Promise<void> {
    await this.db.query(
      'INSERT INTO users (id, name, email, created_at) VALUES ($1,$2,$3,$4) ON CONFLICT (id) DO UPDATE SET name=$2, email=$3',
      [user.id, user.name, user.email, user.createdAt]
    );
  }

  private toDomain(row: UserRow): User {
    return new User(row.id, row.name, row.email, row.created_at);
  }
}
```

**Pros:**
- Domain objects are pure — fully unit testable without DB.
- Clean separation of concerns.
- Domain logic is isolated from infrastructure changes.
- Easier to switch ORMs or databases.

**Cons:**
- More boilerplate.
- Mapper classes can grow complex for rich domain models.
- Overkill for simple CRUD applications.

---

## 3. Prisma

Prisma is a TypeScript-first ORM that takes a schema-first approach. You define your data model in `schema.prisma`, and Prisma generates a fully typed client.

### Schema Definition

```prisma
// schema.prisma
generator client {
  provider = "prisma-client-js"
}

datasource db {
  provider = "postgresql"
  url      = env("DATABASE_URL")
}

model User {
  id        String   @id @default(uuid())
  name      String
  email     String   @unique
  role      Role     @default(USER)
  createdAt DateTime @default(now())
  updatedAt DateTime @updatedAt
  posts     Post[]
  profile   Profile?
}

enum Role {
  USER
  ADMIN
  MODERATOR
}

model Post {
  id          String    @id @default(uuid())
  title       String
  body        String
  published   Boolean   @default(false)
  author      User      @relation(fields: [authorId], references: [id])
  authorId    String
  tags        Tag[]
  createdAt   DateTime  @default(now())

  @@index([authorId, createdAt(sort: Desc)])
}

model Tag {
  id    String @id @default(uuid())
  name  String @unique
  posts Post[]
}

model Profile {
  id     String @id @default(uuid())
  bio    String?
  user   User   @relation(fields: [userId], references: [id])
  userId String @unique
}
```

### Prisma Queries

```typescript
import { PrismaClient, Prisma } from '@prisma/client';
const prisma = new PrismaClient();

// findUnique: exact match (type-safe — returns T | null)
const user = await prisma.user.findUnique({
  where: { email: 'alice@example.com' },
});

// findFirst: first matching row
const recentPost = await prisma.post.findFirst({
  where: { published: true },
  orderBy: { createdAt: 'desc' },
});

// findMany: list with filtering + sorting + pagination
const posts = await prisma.post.findMany({
  where: {
    published: true,
    author: {
      role: { in: ['USER', 'MODERATOR'] },
    },
  },
  orderBy: [{ createdAt: 'desc' }],
  take: 20,
  skip: 0,
  // Cursor pagination:
  // cursor: { id: lastSeenId },
  // skip: 1, // skip the cursor item
});

// create
const newUser = await prisma.user.create({
  data: {
    name: 'Alice',
    email: 'alice@example.com',
    posts: {
      create: [{ title: 'First Post', body: 'Hello world' }], // nested create
    },
  },
  include: { posts: true }, // return with posts
});

// update
const updated = await prisma.user.update({
  where: { id: userId },
  data: { name: 'Alice Updated' },
});

// upsert
const upserted = await prisma.user.upsert({
  where: { email: 'alice@example.com' },
  create: { name: 'Alice', email: 'alice@example.com' },
  update: { name: 'Alice' },
});

// delete
await prisma.user.delete({ where: { id: userId } });

// deleteMany
await prisma.post.deleteMany({
  where: { authorId: userId, published: false },
});

// count
const count = await prisma.post.count({
  where: { authorId: userId },
});

// aggregate
const stats = await prisma.order.aggregate({
  where: { status: 'COMPLETED' },
  _sum: { total: true },
  _avg: { total: true },
  _count: true,
});
```

### select vs include — The N+1 Trap

```typescript
// include: eager loads related records — triggers additional queries (JOINs)
const postsWithAuthors = await prisma.post.findMany({
  include: {
    author: true,           // JOIN users
    tags: true,             // JOIN tags
    _count: {               // subquery for counts
      select: { comments: true },
    },
  },
  take: 50,
});
// Prisma executes: 1 query for posts + 1 for authors + 1 for tags
// Still much better than the N+1 you'd get with the resolver pattern without DataLoader

// select: choose specific fields — does NOT trigger extra queries for scalar fields
const postSummaries = await prisma.post.findMany({
  select: {
    id: true,
    title: true,
    createdAt: true,
    author: {
      select: { name: true, email: true }, // nested select for relation
    },
  },
});

// DANGER: The N+1 trap in a loop
const posts = await prisma.post.findMany({ take: 100 });
for (const post of posts) {
  // This is N+1! 100 extra queries
  const author = await prisma.user.findUnique({ where: { id: post.authorId } });
}

// FIX: use include or select
const posts = await prisma.post.findMany({
  take: 100,
  include: { author: { select: { name: true } } },
});
```

> ⚠️ Never call Prisma inside a loop without having already loaded the relation. Always load related data via `include` or `select` on the initial query.

### Prisma Transactions

```typescript
// $transaction: array form (parallel execution, not sequentially dependent)
const [newPost, updatedUser] = await prisma.$transaction([
  prisma.post.create({ data: { title: 'New Post', body: '...', authorId } }),
  prisma.user.update({ where: { id: authorId }, data: { postCount: { increment: 1 } } }),
]);

// $transaction: interactive (sequential, access results of previous operations)
const result = await prisma.$transaction(async (tx) => {
  const user = await tx.user.findUnique({ where: { id: userId } });
  if (!user) throw new Error('User not found');

  if (user.credits < requiredCredits) throw new Error('Insufficient credits');

  await tx.user.update({
    where: { id: userId },
    data: { credits: { decrement: requiredCredits } },
  });

  const order = await tx.order.create({
    data: { userId, amount: requiredCredits, status: 'PENDING' },
  });

  return order;
});
// If any step throws, the entire transaction rolls back

// With isolation level
await prisma.$transaction(async (tx) => { /* ... */ }, {
  isolationLevel: Prisma.TransactionIsolationLevel.Serializable,
  timeout: 5000, // ms
});
```

### Raw Queries

```typescript
// $queryRaw: use for complex queries Prisma cannot express
const result = await prisma.$queryRaw<Array<{ id: string; score: number }>>`
  SELECT u.id, COUNT(p.id)::int as score
  FROM users u
  LEFT JOIN posts p ON p.author_id = u.id
  WHERE u.created_at > ${thirtyDaysAgo}
  GROUP BY u.id
  ORDER BY score DESC
  LIMIT 10
`;

// $executeRaw: for INSERT/UPDATE/DELETE where you don't need results
const affected = await prisma.$executeRaw`
  UPDATE posts SET views = views + 1 WHERE id = ${postId}
`;
```

---

## 4. Drizzle

Drizzle is a lightweight TypeScript ORM that defines schema as TypeScript code and generates type-safe queries.

```typescript
// schema.ts
import { pgTable, uuid, text, boolean, timestamp, index } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';

export const users = pgTable('users', {
  id: uuid('id').primaryKey().defaultRandom(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  createdAt: timestamp('created_at').defaultNow().notNull(),
});

export const posts = pgTable('posts', {
  id: uuid('id').primaryKey().defaultRandom(),
  title: text('title').notNull(),
  body: text('body').notNull(),
  published: boolean('published').default(false).notNull(),
  authorId: uuid('author_id').notNull().references(() => users.id),
  createdAt: timestamp('created_at').defaultNow().notNull(),
}, (table) => ({
  authorIdx: index('posts_author_idx').on(table.authorId, table.createdAt),
}));

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));
```

```typescript
// queries.ts
import { drizzle } from 'drizzle-orm/node-postgres';
import { eq, desc, and, lt, sql } from 'drizzle-orm';
import { Pool } from 'pg';
import * as schema from './schema';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });
const db = drizzle(pool, { schema });

// Select with where and ordering
const publishedPosts = await db
  .select({
    id: schema.posts.id,
    title: schema.posts.title,
    authorName: schema.users.name,
  })
  .from(schema.posts)
  .leftJoin(schema.users, eq(schema.posts.authorId, schema.users.id))
  .where(and(
    eq(schema.posts.published, true),
    lt(schema.posts.createdAt, new Date())
  ))
  .orderBy(desc(schema.posts.createdAt))
  .limit(20);

// Insert
const [newUser] = await db
  .insert(schema.users)
  .values({ name: 'Alice', email: 'alice@example.com' })
  .returning();

// Update
await db
  .update(schema.posts)
  .set({ published: true })
  .where(eq(schema.posts.id, postId));

// Delete
await db.delete(schema.posts).where(eq(schema.posts.authorId, userId));

// Transactions
await db.transaction(async (tx) => {
  await tx.update(schema.users)
    .set({ credits: sql`credits - ${amount}` })
    .where(eq(schema.users.id, userId));

  await tx.insert(schema.orders)
    .values({ userId, amount });
});
```

> 💡 Drizzle is "SQL-like" — if you know SQL, the Drizzle query API is immediately readable. It is significantly lighter than Prisma and avoids Prisma's query engine binary. Choose Drizzle when you want fine-grained SQL control with TypeScript safety; choose Prisma when you want a higher-level API and automatic migrations.

---

## 5. The Repository Pattern

The repository pattern defines an interface for data access and provides a concrete implementation. This makes the data access layer swappable and, crucially, testable with in-memory implementations.

```typescript
// 1. Domain type (pure)
interface User {
  id: string;
  name: string;
  email: string;
  role: 'USER' | 'ADMIN';
  createdAt: Date;
}

// 2. Repository interface — the contract
interface UserRepository {
  findById(id: string): Promise<User | null>;
  findByEmail(email: string): Promise<User | null>;
  list(options: { limit: number; cursor?: string; role?: User['role'] }): Promise<User[]>;
  save(user: User): Promise<User>;
  delete(id: string): Promise<boolean>;
}

// 3. Prisma implementation
class PrismaUserRepository implements UserRepository {
  constructor(private prisma: PrismaClient) {}

  async findById(id: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { id } });
  }

  async findByEmail(email: string): Promise<User | null> {
    return this.prisma.user.findUnique({ where: { email } });
  }

  async list({ limit, cursor, role }: { limit: number; cursor?: string; role?: User['role'] }): Promise<User[]> {
    return this.prisma.user.findMany({
      where: { role: role ?? undefined },
      take: limit,
      ...(cursor && { cursor: { id: cursor }, skip: 1 }),
      orderBy: { createdAt: 'desc' },
    });
  }

  async save(user: User): Promise<User> {
    return this.prisma.user.upsert({
      where: { id: user.id },
      create: user,
      update: { name: user.name, email: user.email, role: user.role },
    });
  }

  async delete(id: string): Promise<boolean> {
    try {
      await this.prisma.user.delete({ where: { id } });
      return true;
    } catch {
      return false;
    }
  }
}

// 4. In-memory implementation for tests — fast, no DB required
class InMemoryUserRepository implements UserRepository {
  private users = new Map<string, User>();

  async findById(id: string): Promise<User | null> {
    return this.users.get(id) ?? null;
  }

  async findByEmail(email: string): Promise<User | null> {
    return Array.from(this.users.values()).find((u) => u.email === email) ?? null;
  }

  async list({ limit, cursor, role }: { limit: number; cursor?: string; role?: User['role'] }): Promise<User[]> {
    let results = Array.from(this.users.values());
    if (role) results = results.filter((u) => u.role === role);
    if (cursor) {
      const idx = results.findIndex((u) => u.id === cursor);
      results = results.slice(idx + 1);
    }
    return results.slice(0, limit);
  }

  async save(user: User): Promise<User> {
    this.users.set(user.id, { ...user });
    return user;
  }

  async delete(id: string): Promise<boolean> {
    return this.users.delete(id);
  }
}

// 5. Service uses the interface — not the concrete implementation
class UserService {
  constructor(private userRepo: UserRepository) {}

  async getUser(id: string): Promise<User> {
    const user = await this.userRepo.findById(id);
    if (!user) throw new NotFoundError(`User ${id} not found`);
    return user;
  }

  async promoteToAdmin(id: string): Promise<User> {
    const user = await this.getUser(id);
    return this.userRepo.save({ ...user, role: 'ADMIN' });
  }
}
```

**Testing without a database:**

```typescript
// tests/user-service.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { UserService } from '../src/services/user.service';
import { InMemoryUserRepository } from '../src/repositories/in-memory/user.repository';

describe('UserService', () => {
  let repo: InMemoryUserRepository;
  let service: UserService;

  beforeEach(() => {
    repo = new InMemoryUserRepository();
    service = new UserService(repo);
  });

  it('should promote a user to admin', async () => {
    const user = await repo.save({
      id: '1', name: 'Alice', email: 'alice@test.com',
      role: 'USER', createdAt: new Date(),
    });

    const promoted = await service.promoteToAdmin(user.id);

    expect(promoted.role).toBe('ADMIN');
  });

  it('should throw NotFoundError for nonexistent user', async () => {
    await expect(service.getUser('nonexistent'))
      .rejects.toThrow('User nonexistent not found');
  });
});
// These tests run in < 10ms with no DB connection
```

---

## 6. Drizzle vs Prisma: When to Choose Each

| Factor | Prisma | Drizzle |
|--------|--------|---------|
| Type safety | Excellent (generated types) | Excellent (inferred types) |
| Query API | High-level, ORM-like | SQL-like, lower-level |
| Bundle size | Large (query engine binary) | Minimal |
| Serverless/Edge | Works (Prisma Accelerate) | Native support |
| Migrations | Auto (prisma migrate) | Manual SQL + drizzle-kit |
| Raw SQL | `$queryRaw` (tagged template) | Easy (escape hatch) |
| Learning curve | Lower for JS devs | Lower for SQL devs |
| Complex queries | Sometimes awkward | Natural (SQL familiar) |
| Ecosystem maturity | More mature, larger community | Growing fast |

---

## 7. Common Mistakes & Pitfalls

**Calling Prisma in a loop (N+1):**

```typescript
// WRONG: 100 DB queries for 100 posts
const posts = await prisma.post.findMany({ take: 100 });
const enriched = await Promise.all(
  posts.map(async (p) => ({
    ...p,
    author: await prisma.user.findUnique({ where: { id: p.authorId } }),
  }))
);

// CORRECT: 1 query with include
const posts = await prisma.post.findMany({
  take: 100,
  include: { author: { select: { id: true, name: true } } },
});
```

**Not releasing pool connections in transactions:**

```typescript
// WRONG with raw pg: connection leak if error occurs before release
const client = await pool.connect();
await client.query('BEGIN');
const result = await client.query('SELECT ...');
// Error here → client never released → pool exhausted
client.release();

// CORRECT: always use try/finally
const client = await pool.connect();
try {
  await client.query('BEGIN');
  // ...
  await client.query('COMMIT');
} catch (err) {
  await client.query('ROLLBACK');
  throw err;
} finally {
  client.release(); // always runs
}
```

**Prisma client not being a singleton:**

```typescript
// WRONG: creates a new connection pool on every import in hot-reload environments
// (e.g., Next.js dev server)
export const prisma = new PrismaClient();

// CORRECT: singleton pattern
import { PrismaClient } from '@prisma/client';

const globalForPrisma = globalThis as unknown as { prisma: PrismaClient };

export const prisma =
  globalForPrisma.prisma ??
  new PrismaClient({
    log: process.env.NODE_ENV === 'development' ? ['query'] : [],
  });

if (process.env.NODE_ENV !== 'production') {
  globalForPrisma.prisma = prisma;
}
```

---

## 8. When to Use Each Approach

**Active Record:** Small scripts, rapid prototypes, simple CRUD apps without complex domain logic. When simplicity > testability.

**Data Mapper (Prisma/Drizzle as mapper):** Most production applications. The ORM generates the mapper for you. Use the Repository pattern on top for testability.

**Repository pattern:** Any application where unit testing matters. Any application that might change databases. Any application using Domain-Driven Design. Required when you want to write tests that run in < 1 second without a database.

**Raw SQL (via `$queryRaw` or `pg` directly):** Complex aggregations with window functions, recursive CTEs, database-specific features, or performance-critical queries where ORM abstraction generates suboptimal SQL.

---

## 9. Real-World Scenario

**Problem:** A startup's user service has grown to 30 tests, but they all require a live PostgreSQL connection. The test suite takes 4 minutes to run. CI fails randomly due to database connection limits being exceeded.

**Solution: Repository pattern + InMemory implementation**

```typescript
// Before: service directly imports Prisma
class UserService {
  async findActive(): Promise<User[]> {
    return prisma.user.findMany({ where: { status: 'ACTIVE' } }); // hard dependency!
  }
}

// After: service depends on interface
class UserService {
  constructor(private users: UserRepository) {}

  async findActive(): Promise<User[]> {
    return this.users.list({ status: 'ACTIVE' });
  }
}

// Tests: use in-memory repo
const service = new UserService(new InMemoryUserRepository());
// Production: use Prisma repo (wired in the composition root / DI container)
const service = new UserService(new PrismaUserRepository(prisma));
```

**Result:** 30 unit tests run in 180ms instead of 4 minutes. CI reliability goes to 100%. Integration tests (with real DB) still exist but are a separate suite run less frequently.

---

## 10. Interview Questions

**Q1: What is the difference between Active Record and Data Mapper patterns?**

Active Record: the model object handles both domain data/behavior and persistence. The model knows how to save/load itself (`user.save()`, `User.find()`). Simple but couples domain logic to DB infrastructure — hard to unit test. Data Mapper: the domain object is a pure POJO with no DB knowledge. A separate mapper class handles persistence. The domain object can be instantiated and tested without a database. Prisma follows the Data Mapper approach — `PrismaClient` is the mapper, your domain types are separate.

**Q2: What is the N+1 problem in ORMs and how do you fix it in Prisma?**

N+1 occurs when you load N records and then make 1 additional query per record to load related data. Example: load 100 posts (`1 query`), then in a loop load each post's author (`100 queries`) = 101 queries. In Prisma, fix with `include` on the initial query: `prisma.post.findMany({ include: { author: true } })` — Prisma fetches authors in a second batched query (not 100 individual queries). Alternatively use `select` to load only needed fields. Never query inside a loop.

**Q3: Why use a repository pattern on top of an ORM like Prisma?**

The repository pattern adds an abstraction interface between your service layer and the ORM. Benefits: (1) Testability — write an `InMemoryUserRepository` that the service uses in tests, no database needed, tests run in milliseconds. (2) Swappability — change from Prisma to Drizzle by swapping the concrete repository, no service changes needed. (3) Query hiding — service code is clean of query details. (4) Separation of concerns — data access logic is in one place.

**Q4: What is the difference between `select` and `include` in Prisma?**

`include` loads a relation (e.g., author of a post) — Prisma issues an additional query (or JOIN) to fetch the related records. `select` specifies which scalar fields to return and can include nested selects for relations. Both can be used to avoid over-fetching. The critical difference: `include: { author: true }` returns all fields of the related author; `select: { author: { select: { name: true } } }` returns only the author's name. For performance: always select only what you need; never `include` an entire relation if you only need one field from it.

**Q5: How do Prisma transactions work and what are the two modes?**

Prisma offers two transaction APIs: (1) **Array mode** — pass an array of Prisma operations; they run in parallel but all succeed or fail together. Limited to independent operations (cannot use the result of one as input to another). (2) **Interactive transactions** — pass an async callback that receives a `tx` client; operations run sequentially within a single transaction. Use `tx` instead of `prisma` for all operations inside the callback. If the callback throws, the entire transaction rolls back. Use the interactive form whenever operations depend on each other or need to read-then-write.

**Q6: How do you test code that uses an ORM?**

Three strategies: (1) **In-memory repository** — implement the repository interface with an in-memory Map. Tests run without DB, milliseconds per test. Best for unit tests of business logic. (2) **Test database** — spin up a real PostgreSQL instance (via Docker Compose or `@testcontainers/postgresql`) and run migrations before tests. Best for integration tests. (3) **Mock the client** — use `vitest.mock()` to mock `PrismaClient`. Fragile and couples tests to implementation details; prefer the in-memory repository instead.

**Q7: When would you choose Drizzle over Prisma?**

Choose Drizzle when: you need minimal bundle size (important for Edge/serverless deployments where Prisma's query engine binary is a problem), you prefer writing SQL-like queries rather than a high-level ORM API, you need fine-grained control over generated SQL, or you want the schema defined in TypeScript code (better for monorepo/code-sharing scenarios). Choose Prisma when: you prefer auto-generated migrations, you want the richest ecosystem and most mature tooling, or your team is more comfortable with the higher-level query API.

---

## 11. Exercises

**Exercise 1: Generic BaseRepository**

Implement a generic `BaseRepository<T, ID>` class in TypeScript that:
- Has abstract methods: `findById`, `findAll`, `save`, `delete`
- Provides a concrete in-memory implementation as `InMemoryRepository<T, ID>`
- Uses a `Map<ID, T>` as backing store
- Supports `findAll` with optional predicate filter

Then implement `PostRepository extends InMemoryRepository<Post, string>` that adds `findByAuthorId(authorId: string): Promise<Post[]>`.

*Hint:* The generic constraint: `T extends { id: ID }`. Use TypeScript generics to make the base class reusable.

**Exercise 2: N+1 detector**

Instrument Prisma with a `$use` middleware that:
- Tracks all `findUnique` calls within a single request context
- Warns (logs) if the same model is queried individually more than 5 times in a single request (indicating N+1)
- Include the stack trace of the first call in the warning

*Hint:* Use `AsyncLocalStorage` to thread request context through Prisma middleware.

**Exercise 3: Full CRUD with Repository + Validation**

Build a `PostService` that:
- Uses `PostRepository` interface (not Prisma directly)
- `createPost(authorId, data)`: validate title (min 5 chars), check author exists, create post
- `updatePost(id, userId, data)`: check post exists, check userId is author, update
- `deletePost(id, userId)`: check ownership, soft-delete (set deletedAt)
- Write unit tests using `InMemoryPostRepository` (no database required)
- Write one integration test using Prisma with a test database

---

## 12. Further Reading

- [Prisma documentation](https://www.prisma.io/docs)
- [Drizzle ORM documentation](https://orm.drizzle.team/)
- [Prisma — repository pattern guide](https://www.prisma.io/docs/guides/testing/unit-testing)
- [Martin Fowler — Active Record](https://www.martinfowler.com/eaaCatalog/activeRecord.html)
- [Martin Fowler — Data Mapper](https://www.martinfowler.com/eaaCatalog/dataMapper.html)
- [Martin Fowler — Repository](https://www.martinfowler.com/eaaCatalog/repository.html)
- Book: *Patterns of Enterprise Application Architecture* by Martin Fowler (the source for these patterns)
- Book: *Domain-Driven Design* by Eric Evans (why Data Mapper and Repository matter for rich domains)
