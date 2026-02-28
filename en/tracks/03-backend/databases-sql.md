# Databases: SQL

## 1. What & Why

Relational databases have dominated data storage for over 40 years because they provide a powerful combination: a flexible query language (SQL), strong consistency guarantees (ACID), and a mature ecosystem. PostgreSQL in particular has grown to be the default choice for most backend applications because it combines SQL with advanced features — JSONB, window functions, full-text search, and extensibility — that remove the need for many specialized databases.

Understanding SQL deeply — not just CRUD, but indexes, execution plans, transactions, and isolation levels — separates engineers who build systems that work in production from those who produce demos that fall over under load.

---

## 2. Core Concepts

### ACID Properties

ACID defines the guarantees that a database transaction must provide:

**Atomicity** — A transaction is all-or-nothing. If any step fails, the entire transaction rolls back. No partial states are committed to the database.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- If the second UPDATE fails, the first is also rolled back
COMMIT;
```

**Consistency** — A transaction takes the database from one valid state to another. Constraints (foreign keys, check constraints, unique constraints) are enforced. A transaction that violates consistency is aborted.

**Isolation** — Concurrent transactions appear to execute serially. Each transaction works with a consistent snapshot of the data. The degree of isolation is configurable (see Isolation Levels).

**Durability** — Once committed, data survives crashes. PostgreSQL achieves this with Write-Ahead Logging (WAL): changes are written to a WAL log file before being applied to data files. On crash, the WAL is replayed on restart.

---

### Normalization

Normalization reduces data redundancy and improves integrity by organizing tables following normal forms.

**1NF (First Normal Form):**
- Each column contains atomic (indivisible) values.
- No repeating groups or arrays in columns.
- Each row is uniquely identifiable.

```sql
-- Violates 1NF: multiple values in one column
CREATE TABLE orders (
  id INT,
  customer_name TEXT,
  items TEXT  -- "apple, banana, cherry" ← NOT atomic
);

-- 1NF compliant: separate rows per item
CREATE TABLE order_items (
  order_id INT,
  item_name TEXT,
  quantity INT
);
```

**2NF (Second Normal Form):** Must be in 1NF + no partial dependencies on a composite primary key. Every non-key column must depend on the WHOLE primary key.

```sql
-- Violates 2NF: customer_name depends only on customer_id, not on (customer_id, product_id)
CREATE TABLE order_products (
  customer_id INT,
  product_id INT,
  customer_name TEXT,   -- ← partial dependency on customer_id alone
  quantity INT,
  PRIMARY KEY (customer_id, product_id)
);

-- 2NF compliant: extract customer to its own table
CREATE TABLE customers (id INT PRIMARY KEY, name TEXT);
CREATE TABLE order_products (
  customer_id INT REFERENCES customers(id),
  product_id INT,
  quantity INT,
  PRIMARY KEY (customer_id, product_id)
);
```

**3NF (Third Normal Form):** Must be in 2NF + no transitive dependencies. Non-key columns must not depend on other non-key columns.

```sql
-- Violates 3NF: zip_code determines city (transitive dependency)
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name TEXT,
  zip_code TEXT,
  city TEXT  -- ← depends on zip_code, not on id
);

