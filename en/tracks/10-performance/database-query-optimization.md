# Database Query Optimization

## Overview

The database is the most common performance bottleneck in web applications. Slow queries, missing indexes, N+1 query patterns, and inefficient joins can turn a fast API into one that times out under load. This chapter covers diagnosing slow queries with EXPLAIN ANALYZE, designing effective indexes, eliminating N+1 queries, and common Postgres optimization patterns.

---

## Prerequisites

- SQL fundamentals (SELECT, JOIN, WHERE, ORDER BY)
- Prisma or another ORM
- Track 08: Database Scaling (for broader context)

---

## Core Concepts

### How Postgres executes queries

When you run a query, Postgres's query planner evaluates multiple execution strategies and picks the one with the lowest estimated cost. The planner's choices — sequential scan vs. index scan, hash join vs. nested loop — determine performance. `EXPLAIN ANALYZE` shows you the plan it chose and the actual time it took.

### The three main performance killers

1. **Missing indexes** — sequential scan on a large table
2. **N+1 queries** — one query to get a list, then one query per item
3. **Non-selective queries** — WHERE clauses that don't narrow the result set enough

---

## Hands-On Examples

### EXPLAIN ANALYZE

```sql
-- Find slow queries
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Analyze a specific query
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.name, u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC;
```

Sample output:

```
Sort  (cost=2847.23..2848.12 rows=356 width=248) (actual time=125.432..125.891 rows=356 loops=1)
  Sort Key: o.created_at DESC
  Sort Method: quicksort  Memory: 185kB
  ->  Hash Join  (cost=...) (actual time=45.123..124.012 rows=356 loops=1)
        ...
        ->  Seq Scan on orders o  (cost=0.00..8423.50 rows=356 width=...)
              Filter: (status = 'pending' AND created_at > ...)
              Rows Removed by Filter: 129644     ← 130K rows scanned, 356 returned
Planning Time: 2.3 ms
Execution Time: 126.2 ms
```

The `Seq Scan` with `Rows Removed by Filter: 129644` indicates a missing index — Postgres scanned all 130K rows to find 356.

### Creating effective indexes

```sql
-- Single column index for equality queries
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);

-- Composite index for multi-column WHERE + ORDER BY
-- Columns in the WHERE clause first, then ORDER BY columns
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at DESC);

-- Partial index — only index rows matching a condition
-- If you only ever query pending/processing orders (not historical ones)
CREATE INDEX CONCURRENTLY idx_orders_active
  ON orders (created_at DESC)
  WHERE status IN ('pending', 'processing');

-- Index for LIKE prefix searches
CREATE INDEX CONCURRENTLY idx_users_email_prefix
  ON users (email text_pattern_ops);  -- enables LIKE 'prefix%'

-- GIN index for full-text search
ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || description)) STORED;

CREATE INDEX idx_products_search ON products USING GIN (search_vector);
```

```typescript
// Prisma schema equivalent
model Order {
  id        String   @id @default(cuid())
  status    String
  createdAt DateTime @default(now())
  userId    String

  // Composite index
  @@index([status, createdAt(sort: Desc)])
}
```

After adding the index, rerun EXPLAIN ANALYZE:

```
Index Scan using idx_orders_status_created on orders o
  (cost=0.43..12.34 rows=356 width=...) (actual time=0.05..2.34 rows=356 loops=1)
  Index Cond: (status = 'pending')
  Filter: (created_at > ...)
Execution Time: 3.1 ms   ← 40x faster
```

### Eliminating N+1 queries

The N+1 pattern: one query to fetch a list, then one additional query per item.

**N+1 in Prisma (bad):**

```typescript
// This makes 1 + N queries (1 to get orders, N to get each user)
const orders = await db.order.findMany({ where: { status: 'pending' } });

for (const order of orders) {
  // N queries inside the loop
  const user = await db.user.findUnique({ where: { id: order.userId } });
  console.log(`${user.name}: ${order.total}`);
}
```

**Fixed with `include` (eager loading):**

```typescript
// 2 queries: one for orders + one JOIN for users
const orders = await db.order.findMany({
  where: { status: 'pending' },
  include: { user: true },  // JOIN in a single query or batched query
});

for (const order of orders) {
  console.log(`${order.user.name}: ${order.total}`); // no additional query
}
```

**Detecting N+1 with Prisma logging:**

```typescript
const db = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
  ],
});

let queryCount = 0;
db.$on('query', () => { queryCount++; });

// Test a route handler
const start = Date.now();
queryCount = 0;
await getOrdersWithUsers(userId);
console.log(`Queries: ${queryCount}, Time: ${Date.now() - start}ms`);
// If queryCount > 3 for a list of 100 items, you have N+1
```

