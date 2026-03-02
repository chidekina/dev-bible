# SQL Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## SELECT Basics

```sql
SELECT col1, col2, col3 AS alias
FROM   table_name
WHERE  condition
ORDER  BY col1 ASC, col2 DESC
LIMIT  10
OFFSET 20;

-- Distinct
SELECT DISTINCT department FROM employees;

-- Filtering
WHERE age BETWEEN 18 AND 65
WHERE name LIKE 'A%'           -- starts with A
WHERE name ILIKE '%smith%'     -- case-insensitive (PostgreSQL)
WHERE status IN ('active', 'pending')
WHERE deleted_at IS NULL
WHERE score IS NOT NULL
```

---

## JOINs

```sql
-- INNER JOIN: only matching rows
SELECT u.name, o.total
FROM   users u
INNER  JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: all from left, NULLs for non-matching right
SELECT u.name, o.total
FROM   users u
LEFT   JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN: all rows from both sides
-- CROSS JOIN: cartesian product (every combination)

-- Self join
SELECT e.name, m.name AS manager
FROM   employees e
LEFT   JOIN employees m ON e.manager_id = m.id;

-- Multiple joins
SELECT u.name, o.id, p.name AS product
FROM   users u
JOIN   orders o      ON u.id = o.user_id
JOIN   order_items oi ON o.id = oi.order_id
JOIN   products p    ON oi.product_id = p.id;
```

---

## Aggregation & GROUP BY

```sql
SELECT
  department,
  COUNT(*)            AS headcount,
  AVG(salary)         AS avg_salary,
  MAX(salary)         AS top_salary,
  MIN(hire_date)      AS first_hire,
  SUM(sales)          AS total_sales
FROM   employees
WHERE  active = true
GROUP  BY department
HAVING COUNT(*) > 5         -- filter AFTER aggregation
ORDER  BY avg_salary DESC;

-- Aggregate functions
COUNT(*)                          -- all rows
COUNT(col)                        -- non-NULL values only
COUNT(DISTINCT col)
SUM / AVG / MIN / MAX
STRING_AGG(col, ', ')             -- PostgreSQL
GROUP_CONCAT(col SEPARATOR ',')   -- MySQL
ARRAY_AGG(col)                    -- PostgreSQL
```

---

## Subqueries & CTEs

```sql
-- Subquery in WHERE
SELECT name FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Subquery in FROM (derived table)
SELECT dept, avg_sal FROM (
  SELECT department AS dept, AVG(salary) AS avg_sal
  FROM employees GROUP BY department
) sub WHERE avg_sal > 70000;

-- CTE (WITH)
WITH high_earners AS (
  SELECT id, name, salary FROM employees WHERE salary > 100000
),
their_orders AS (
  SELECT user_id, COUNT(*) AS order_count
  FROM orders GROUP BY user_id
)
SELECT h.name, h.salary, COALESCE(o.order_count, 0) AS orders
FROM   high_earners h
LEFT   JOIN their_orders o ON h.id = o.user_id;

-- Recursive CTE (tree traversal)
WITH RECURSIVE org_tree AS (
  SELECT id, name, manager_id, 0 AS depth
  FROM   employees WHERE manager_id IS NULL
  UNION ALL
  SELECT e.id, e.name, e.manager_id, t.depth + 1
  FROM   employees e
  JOIN   org_tree t ON e.manager_id = t.id
)
SELECT * FROM org_tree ORDER BY depth;
```

---

## Window Functions

```sql
SELECT
  name, department, salary,
  -- Ranking
  ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC) AS row_num,
  RANK()       OVER (PARTITION BY department ORDER BY salary DESC) AS rank,
  DENSE_RANK() OVER (PARTITION BY department ORDER BY salary DESC) AS dense_rank,
  NTILE(4)     OVER (ORDER BY salary)                              AS quartile,
  -- Offsets
  LAG(salary, 1)  OVER (PARTITION BY department ORDER BY hire_date) AS prev_salary,
  LEAD(salary, 1) OVER (PARTITION BY department ORDER BY hire_date) AS next_salary,
  FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS top_in_dept,
  -- Aggregates as window
  SUM(salary) OVER (PARTITION BY department)            AS dept_total,
  AVG(salary) OVER ()                                   AS company_avg,
  SUM(salary) OVER (ORDER BY hire_date ROWS UNBOUNDED PRECEDING) AS running_total
FROM employees;
```

---

## DML: INSERT / UPDATE / DELETE / UPSERT

```sql
-- INSERT
INSERT INTO users (name, email, created_at)
VALUES ('Alice', 'alice@example.com', NOW());

-- INSERT multiple rows
INSERT INTO tags (name) VALUES ('js'), ('ts'), ('sql');

-- INSERT from SELECT
INSERT INTO archived_users SELECT * FROM users WHERE deleted_at IS NOT NULL;

-- UPDATE
UPDATE products SET price = price * 1.10, updated_at = NOW()
WHERE category = 'electronics';

-- UPDATE with JOIN (PostgreSQL)
UPDATE orders o SET status = 'fulfilled'
FROM shipments s
WHERE s.order_id = o.id AND s.delivered_at IS NOT NULL;

-- DELETE
DELETE FROM sessions WHERE expires_at < NOW();

-- UPSERT (PostgreSQL)
INSERT INTO settings (user_id, key, value)
VALUES (1, 'theme', 'dark')
ON CONFLICT (user_id, key) DO UPDATE SET value = EXCLUDED.value;

-- UPSERT (MySQL)
INSERT INTO settings (user_id, key, value)
VALUES (1, 'theme', 'dark')
ON DUPLICATE KEY UPDATE value = VALUES(value);
```