-- 3NF compliant
CREATE TABLE zip_codes (zip TEXT PRIMARY KEY, city TEXT);
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name TEXT,
  zip_code TEXT REFERENCES zip_codes(zip)
);
```

**When to denormalize:**
- Read-heavy reporting queries that JOIN many tables — the JOIN overhead can exceed the redundancy cost.
- Pre-computed aggregates (e.g., `post_count` column on users table) for frequently-accessed counters.
- Analytics/OLAP workloads where reads dominate and update anomalies are acceptable.
- When you add denormalization, document it explicitly and add triggers or application-level code to maintain consistency.

---

### Indexes

An index is a separate data structure that allows the database to find rows without scanning the entire table.

**B-tree (default):**
- Balanced tree, O(log n) lookup.
- Supports: equality (`=`), range queries (`<`, `>`, `BETWEEN`), `LIKE 'prefix%'`, `ORDER BY`, `IS NULL`.
- Default for most columns.

**Hash:**
- O(1) lookup for exact equality only.
- Does NOT support range queries or sorting.
- Rarely chosen over B-tree in practice (B-tree handles equality too and supports more query types).

**GIN (Generalized Inverted Index):**
- For array containment, JSONB operators, full-text search.
- Stores a mapping from individual elements/tokens to their containing rows.

**Partial index:**
- Only indexes rows matching a WHERE condition.
- Smaller than a full index; more effective when most queries target a subset.

```sql
-- Partial index: most queries only access non-deleted records
CREATE INDEX idx_users_email_active
ON users(email)
WHERE deleted_at IS NULL;

-- Full-text index
CREATE INDEX idx_posts_search ON posts USING GIN(to_tsvector('english', title || ' ' || body));

-- JSONB index: allows fast @> containment queries
CREATE INDEX idx_events_metadata ON events USING GIN(metadata);

-- Covering index: includes extra columns to avoid going to the heap
-- Query: SELECT name, email FROM users WHERE status = 'active'
CREATE INDEX idx_users_status_covering ON users(status) INCLUDE (name, email);
-- With INCLUDE, the index contains name and email — the query can be satisfied from
-- the index alone (Index Only Scan), no heap fetch needed
```

**Composite index column order matters:**

```sql
-- Index on (status, created_at)
CREATE INDEX idx_orders ON orders(status, created_at DESC);

-- Uses index: status is the leading column
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;

-- Uses index (leading column present)
SELECT * FROM orders WHERE status = 'pending';

-- Does NOT efficiently use index: leading column missing
SELECT * FROM orders WHERE created_at > '2024-01-01';
-- Would use idx_orders(created_at) if it existed separately

-- Rule: leftmost prefix of the composite index must be in the query
```

> 💡 Index the most selective column first in a composite index for maximum filter effectiveness. But if you almost always filter by a specific column, put it first regardless — the optimizer will use the index for that column even if not the most selective.

> ⚠️ Indexes have costs: each write (INSERT/UPDATE/DELETE) must also update all relevant indexes. A table with 15 indexes will have slower writes. Profile before adding indexes speculatively.

---

### EXPLAIN ANALYZE

The most important tool for understanding query performance:

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON p.author_id = u.id
WHERE u.status = 'active'
GROUP BY u.id, u.name
ORDER BY post_count DESC
LIMIT 10;

-- Example output:
-- Limit  (cost=234.50..234.53 rows=10 width=36) (actual time=12.543..12.545 rows=10 loops=1)
--   ->  Sort  (cost=234.50..236.50 rows=800 width=36) (actual time=12.541..12.542 rows=10 loops=1)
--         Sort Key: (count(p.id)) DESC
--         Sort Method: top-N heapsort  Memory: 25kB
--         ->  HashAggregate  (cost=192.00..200.00 rows=800 width=36) (actual time=12.300..12.410 rows=812 loops=1)
--               Group Key: u.id
--               ->  Hash Left Join  (cost=45.00..176.00 rows=3200 width=20) (actual time=1.234..10.456 rows=3200 loops=1)
--                     Hash Cond: (p.author_id = u.id)
--                     ->  Seq Scan on posts p  (cost=0.00..64.00 rows=3200 width=8) (actual time=0.012..2.345 rows=3200 loops=1)
--                     ->  Hash  (cost=35.00..35.00 rows=800 width=20) (actual time=0.890..0.890 rows=812 loops=1)
--                           Buckets: 1024  Batches: 1  Memory Usage: 48kB
--                           ->  Index Scan using idx_users_status on users u  (cost=0.28..35.00 rows=800 width=20) (actual time=0.034..0.723 rows=812 loops=1)
--                                 Index Cond: ((status)::text = 'active'::text)
-- Planning Time: 0.456 ms
-- Execution Time: 12.678 ms
```