### DataLoader pattern for batching

When you cannot use `include` (e.g., in a GraphQL resolver), use DataLoader to batch individual lookups into a single query:

```typescript
import DataLoader from 'dataloader';

const userLoader = new DataLoader<string, User | null>(
  async (userIds) => {
    const users = await db.user.findMany({
      where: { id: { in: userIds as string[] } },
    });

    // DataLoader requires results in the same order as keys
    const userMap = new Map(users.map((u) => [u.id, u]));
    return userIds.map((id) => userMap.get(id) ?? null);
  },
  { cache: true } // cache within a request
);

// In resolvers — each call to load() is batched
async function resolveOrderUser(order: Order) {
  return userLoader.load(order.userId); // batched with all other calls in the same tick
}
```

### Query optimization patterns

**Select only needed columns:**

```typescript
// Bad — fetches all columns including large blobs
const users = await db.user.findMany();

// Good — select only what you display
const users = await db.user.findMany({
  select: { id: true, name: true, email: true, role: true },
  // No passwordHash, no profileJson (large JSON), no other unused columns
});
```

**Pagination — cursor-based vs offset:**

```typescript
// Offset pagination — O(n) — Postgres scans through all preceding rows
const page3 = await db.product.findMany({
  skip: 200,  // scans 200 rows just to skip them
  take: 20,
  orderBy: { createdAt: 'desc' },
});

// Cursor pagination — O(1) — uses index directly
const afterId = req.query.cursor; // ID of last item on previous page

const nextPage = await db.product.findMany({
  take: 20,
  skip: afterId ? 1 : 0,
  cursor: afterId ? { id: afterId } : undefined,
  orderBy: { createdAt: 'desc' },
});
```

**Counting efficiently:**

```typescript
// Bad — fetches all rows to count
const total = (await db.order.findMany({ where: { userId } })).length;

// Good — COUNT query
const total = await db.order.count({ where: { userId } });

// Good — fetch list and count in parallel
const [orders, total] = await Promise.all([
  db.order.findMany({ where: { userId }, take: 20, skip: offset }),
  db.order.count({ where: { userId } }),
]);
```

---

## Common Patterns & Best Practices

- **Index every foreign key** — Postgres does not create them automatically
- **Use composite indexes matching your WHERE + ORDER BY pattern**
- **Create indexes CONCURRENTLY** — does not lock the table during index creation
- **Monitor slow query log** — set `log_min_duration_statement = 100` in Postgres (log queries > 100ms)
- **Use `select` to fetch only needed columns** — especially avoid fetching large JSON or binary fields
- **Cursor-based pagination** for large datasets — offset degrades at page 1000+

---

## Anti-Patterns to Avoid

- N+1 queries — always check how many queries a request makes
- Fetching all rows to count them
- `SELECT *` — fetches unnecessary data, may include large columns
- Indexes on low-cardinality columns (status with 3 values) without a partial index — the planner may ignore them
- Skipping `EXPLAIN ANALYZE` — optimization without profiling is guesswork

---

## Debugging & Troubleshooting

**"EXPLAIN shows index scan but it's still slow"**
The index exists but the data it returns is large. Check `actual rows` — if 50K rows are returned after an index scan, you may need a more selective WHERE clause or a covering index.

**"Adding an index made things worse"**
Indexes have write overhead. On write-heavy tables, too many indexes slow INSERT/UPDATE. Remove unused indexes with `SELECT schemaname, tablename, indexname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0`.

**"Prisma is making unexpected extra queries"**
Use `log: ['query']` to see all queries. Prisma sometimes uses a separate query to count relations when you use `include` with pagination. Use raw queries or adjust the query structure.

---

## Further Reading

- [Use the Index, Luke — SQL indexing guide](https://use-the-index-luke.com/)
- [PostgreSQL EXPLAIN documentation](https://www.postgresql.org/docs/current/sql-explain.html)
- [Prisma: query optimization](https://www.prisma.io/docs/guides/performance-and-optimization/query-optimization-performance)
- [DataLoader](https://github.com/graphql/dataloader)
- Track 08: [Database Scaling](../08-system-design/database-scaling.md)

---

## Summary

Database query optimization follows a simple process: identify slow queries with `pg_stat_statements`, analyze them with `EXPLAIN ANALYZE`, add targeted indexes, and eliminate N+1 patterns. The biggest wins come from: adding composite indexes that match your WHERE + ORDER BY patterns, using `include` or DataLoader instead of per-item queries, and selecting only the columns you need. Cursor-based pagination replaces offset pagination for large datasets. Always measure before and after each change with `EXPLAIN ANALYZE` to confirm the improvement.
