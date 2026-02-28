# Track 03 — Backend

Server-side development: Node.js internals, API design, authentication, databases, caching, and messaging.

**Prerequisite:** Track 01 (JavaScript/TypeScript)

**Estimated time:** 4–5 weeks

---

## Topics

1. [Node.js Internals](nodejs-internals.md) — V8, libuv, event loop, streams, modules
2. [REST API Design](rest-api-design.md) — resources, methods, status codes, pagination, versioning
3. [GraphQL](graphql.md) — schema, resolvers, N+1 problem, DataLoader, federation
4. [Auth: JWT & OAuth2](auth-jwt-oauth.md) — sessions vs tokens, JWT pitfalls, OAuth2 flows, PKCE
5. [SQL Databases](databases-sql.md) — ACID, normalization, indexes, transactions, PostgreSQL deep dive
6. [NoSQL Databases](databases-nosql.md) — document, key-value, column, graph, CAP theorem
7. [ORM Patterns](orm-patterns.md) — Active Record vs Data Mapper, Prisma, Drizzle, repository pattern
8. [Caching & Redis](caching-redis.md) — cache strategies, Redis data types, eviction, distributed locking
9. [Message Queues](message-queues.md) — decoupling, delivery guarantees, RabbitMQ, BullMQ, Kafka

---

## Suggested Order

**1 → 2 → 5 → 7 → 4 → 8 → 9 → 3 → 6**

> Master SQL before NoSQL. 90% of projects need a relational database.

---

## Practice

→ [Backend Challenges](../../exercises/backend-challenges.md)
