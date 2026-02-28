# Database Scaling

## Overview

As an application grows, the database often becomes the bottleneck. A single Postgres instance handles thousands of queries per second under normal conditions, but write-heavy workloads, analytical queries, and high concurrency can overwhelm a single node. This chapter covers the main strategies for scaling databases: read replicas, connection pooling, sharding, and vertical scaling — and when to use each one.

---

## Prerequisites

- SQL basics (queries, indexes, transactions)
- Basic understanding of network concepts
- Track 03 — Backend: databases and Prisma/ORM usage

---

## Core Concepts

### Scaling dimensions

**Vertical scaling (scale up):** Give the database more CPU, RAM, and faster storage. Simple, no application changes required, but has hard limits.

**Horizontal scaling (scale out):** Add more database nodes. Requires application changes and introduces distributed systems complexity, but has no hard ceiling.

### Read vs write scaling

Most web applications are read-heavy (80–95% reads). This means you can often scale dramatically by distributing reads across multiple nodes, even though writes still go to a single primary.

---

## Hands-On Examples

### Read replicas with Prisma

```typescript
// src/lib/db.ts
import { PrismaClient } from '@prisma/client';

// Primary — all writes, critical reads
export const primaryDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_PRIMARY_URL } },
  log: ['warn', 'error'],
});

// Read replica — bulk of reads
export const replicaDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_REPLICA_URL } },
  log: ['warn', 'error'],
});

// Convenience: most reads should use this
export const db = replicaDb;
```

```typescript
// src/services/user.service.ts
import { primaryDb, replicaDb } from '../lib/db.js';

export class UserService {
  // Writes always go to primary
  async createUser(data: CreateUserDto) {
    return primaryDb.user.create({ data });
  }

  async updateUser(id: string, data: UpdateUserDto) {
    return primaryDb.user.update({ where: { id }, data });
  }

  // Non-critical reads can use replica
  async listUsers(page: number, limit: number) {
    return replicaDb.user.findMany({
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' },
    });
  }

  // Critical reads (after a write) go to primary
  async getUserById(id: string, mustBeFresh = false) {
    const client = mustBeFresh ? primaryDb : replicaDb;
    return client.user.findUnique({ where: { id } });
  }
}
```

### Connection pooling with PgBouncer

PostgreSQL has a hard limit on connections (typically 100–500). Each connection consumes ~5–10MB of memory. At scale, connection pooling is essential.

PgBouncer sits between your app and Postgres, pooling connections:

```yaml
# docker-compose.yml
services:
  pgbouncer:
    image: edoburu/pgbouncer:latest
    environment:
      DB_USER: myapp
      DB_PASSWORD: secret
      DB_HOST: postgres
      DB_NAME: myapp
      POOL_MODE: transaction      # connection returned to pool after each transaction
      MAX_CLIENT_CONN: 1000       # max simultaneous app connections
      DEFAULT_POOL_SIZE: 20       # max actual Postgres connections per database/user
    ports:
      - "5432:5432"

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
```

```bash
# Application connects to PgBouncer, not Postgres directly
DATABASE_URL=postgresql://myapp:secret@pgbouncer:5432/myapp
```

With `transaction` pool mode, your app can have 1000 connections to PgBouncer, but only 20 actual Postgres connections are active at a time.

**Important:** In transaction mode, `SET` session variables and prepared statements do not persist between requests. Use `session` mode if you need those features (but session mode pools less aggressively).

### Indexing strategy

Indexes are the first optimization to reach for before scaling the database hardware:

```sql
-- Find slow queries (pg_stat_statements)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Find missing indexes (sequential scans on large tables)
SELECT schemaname, tablename, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
ORDER BY seq_scan DESC;

-- EXPLAIN ANALYZE to understand query execution
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = '123' AND status = 'pending'
ORDER BY created_at DESC;
```

```typescript
// In Prisma schema — composite index for common query pattern
model Order {
  id        String   @id @default(cuid())
  userId    String
  status    String
  createdAt DateTime @default(now())

  @@index([userId, status, createdAt(sort: Desc)])
  // Covers: WHERE userId = ? AND status = ? ORDER BY createdAt DESC
}
```

### Sharding concepts

Sharding splits data across multiple database instances based on a partition key:

```typescript
// Conceptual sharding by user ID
const SHARD_COUNT = 4;

function getShardIndex(userId: string): number {
  // Consistent hash of the user ID
  let hash = 0;
  for (const char of userId) {
    hash = (hash * 31 + char.charCodeAt(0)) % SHARD_COUNT;
  }
  return Math.abs(hash);
}

const shardClients = [
  new PrismaClient({ datasources: { db: { url: process.env.SHARD_0_URL } } }),
  new PrismaClient({ datasources: { db: { url: process.env.SHARD_1_URL } } }),
  new PrismaClient({ datasources: { db: { url: process.env.SHARD_2_URL } } }),
  new PrismaClient({ datasources: { db: { url: process.env.SHARD_3_URL } } }),
];

export function getShardClient(userId: string): PrismaClient {
  return shardClients[getShardIndex(userId)];
}

// Usage
const db = getShardClient(request.user.id);
const orders = await db.order.findMany({ where: { userId: request.user.id } });
```

