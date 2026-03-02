# PostgreSQL Cheatsheet

## JSONB Operators

```sql
-- Access operators
data->'key'           -- returns JSONB
data->>'key'          -- returns text
data#>'{a,b}'         -- nested path → JSONB
data#>>'{a,b}'        -- nested path → text
data->0               -- array element by index

-- Containment
'{"a":1}'::jsonb @> '{"a":1}'::jsonb   -- left contains right
'{"a":1}'::jsonb <@ '{"a":1}'::jsonb   -- left contained in right

-- Key existence
data ? 'key'           -- key exists
data ?| ARRAY['a','b'] -- any key exists
data ?& ARRAY['a','b'] -- all keys exist

-- Modification
data || '{"c":3}'::jsonb              -- merge/concatenate
data - 'key'                          -- delete key
data - ARRAY['a','b']                 -- delete multiple keys
data #- '{a,b}'                       -- delete nested path
jsonb_set(data, '{a,b}', '"new"')     -- set nested value
jsonb_insert(data, '{arr,0}', '99')   -- insert into array

-- Iteration
jsonb_each(data)                   -- set of (key text, value jsonb)
jsonb_each_text(data)              -- set of (key text, value text)
jsonb_object_keys(data)            -- set of keys
jsonb_array_elements(data)         -- expand array to rows
jsonb_array_length(data)           -- array length

-- Querying
SELECT * FROM products WHERE attributes @> '{"color":"red"}';
SELECT * FROM logs WHERE payload->>'status' = 'error';
SELECT * FROM events WHERE data ? 'urgent';

-- Index for containment queries
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
CREATE INDEX idx_products_attrs_ops ON products USING GIN (attributes jsonb_path_ops);
-- jsonb_path_ops is smaller and faster for @> only (no ? operators)
```

---

## Full-Text Search

```sql
-- tsvector — preprocessed document
to_tsvector('english', 'The quick brown fox')
-- → 'brown':3 'fox':4 'quick':2

-- tsquery — search expression
to_tsquery('english', 'quick & fox')       -- AND
to_tsquery('english', 'quick | slow')      -- OR
to_tsquery('english', '!quick & fox')      -- NOT
plainto_tsquery('english', 'quick fox')    -- phrase → quick & fox
phraseto_tsquery('english', 'quick fox')   -- adjacent → quick <-> fox
websearch_to_tsquery('english', '"quick fox" -slow') -- Google-style

-- Match operator
to_tsvector('english', title) @@ to_tsquery('english', 'database')

-- Ranking
ts_rank(vector, query)         -- relevance score based on frequency
ts_rank_cd(vector, query)      -- cover density (considers proximity)
ts_headline('english', text, query)  -- highlight matching terms

-- Generated column for indexing (PostgreSQL 12+)
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('english', coalesce(body,'')), 'B')
  ) STORED;

CREATE INDEX idx_articles_fts ON articles USING GIN (search_vector);

-- Query
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgres & index') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

---

## Window Functions (Advanced)

```sql
-- Syntax: function() OVER (PARTITION BY ... ORDER BY ... ROWS/RANGE ...)

-- Ranking
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)   -- unique rank
RANK()       OVER (PARTITION BY dept ORDER BY salary DESC)   -- gaps on ties
DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC)   -- no gaps
NTILE(4)     OVER (ORDER BY salary)                          -- quartile
PERCENT_RANK() OVER (ORDER BY salary)                        -- 0.0 to 1.0

-- Offset
LAG(salary, 1, 0)  OVER (PARTITION BY dept ORDER BY hire_date)  -- previous row
LEAD(salary, 1, 0) OVER (PARTITION BY dept ORDER BY hire_date)  -- next row
FIRST_VALUE(salary) OVER (PARTITION BY dept ORDER BY salary DESC)
LAST_VALUE(salary)  OVER (PARTITION BY dept ORDER BY salary DESC
                           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- Aggregates as window functions
SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at)  -- running total
AVG(amount) OVER (PARTITION BY user_id
                  ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- 7-day moving avg
COUNT(*) OVER (PARTITION BY category)                        -- group size

-- Named windows (DRY)
SELECT
  salary,
  AVG(salary) OVER w AS dept_avg,
  RANK()       OVER w AS dept_rank
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);