**Reading the output:**
- `cost=X..Y`: estimated startup cost .. total cost (arbitrary units).
- `actual time=X..Y`: real milliseconds startup..finish.
- `rows`: estimated (from statistics) vs actual rows.
- Large discrepancy between estimated and actual rows = stale statistics → run `ANALYZE`.
- **Seq Scan**: reads every row in the table — expected for small tables or high-selectivity queries; problematic for large tables with low-selectivity predicates.
- **Index Scan**: looks up rows via index, fetches from heap. Good.
- **Index Only Scan**: all needed columns in index — no heap fetch. Best for read performance.
- **Bitmap Index Scan**: collects row IDs from index, then fetches heap in page order. Used when many rows match — more efficient than repeated random heap accesses.

---

### Transaction Isolation Levels

| Level | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|-----------|---------------------|--------------|
| Read Uncommitted | Possible | Possible | Possible |
| Read Committed | Prevented | Possible | Possible |
| Repeatable Read | Prevented | Prevented | Possible* |
| Serializable | Prevented | Prevented | Prevented |

*PostgreSQL's MVCC-based Repeatable Read also prevents phantom reads in most cases.

**PostgreSQL default: Read Committed.**

**Phenomena:**
- **Dirty read**: reading uncommitted changes from another transaction (data that might be rolled back).
- **Non-repeatable read**: reading the same row twice within a transaction gets different values (another transaction committed an update between the two reads).
- **Phantom read**: a range query returns different sets of rows on two executions within the same transaction (another transaction inserted/deleted rows).

```sql
-- Serializable isolation: prevents all anomalies (most restrictive)
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... complex multi-read + write operations that must be atomic
COMMIT;

-- In PostgreSQL, SERIALIZABLE uses SSI (Serializable Snapshot Isolation)
-- which has much lower overhead than traditional locking
-- but can generate serialization failures (retry required)
```

---

## 3. How It Works — Common SQL Patterns

### Soft Delete

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = $1;

-- Queries exclude soft-deleted records
SELECT * FROM users WHERE deleted_at IS NULL;

-- Partial index for performance
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;