**Sharding trade-offs:**
- Cross-shard queries (joins across shards) are expensive or impossible
- Rebalancing data when adding shards is complex
- Consider sharding only after read replicas and caching are exhausted

### CQRS — Command Query Responsibility Segregation

Separate the data models for reads and writes:

```typescript
// Write model (normalized, transactionally consistent)
export class OrderCommandService {
  async placeOrder(userId: string, items: OrderItem[]) {
    return primaryDb.$transaction(async (tx) => {
      // Check inventory
      for (const item of items) {
        const product = await tx.product.findUnique({ where: { id: item.productId } });
        if (!product || product.stock < item.quantity) {
          throw new Error(`Insufficient stock for ${item.productId}`);
        }
      }

      // Create order
      const order = await tx.order.create({
        data: { userId, status: 'pending', items: { create: items } },
      });

      // Decrement inventory
      await Promise.all(
        items.map((item) =>
          tx.product.update({
            where: { id: item.productId },
            data: { stock: { decrement: item.quantity } },
          })
        )
      );

      return order;
    });
  }
}

// Read model (denormalized, optimized for display)
export class OrderQueryService {
  async getUserOrders(userId: string) {
    // Can read from replica, or a separate read-optimized store
    return replicaDb.orderSummary.findMany({
      where: { userId },
      // orderSummary is a denormalized view or materialized view
    });
  }
}
```

---

## Common Patterns & Best Practices

- **Add read replicas before sharding** — simpler, handles 90% of scaling needs
- **Index before scaling** — a missing index is 100× cheaper to fix than adding hardware
- **Connection pooling from day one** — PgBouncer or Prisma's built-in pool prevent connection exhaustion
- **Monitor query latency percentiles** — p99 matters more than p50; slow outliers degrade user experience
- **Use EXPLAIN ANALYZE** on any query taking > 100ms in production
- **Avoid SELECT \*** — fetch only the columns you need; reduces data transfer and memory usage
- **Use materialized views** for expensive aggregations that run frequently

---

## Anti-Patterns to Avoid

- Sharding too early — it is complex and you probably don't need it yet
- Reading from replicas for financial operations — the lag can cause double-spends or incorrect balance checks
- N+1 queries — loading related records one at a time in a loop (use `include` in Prisma or DataLoader)
- Not monitoring replication lag — if lag grows, replica reads drift further from primary
- Long-running transactions that lock rows for seconds — causes connection pile-up

---

## Debugging & Troubleshooting

**"Database CPU is at 100% under normal load"**
Run `pg_stat_statements` to find the top queries by total execution time. The culprit is usually 2-3 queries hitting large tables without indexes.

**"Too many connections" error**
Add PgBouncer. Reduce `max_connections` in your Prisma client if you're spawning many instances. Each Node.js instance should use 5-10 Postgres connections, not 100.

**"Replica is 30 seconds behind primary"**
Check write load on primary — heavy bulk writes cause replication lag. Also check network between primary and replica. Route critical reads to primary until lag recovers.

---

## Real-World Scenarios

**Scenario: Scaling a SaaS API from 10K to 1M users**

Phase 1 (0–100K users): Single Postgres instance, proper indexes, PgBouncer.
Phase 2 (100K–500K): Add 1-2 read replicas, route 80% of reads to replicas.
Phase 3 (500K–1M): Add Redis caching layer (Track 08: Caching), reduce DB read traffic by 90%.
Phase 4 (1M+): Re-evaluate: separate databases per service, consider sharding hot tables.

---

## Further Reading

- [Postgres performance tuning guide](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Use the Index, Luke — SQL indexing guide](https://use-the-index-luke.com/)
- [PgBouncer documentation](https://www.pgbouncer.org/config.html)
- Track 08: [Caching Strategies](caching-strategies.md)
- Track 08: [CAP Theorem](cap-theorem.md)

---

## Summary

Database scaling follows a predictable progression: index optimization first (free), then connection pooling (cheap), then read replicas (moderate complexity), then caching (significant impact), then sharding (expensive and complex). Most applications never need sharding. The highest-leverage actions are proper indexing, `EXPLAIN ANALYZE` on slow queries, and adding a read replica for bulk read traffic. Connection pooling with PgBouncer or a similar tool is essential at any non-trivial scale.