-- Top-N per group (classic use case)
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY created_at DESC) AS rn
  FROM products
)
SELECT * FROM ranked WHERE rn <= 3;
```

---

## Partitioning

```sql
-- Range partitioning by date
CREATE TABLE events (
  id        bigint,
  user_id   int,
  created_at timestamptz NOT NULL,
  payload   jsonb
) PARTITION BY RANGE (created_at);

CREATE TABLE events_2024_q1 PARTITION OF events
  FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');
CREATE TABLE events_2024_q2 PARTITION OF events
  FOR VALUES FROM ('2024-04-01') TO ('2024-07-01');

-- Default partition (catches anything unmatched)
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- List partitioning by value
CREATE TABLE orders (
  id      bigint,
  region  text,
  amount  numeric
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('us-east', 'us-west');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('eu-west', 'eu-north');

-- Hash partitioning for even distribution
CREATE TABLE sessions (id bigint, data jsonb) PARTITION BY HASH (id);
CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... sessions_2, sessions_3

-- Indexes on partitioned table (creates on all partitions)
CREATE INDEX ON events (user_id, created_at);

-- Attach/detach partitions (zero downtime archival)
ALTER TABLE events DETACH PARTITION events_2022_q1;
ALTER TABLE events ATTACH PARTITION events_2025_q1
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
```

---

## Replication Config

```ini
# postgresql.conf — primary server
wal_level = replica            # minimum for replication
max_wal_senders = 5            # number of replication connections
wal_keep_size = 256MB          # WAL retained for standbys
synchronous_commit = on        # 'off' for async (faster, risk of data loss)
# synchronous_standby_names = 'standby1'  # for synchronous replication

# pg_hba.conf — allow replication connections
host replication replicator 10.0.0.2/32 scram-sha-256

# Create replication user
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secret';

# On standby: recovery.conf (PG < 12) or postgresql.conf (PG >= 12)
primary_conninfo = 'host=10.0.0.1 user=replicator password=secret'
hot_standby = on               # allow read queries on standby

# Logical replication (for table-level replication, cross-version)
-- On publisher:
CREATE PUBLICATION my_pub FOR TABLE orders, products;
-- On subscriber:
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=publisher dbname=mydb user=replicator password=secret'
  PUBLICATION my_pub;
```

---

## Useful pg_stat Views

```sql
-- Active queries and wait events
SELECT pid, state, wait_event_type, wait_event, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Kill long-running query
SELECT pg_cancel_backend(pid);   -- sends SIGINT (cancels query, keeps connection)
SELECT pg_terminate_backend(pid); -- sends SIGTERM (kills connection)

-- Table access stats (cache hit ratio should be > 99%)
SELECT relname,
  heap_blks_read, heap_blks_hit,
  round(heap_blks_hit::numeric / nullif(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;

-- Index usage
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Unused indexes (candidates for removal)
SELECT schemaname, tablename, indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE 'pg_toast%';

-- Table bloat (dead tuples)
SELECT relname, n_live_tup, n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_ratio,
  last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY dead_ratio DESC;

-- Lock waits
SELECT blocked.pid, blocked.query, blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Database size
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;

-- Table + index sizes
SELECT tablename,
  pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total,
  pg_size_pretty(pg_relation_size(tablename::regclass)) AS table_only,
  pg_size_pretty(pg_total_relation_size(tablename::regclass) - pg_relation_size(tablename::regclass)) AS indexes
FROM pg_tables WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;
```

---

## VACUUM / ANALYZE

```sql
-- Manual vacuum (reclaims dead tuple space, does not return to OS)
VACUUM table_name;

-- VACUUM FULL (rewrites table, returns space to OS, locks table)
VACUUM FULL table_name;

-- Update statistics for planner
ANALYZE table_name;

-- Combined
VACUUM ANALYZE table_name;

-- Verbose output
VACUUM VERBOSE ANALYZE table_name;

-- Autovacuum tuning (per table)
ALTER TABLE high_churn_table SET (
  autovacuum_vacuum_scale_factor = 0.01,    -- vacuum when 1% is dead (default 20%)
  autovacuum_analyze_scale_factor = 0.005,  -- analyze when 0.5% changed
  autovacuum_vacuum_cost_delay = 2          -- reduce IO throttling (ms)
);

-- Check autovacuum is running
SHOW autovacuum;
SELECT * FROM pg_stat_activity WHERE query LIKE 'autovacuum%';
```

---

## Common Extensions

```sql
-- uuid-ossp: generate UUIDs
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();          -- random UUID
SELECT uuid_generate_v1mc();        -- time-based, privacy-preserving

-- pgcrypto: hashing and encryption
CREATE EXTENSION IF NOT EXISTS pgcrypto;
SELECT crypt('password', gen_salt('bf', 12));      -- bcrypt hash
SELECT crypt('password', stored_hash) = stored_hash; -- verify
SELECT encode(digest('data', 'sha256'), 'hex');    -- SHA-256

-- pg_trgm: fuzzy search and similarity
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SELECT similarity('hello', 'helo');                -- 0.44
SELECT 'hello' % 'helo';                           -- true if similarity > threshold
CREATE INDEX idx_trgm ON articles USING GIN (title gin_trgm_ops);
-- Supports LIKE/ILIKE queries on the index:
SELECT * FROM articles WHERE title ILIKE '%postgr%';

-- pg_stat_statements: query performance tracking
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- Add to postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;
pg_stat_statements_reset(); -- reset counters

-- btree_gist: GiST indexes on scalar types (for exclusion constraints)
CREATE EXTENSION IF NOT EXISTS btree_gist;
-- Prevent double-booking:
ALTER TABLE reservations ADD CONSTRAINT no_overlap
  EXCLUDE USING GIST (room_id WITH =, tstzrange(start_at, end_at) WITH &&);

-- hstore: key-value in a column (prefer JSONB for new code)
CREATE EXTENSION IF NOT EXISTS hstore;
```

---

## Connection Pooling (PgBouncer)

```ini
# pgbouncer.ini
[databases]
mydb = host=127.0.0.1 port=5432 dbname=mydb

[pgbouncer]
listen_port = 5432
listen_addr = 0.0.0.0
auth_type = scram-sha-256
auth_file = /etc/pgbouncer/userlist.txt
logfile = /var/log/pgbouncer/pgbouncer.log
pidfile = /var/run/pgbouncer/pgbouncer.pid

# Pool mode: session | transaction | statement
# transaction = recommended for stateless apps
pool_mode = transaction

# Pool sizes
default_pool_size = 25       # connections per db/user pair
max_client_conn = 1000       # total client connections
reserve_pool_size = 5        # extra connections for emergency
reserve_pool_timeout = 3     # seconds before using reserve pool

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0

# Transactions mode caveats — these features break with transaction pooling:
# SET statements, advisory locks, LISTEN/NOTIFY, temp tables
# Use session pooling if you need them
```

```bash
# Monitor PgBouncer
psql -p 5432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
psql -p 5432 -U pgbouncer pgbouncer -c "SHOW STATS;"
psql -p 5432 -U pgbouncer pgbouncer -c "SHOW CLIENTS;"
```

---

## Query Optimization Quick Reference

```sql
-- Explain plans
EXPLAIN SELECT ...;                  -- estimated plan
EXPLAIN ANALYZE SELECT ...;          -- actual plan + timing (executes query!)
EXPLAIN (ANALYZE, BUFFERS) SELECT ...; -- includes cache hit info

-- Force index (for testing)
SET enable_seqscan = off;
-- Remember to reset: SET enable_seqscan = on;

-- Partial index (index subset of rows)
CREATE INDEX idx_active_users ON users (email) WHERE active = true;

-- Expression index
CREATE INDEX idx_lower_email ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com'; -- uses index

-- Covering index (INCLUDE — avoids heap fetch for indexed queries)
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (total, created_at);
SELECT total, created_at FROM orders WHERE user_id = 1; -- index-only scan
```