-- Restore
UPDATE users SET deleted_at = NULL WHERE id = $1;
```

### Audit Log

```sql
CREATE TABLE audit_log (
  id          BIGSERIAL PRIMARY KEY,
  table_name  TEXT NOT NULL,
  record_id   UUID NOT NULL,
  operation   TEXT NOT NULL CHECK (operation IN ('INSERT', 'UPDATE', 'DELETE')),
  old_data    JSONB,
  new_data    JSONB,
  changed_by  UUID REFERENCES users(id),
  changed_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

-- Trigger function
CREATE OR REPLACE FUNCTION audit_trigger() RETURNS TRIGGER AS $$
BEGIN
  INSERT INTO audit_log (table_name, record_id, operation, old_data, new_data, changed_by)
  VALUES (
    TG_TABLE_NAME,
    COALESCE(NEW.id, OLD.id),
    TG_OP,
    CASE WHEN TG_OP = 'INSERT' THEN NULL ELSE row_to_json(OLD)::JSONB END,
    CASE WHEN TG_OP = 'DELETE' THEN NULL ELSE row_to_json(NEW)::JSONB END,
    current_setting('app.current_user_id', TRUE)::UUID
  );
  RETURN NEW;
END;
$$ LANGUAGE plpgsql;

-- Attach to tables
CREATE TRIGGER users_audit AFTER INSERT OR UPDATE OR DELETE ON users
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### Cursor Pagination

```sql
-- First page (no cursor)
SELECT id, title, created_at
FROM posts
WHERE status = 'published'
ORDER BY created_at DESC, id DESC
LIMIT 21; -- fetch 21, if 21 returned → hasMore = true

-- Subsequent pages using cursor (created_at + id from last item)
SELECT id, title, created_at
FROM posts
WHERE status = 'published'
  AND (created_at, id) < ($last_created_at, $last_id)
ORDER BY created_at DESC, id DESC
LIMIT 21;

-- Index to support this query
CREATE INDEX idx_posts_cursor ON posts(status, created_at DESC, id DESC)
WHERE status = 'published';
```

### Optimistic Locking

```sql
ALTER TABLE documents ADD COLUMN version INT NOT NULL DEFAULT 1;

-- Read
SELECT id, content, version FROM documents WHERE id = $1;

-- Update: include version in WHERE clause
UPDATE documents
SET content = $new_content, version = version + 1
WHERE id = $1 AND version = $expected_version;

-- In application: check rows affected
-- If rowCount = 0 → version mismatch → another process modified the record → retry
```

---

## 4. Code Examples (TypeScript / PostgreSQL)

### PostgreSQL-specific Features

```typescript
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// JSONB operators
// -> returns JSON value, ->> returns text
// @> containment operator: does the left JSONB contain the right?
const result = await pool.query(`
  SELECT id, metadata->>'plan' as plan_name, metadata->'features' as features
  FROM subscriptions
  WHERE metadata @> '{"status": "active"}'
    AND metadata->>'plan' IN ('pro', 'enterprise')
`);

// UPSERT with ON CONFLICT
await pool.query(`
  INSERT INTO user_preferences (user_id, key, value)
  VALUES ($1, $2, $3)
  ON CONFLICT (user_id, key)
  DO UPDATE SET value = EXCLUDED.value, updated_at = NOW()
  RETURNING *
`, [userId, 'theme', 'dark']);

// CTE (Common Table Expression) — readable, reusable subqueries
const report = await pool.query(`
  WITH monthly_revenue AS (
    SELECT
      DATE_TRUNC('month', created_at) as month,
      SUM(amount) as total,
      COUNT(*) as order_count
    FROM orders
    WHERE status = 'paid'
    GROUP BY DATE_TRUNC('month', created_at)
  ),
  prev_month AS (
    SELECT month, total, LAG(total) OVER (ORDER BY month) as prev_total
    FROM monthly_revenue
  )
  SELECT
    month,
    total,
    ROUND(((total - prev_total) / prev_total) * 100, 2) as growth_pct
  FROM prev_month
  ORDER BY month DESC
  LIMIT 12;
`);

// Window functions
const rankings = await pool.query(`
  SELECT
    user_id,
    score,
    ROW_NUMBER() OVER (ORDER BY score DESC) as rank,
    RANK() OVER (ORDER BY score DESC) as rank_with_ties,
    LAG(score, 1) OVER (ORDER BY score DESC) as prev_score,
    SUM(score) OVER () as total_score,
    ROUND(score::NUMERIC / SUM(score) OVER () * 100, 2) as pct_of_total
  FROM leaderboard
  WHERE season = $1
`, [currentSeason]);

// RETURNING: get inserted/updated rows back
const newPost = await pool.query(`
  INSERT INTO posts (title, body, author_id, slug)
  VALUES ($1, $2, $3, $4)
  RETURNING id, title, slug, created_at
`, [title, body, authorId, slug]);
```

### Transactions in Node.js

```typescript
async function transferFunds(
  fromId: string,
  toId: string,
  amount: number
): Promise<void> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Lock both rows in a consistent order (lower id first) to prevent deadlocks
    const [id1, id2] = [fromId, toId].sort();
    await client.query(
      'SELECT id FROM accounts WHERE id = ANY($1) FOR UPDATE',
      [[id1, id2]]
    );

    const { rows: [from] } = await client.query(
      'SELECT balance FROM accounts WHERE id = $1',
      [fromId]
    );

    if (from.balance < amount) {
      throw new Error('Insufficient funds');
    }

    await client.query(
      'UPDATE accounts SET balance = balance - $1 WHERE id = $2',
      [amount, fromId]
    );

    await client.query(
      'UPDATE accounts SET balance = balance + $1 WHERE id = $2',
      [amount, toId]
    );

    await client.query(
      'INSERT INTO transactions (from_id, to_id, amount) VALUES ($1, $2, $3)',
      [fromId, toId, amount]
    );

    await client.query('COMMIT');
  } catch (err) {
    await client.query('ROLLBACK');
    throw err;
  } finally {
    client.release(); // always release back to pool
  }
}
```

### Connection Pooling with PgBouncer

```
PgBouncer pooling modes:

Session mode:
  - One server connection per client connection for the entire session
  - Least restrictive; supports all PostgreSQL features including LISTEN/NOTIFY
  - Minimal connection savings

Transaction mode (most common):
  - Server connection assigned only for duration of a transaction
  - Cannot use LISTEN/NOTIFY, SET configuration, advisory locks outside transactions
  - 10x-50x more connections supported per server connection
  - Best for typical web APIs using short transactions

Statement mode:
  - Connection returned after every single statement
  - Cannot use multi-statement transactions
  - Rarely used
```

```
# pgbouncer.ini
[databases]
myapp = host=postgres port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000    # max connections from apps
default_pool_size = 20    # connections to PostgreSQL
min_pool_size = 5
server_idle_timeout = 60
```

---

## 5. Common Mistakes & Pitfalls

**Missing indexes on foreign keys:**

```sql
-- PROBLEM: every query on posts that joins users will do a Seq Scan on posts
-- unless there's an index on the foreign key
CREATE TABLE posts (
  id UUID PRIMARY KEY,
  author_id UUID REFERENCES users(id)
  -- No index on author_id!
);

-- FIX
CREATE INDEX idx_posts_author_id ON posts(author_id);
-- PostgreSQL does NOT automatically index foreign key columns
```

**N+1 queries in application code:**

```typescript
// PROBLEM: 1 query for posts + 1 query per post for author = N+1
const posts = await db.query('SELECT * FROM posts LIMIT 100');
for (const post of posts) {
  const author = await db.query('SELECT * FROM users WHERE id = $1', [post.author_id]);
}

// FIX: JOIN in SQL
const posts = await db.query(`
  SELECT p.*, u.name as author_name, u.avatar_url
  FROM posts p
  JOIN users u ON u.id = p.author_id
  LIMIT 100
`);
```

**Using OFFSET for large pages:**

```sql
-- SLOW: database scans and discards 1,000,000 rows before returning 20
SELECT * FROM events ORDER BY created_at DESC OFFSET 1000000 LIMIT 20;

-- FAST: cursor-based, uses index efficiently
SELECT * FROM events
WHERE created_at < $cursor_timestamp
ORDER BY created_at DESC
LIMIT 20;
```

**Not handling serialization failures:**

```typescript
// SERIALIZABLE transactions can fail with:
// ERROR: could not serialize access due to concurrent update
// Applications MUST retry on this error

async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err: any) {
      if (err.code === '40001' && attempt < maxRetries) {
        // 40001 = serialization_failure — safe to retry
        await new Promise((r) => setTimeout(r, Math.random() * 100 * attempt));
        continue;
      }
      throw err;
    }
  }
  throw new Error('Max retries exceeded');
}
```

---

## 6. When to Use / Not Use

**Use PostgreSQL (SQL) when:**
- Complex relationships between entities with referential integrity
- ACID transactions are required (financial, medical, inventory)
- Ad-hoc reporting and analytics queries
- The data model is relatively stable and well-understood
- You need JOINs, aggregations, window functions

**Consider NoSQL when:**
- Schema is truly unknown or changes very frequently
- Document-centric data with no relationships (product catalogs with variable attributes)
- Extreme write throughput with simple access patterns (time-series, logs)
- Global distribution with low-latency reads across regions

---

## 7. Real-World Scenario

**Problem:** An e-commerce platform's order listing page takes 12 seconds to load for active merchants with 500,000+ orders. The query uses `OFFSET/LIMIT` pagination.

**Investigation:**

```sql
EXPLAIN ANALYZE
SELECT o.*, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.merchant_id = $1
  AND o.status = 'completed'
ORDER BY o.created_at DESC
OFFSET 490000 LIMIT 20;

-- Output shows: Seq Scan on orders, actual time=11834ms
```

**Root causes:**
1. `OFFSET 490000` forces PostgreSQL to scan 490,020 rows and discard most.
2. No index on `(merchant_id, status, created_at)`.

**Fix:**

```sql
-- 1. Add composite index
CREATE INDEX idx_orders_merchant_status_cursor
ON orders(merchant_id, status, created_at DESC)
WHERE status = 'completed'; -- partial index for the common case

-- 2. Switch to cursor pagination
SELECT o.id, o.total, o.created_at, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.merchant_id = $1
  AND o.status = 'completed'
  AND o.created_at < $cursor_timestamp  -- cursor from last item on previous page
ORDER BY o.created_at DESC
LIMIT 21; -- 21 to detect hasMore
```

**Result:** Query time drops from 12 seconds to 4ms. The index is used, and the cursor replaces the expensive OFFSET scan.

---

## 8. Interview Questions

**Q1: What is an index and how does a B-tree index work internally?**

An index is a separate data structure that stores a sorted subset of table columns with pointers to the actual table rows (heap tuples). A B-tree is a self-balancing tree where each node holds multiple key-pointer pairs. Lookups start at the root, follow pointers based on key comparisons, and reach a leaf node in O(log n) steps. Leaf nodes are linked sequentially, enabling efficient range scans. PostgreSQL's B-tree supports equality, inequality, range queries, and sorting. Writes must update the tree, which is why too many indexes slow down writes.

**Q2: Explain the four ACID properties with an example.**

A bank transfer: (1) **Atomicity** — debit $100 from account A and credit $100 to account B are a single unit — if the credit fails, the debit is rolled back. (2) **Consistency** — the total money in the system is the same before and after (constraint: sum of all balances = constant). (3) **Isolation** — two concurrent transfers do not see each other's intermediate states — one transfer sees the pre-transfer state, the other sees the post-transfer state. (4) **Durability** — after COMMIT, the transfer survives a power outage; WAL ensures it can be replayed on recovery.

**Q3: When would you use a covering index?**

When a query can be satisfied entirely from the index without fetching the actual table rows (Index Only Scan). For example: `SELECT name, email FROM users WHERE status = 'active'` — if the index is `CREATE INDEX ON users(status) INCLUDE (name, email)`, PostgreSQL can return `name` and `email` directly from the index without a heap fetch. This is especially valuable for high-frequency read queries on large tables where the heap access is the bottleneck.

**Q4: What isolation level does PostgreSQL use by default and what phenomena does it prevent?**

PostgreSQL defaults to **Read Committed**. It prevents dirty reads (reading another transaction's uncommitted data) but allows non-repeatable reads (the same row can return different values within one transaction if another transaction commits between reads) and phantom reads (a range query can return different rows). For most web applications, Read Committed is sufficient. Use Repeatable Read or Serializable for financial operations where consistency across multiple reads within a transaction is required.

**Q5: What is the difference between a dirty read and a phantom read?**

A **dirty read** is reading uncommitted data from another transaction — you might read data that is then rolled back. Prevented in Read Committed and above. A **phantom read** is when a query that reads a range returns different rows on two executions within the same transaction because another transaction inserted or deleted rows in that range. Example: COUNT(*) WHERE status='pending' returns 5, then 7 on a second read. Prevented only at Serializable (and mostly at Repeatable Read in PostgreSQL due to MVCC).

**Q6: What is a CTE and when would you use one?**

A CTE (Common Table Expression), written with the `WITH` keyword, is a named temporary result set within a query. Use cases: (1) **Readability** — break complex queries into named logical steps. (2) **Recursive queries** — `WITH RECURSIVE` for tree/graph traversal (e.g., organizational hierarchies, bill of materials). (3) **Multi-step transformations** — chain multiple result sets in one query. In PostgreSQL, non-recursive CTEs can be inlined into the main query by the optimizer; use `MATERIALIZED` to force them to be computed once.

**Q7: Explain window functions with an example.**

Window functions compute a value for each row using a "window" of related rows, without collapsing them into a single aggregate row. `SUM(score) OVER ()` computes the total score for all rows and attaches it to each row. `ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)` assigns a rank within each department by salary. Key functions: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `SUM/AVG/COUNT OVER`. Essential for analytics queries: running totals, percentages of total, previous/next value comparisons.

**Q8: Why is cursor pagination better than offset pagination for large datasets?**

Offset pagination (`OFFSET 10000 LIMIT 20`) requires the database to scan and discard 10,000 rows before returning 20 — this cost grows linearly with the page number. Cursor pagination uses a WHERE clause that leverages an index directly: `WHERE created_at < $cursor ORDER BY created_at DESC LIMIT 20` — this is O(log n) regardless of which page you are on. Additionally, offset pagination gives inconsistent results when rows are inserted or deleted between page loads (rows shift), while cursor pagination is stable because it uses absolute position.

---

## 9. Exercises

**Exercise 1: Index audit**

Given a table `events(id, user_id, type, payload JSONB, created_at)` with 50 million rows, write the optimal indexes for these query patterns:
- `WHERE user_id = $1 AND type = 'click' ORDER BY created_at DESC LIMIT 20`
- `WHERE payload @> '{"campaign_id": "X"}' AND created_at > NOW() - INTERVAL '7 days'`
- `WHERE user_id = $1` with SELECT only `id, type, created_at` (no payload)

Explain why you chose each index type and column order.

*Hint:* Consider composite, partial, GIN, and covering indexes. Run `EXPLAIN ANALYZE` on each pattern.

**Exercise 2: Window function report**

Write a single SQL query (using CTEs and window functions) that returns, for each month in the last 12 months:
- Total revenue
- Number of orders
- Revenue growth vs previous month (as a percentage)
- Cumulative revenue year-to-date
- This month's revenue as a percentage of the year's total

*Hint:* Use `LAG()` for previous month, `SUM() OVER (PARTITION BY year ORDER BY month)` for YTD.

**Exercise 3: Optimistic locking implementation**

Implement an article editing system where:
- Multiple users can edit the same article concurrently
- If two users try to save at the same time, the second save fails with a conflict error
- The client must resend with the latest version to proceed
- Write the SQL and TypeScript service layer

*Hint:* Add `version INT DEFAULT 1` column. `UPDATE ... WHERE id = $1 AND version = $expected; RETURNING *` — if 0 rows returned, version conflict.

---

## 10. Further Reading

- [PostgreSQL documentation — EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [PostgreSQL documentation — Indexes](https://www.postgresql.org/docs/current/indexes.html)
- [PostgreSQL documentation — Transaction isolation](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Use the Index, Luke — indexing for developers](https://use-the-index-luke.com/)
- [pganalyze — index advisor](https://pganalyze.com/)
- [PgBouncer documentation](https://www.pgbouncer.org/config.html)
- [Postgres EXPLAIN visualizer (explain.depesz.com)](https://explain.depesz.com/)
- Book: *PostgreSQL: Up and Running* by Regina Obe & Leo Hsu
- Book: *The Art of PostgreSQL* by Dimitri Fontaine
