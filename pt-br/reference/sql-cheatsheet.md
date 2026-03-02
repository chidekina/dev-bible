# SQL Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa.

---

## SELECT Básico

```sql
SELECT col1, col2, col3 AS alias
FROM   table_name
WHERE  condition
ORDER  BY col1 ASC, col2 DESC
LIMIT  10
OFFSET 20;

-- Distinct
SELECT DISTINCT department FROM employees;

-- Filtragem
WHERE age BETWEEN 18 AND 65
WHERE name LIKE 'A%'           -- começa com A
WHERE name ILIKE '%smith%'     -- sem distinção de maiúsculas (PostgreSQL)
WHERE status IN ('active', 'pending')
WHERE deleted_at IS NULL
WHERE score IS NOT NULL
```

---

## JOINs

```sql
-- INNER JOIN: apenas linhas correspondentes
SELECT u.name, o.total
FROM   users u
INNER  JOIN orders o ON u.id = o.user_id;

-- LEFT JOIN: todos da esquerda, NULL para os não correspondentes da direita
SELECT u.name, o.total
FROM   users u
LEFT   JOIN orders o ON u.id = o.user_id;

-- FULL OUTER JOIN: todas as linhas de ambos os lados
-- CROSS JOIN: produto cartesiano (toda combinação possível)

-- Self join
SELECT e.name, m.name AS manager
FROM   employees e
LEFT   JOIN employees m ON e.manager_id = m.id;

-- Múltiplos joins
SELECT u.name, o.id, p.name AS product
FROM   users u
JOIN   orders o      ON u.id = o.user_id
JOIN   order_items oi ON o.id = oi.order_id
JOIN   products p    ON oi.product_id = p.id;
```

---

## Agregação e GROUP BY

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
HAVING COUNT(*) > 5         -- filtra DEPOIS da agregação
ORDER  BY avg_salary DESC;

-- Funções de agregação
COUNT(*)                          -- todas as linhas
COUNT(col)                        -- apenas valores não-NULL
COUNT(DISTINCT col)
SUM / AVG / MIN / MAX
STRING_AGG(col, ', ')             -- PostgreSQL
GROUP_CONCAT(col SEPARATOR ',')   -- MySQL
ARRAY_AGG(col)                    -- PostgreSQL
```

---

## Subqueries e CTEs

```sql
-- Subquery no WHERE
SELECT name FROM users
WHERE id IN (SELECT user_id FROM orders WHERE total > 1000);

-- Subquery no FROM (tabela derivada)
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

-- CTE recursiva (travessia de árvore)
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
  -- Deslocamentos
  LAG(salary, 1)  OVER (PARTITION BY department ORDER BY hire_date) AS prev_salary,
  LEAD(salary, 1) OVER (PARTITION BY department ORDER BY hire_date) AS next_salary,
  FIRST_VALUE(salary) OVER (PARTITION BY department ORDER BY salary DESC) AS top_in_dept,
  -- Agregados como window
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

-- INSERT de múltiplas linhas
INSERT INTO tags (name) VALUES ('js'), ('ts'), ('sql');

-- INSERT a partir de SELECT
INSERT INTO archived_users SELECT * FROM users WHERE deleted_at IS NOT NULL;

-- UPDATE
UPDATE products SET price = price * 1.10, updated_at = NOW()
WHERE category = 'electronics';

-- UPDATE com JOIN (PostgreSQL)
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

-- Index parcial
CREATE INDEX idx_active_users ON users(email) WHERE deleted_at IS NULL;

-- Index cobrindo (PostgreSQL)
CREATE INDEX idx_orders_cover ON orders(user_id) INCLUDE (status, total);

DROP TABLE IF EXISTS temp_data;
DROP INDEX IF EXISTS idx_old;
```

---

## Transações

```sql
BEGIN;
  UPDATE accounts SET balance = balance - 500 WHERE id = 1;
  UPDATE accounts SET balance = balance + 500 WHERE id = 2;