---

## DDL: CREATE / ALTER / DROP

```sql
CREATE TABLE products (
  id          SERIAL PRIMARY KEY,
  sku         VARCHAR(64) NOT NULL UNIQUE,
  name        TEXT NOT NULL,
  price       NUMERIC(10,2) NOT NULL CHECK (price >= 0),
  category_id INT REFERENCES categories(id) ON DELETE SET NULL,
  created_at  TIMESTAMPTZ NOT NULL DEFAULT NOW()
);

ALTER TABLE products ADD COLUMN weight NUMERIC(8,3);
ALTER TABLE products ALTER COLUMN name SET NOT NULL;
ALTER TABLE products DROP COLUMN legacy_field;

-- Indexes
CREATE INDEX idx_products_category ON products(category_id);
CREATE UNIQUE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_created ON orders(created_at DESC);

-- Partial index
CREATE INDEX idx_active_users ON users(email) WHERE deleted_at IS NULL;

-- Covering index (PostgreSQL)
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (status, total);

DROP TABLE IF EXISTS temp_data;
DROP INDEX IF EXISTS idx_old;
```

---

## Transactions

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
-- On error: ROLLBACK;

-- Savepoints
BEGIN;
  INSERT INTO orders ...;
  SAVEPOINT sp1;
  INSERT INTO order_items ...;  -- might fail
  ROLLBACK TO sp1;              -- undo only last insert
COMMIT;

-- Isolation levels (PostgreSQL)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- default
SET TRANSACTION ISOLATION LEVEL REPEATABLE READ;
SET TRANSACTION ISOLATION LEVEL SERIALIZABLE;
```

---

## EXPLAIN / EXPLAIN ANALYZE

```sql
EXPLAIN SELECT * FROM orders WHERE user_id = 42;

EXPLAIN ANALYZE SELECT * FROM orders WHERE user_id = 42;

EXPLAIN (ANALYZE, BUFFERS, FORMAT JSON)
SELECT * FROM orders o JOIN users u ON o.user_id = u.id;
```

| Term | Meaning |
|------|---------|
| `Seq Scan` | Full table scan — consider adding an index |
| `Index Scan` | Uses index to find rows |
| `Index Only Scan` | Reads only the index (fastest) |
| `Bitmap Heap Scan` | Batch index lookup |
| `Nested Loop` | JOIN for small row sets |
| `Hash Join` | JOIN for larger sets |
| `Merge Join` | JOIN on pre-sorted sets |
| `rows=` | Estimated vs actual rows |
| `cost=` | Planner relative cost unit |

---

## Useful Functions

```sql
-- String
UPPER(s), LOWER(s), LENGTH(s), TRIM(s)
SUBSTRING(s FROM 1 FOR 5)
REPLACE(s, 'old', 'new')
CONCAT(s1, ' ', s2)           -- or: s1 || ' ' || s2
SPLIT_PART(s, ',', 1)         -- PostgreSQL
REGEXP_REPLACE(s, '[0-9]+', 'N')

-- Numeric
ROUND(3.14159, 2)  -- 3.14
CEIL(2.1)          -- 3
FLOOR(2.9)         -- 2
ABS(-5)            -- 5
MOD(10, 3)         -- 1

-- Date/Time (PostgreSQL)
NOW()                           -- current timestamp with TZ
CURRENT_DATE                    -- today date
DATE_TRUNC('month', NOW())      -- first of current month
EXTRACT(YEAR FROM created_at)
AGE(NOW(), hire_date)           -- interval
created_at + INTERVAL '7 days'
TO_CHAR(NOW(), 'YYYY-MM-DD')

-- Conditional
COALESCE(a, b, c)               -- first non-NULL
NULLIF(a, b)                    -- NULL if a = b
GREATEST(a, b, c) / LEAST(a, b, c)
CASE WHEN x > 0 THEN 'positive' WHEN x < 0 THEN 'negative' ELSE 'zero' END
```

---

## Common Patterns

```sql
-- Pagination (offset-based)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 40;

-- Keyset pagination (faster for large offsets)
SELECT * FROM posts
WHERE created_at < :last_seen_date
ORDER BY created_at DESC LIMIT 20;

-- Top N per group
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY score DESC) AS rn
  FROM products
) t WHERE rn <= 3;

-- Running total
SELECT date, revenue,
  SUM(revenue) OVER (ORDER BY date) AS cumulative
FROM daily_sales;

-- Soft delete
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;
UPDATE users SET deleted_at = NOW() WHERE id = 42;  -- "delete"
```

---

## Index Strategy

| Use Case | Index Type |
|----------|-----------|
| Exact match / range queries | B-tree (default) |
| Full-text search | GIN + `tsvector` |
| JSON field queries | GIN |
| Geospatial / range types | GiST |
| Array contains operator | GIN |
| Low-cardinality + filter | Partial index |
| Cover query without heap | Covering index (`INCLUDE`) |
| Case-insensitive search | Index on `LOWER(col)` |