COMMIT;
-- Em caso de erro: ROLLBACK;

-- Savepoints
BEGIN;
  INSERT INTO orders ...;
  SAVEPOINT sp1;
  INSERT INTO order_items ...;  -- pode falhar
  ROLLBACK TO sp1;              -- desfaz apenas o último insert
COMMIT;

-- Níveis de isolamento (PostgreSQL)
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;   -- padrão
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

| Termo | Significado |
|-------|-------------|
| `Seq Scan` | Varredura completa da tabela — considere adicionar um index |
| `Index Scan` | Usa index para localizar linhas |
| `Index Only Scan` | Lê apenas o index (mais rápido) |
| `Bitmap Heap Scan` | Lookup em lote via index |
| `Nested Loop` | JOIN para conjuntos pequenos de linhas |
| `Hash Join` | JOIN para conjuntos maiores |
| `Merge Join` | JOIN em conjuntos pré-ordenados |
| `rows=` | Linhas estimadas vs. reais |
| `cost=` | Unidade de custo relativa do planejador |

---

## Funções Úteis

```sql
-- String
UPPER(s), LOWER(s), LENGTH(s), TRIM(s)
SUBSTRING(s FROM 1 FOR 5)
REPLACE(s, 'old', 'new')
CONCAT(s1, ' ', s2)           -- ou: s1 || ' ' || s2
SPLIT_PART(s, ',', 1)         -- PostgreSQL
REGEXP_REPLACE(s, '[0-9]+', 'N')

-- Numérico
ROUND(3.14159, 2)  -- 3.14
CEIL(2.1)          -- 3
FLOOR(2.9)         -- 2
ABS(-5)            -- 5
MOD(10, 3)         -- 1

-- Data/Hora (PostgreSQL)
NOW()                           -- timestamp atual com fuso horário
CURRENT_DATE                    -- data de hoje
DATE_TRUNC('month', NOW())      -- primeiro dia do mês atual
EXTRACT(YEAR FROM created_at)
AGE(NOW(), hire_date)           -- intervalo
created_at + INTERVAL '7 days'
TO_CHAR(NOW(), 'YYYY-MM-DD')

-- Condicional
COALESCE(a, b, c)               -- primeiro valor não-NULL
NULLIF(a, b)                    -- NULL se a = b
GREATEST(a, b, c) / LEAST(a, b, c)
CASE WHEN x > 0 THEN 'positive' WHEN x < 0 THEN 'negative' ELSE 'zero' END
```

---

## Padrões Comuns

```sql
-- Paginação (baseada em offset)
SELECT * FROM posts ORDER BY created_at DESC LIMIT 20 OFFSET 40;

-- Paginação por cursor (mais rápida para grandes offsets)
SELECT * FROM posts
WHERE created_at < :last_seen_date
ORDER BY created_at DESC LIMIT 20;

-- Top N por grupo
SELECT * FROM (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY score DESC) AS rn
  FROM products
) t WHERE rn <= 3;

-- Total acumulado
SELECT date, revenue,
  SUM(revenue) OVER (ORDER BY date) AS cumulative
FROM daily_sales;

-- Soft delete
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;
CREATE VIEW active_users AS SELECT * FROM users WHERE deleted_at IS NULL;
UPDATE users SET deleted_at = NOW() WHERE id = 42;  -- "deletar"
```

---

## Estratégia de Indexes

| Caso de Uso | Tipo de Index |
|-------------|---------------|
| Busca exata / range queries | B-tree (padrão) |
| Busca full-text | GIN + `tsvector` |
| Queries em campos JSON | GIN |
| Tipos geoespaciais / range | GiST |
| Operador de array contains | GIN |
| Baixa cardinalidade + filtro | Index parcial |
| Cobrir query sem heap | Index cobrindo (`INCLUDE`) |
| Busca sem distinção de maiúsculas | Index em `LOWER(col)` |
