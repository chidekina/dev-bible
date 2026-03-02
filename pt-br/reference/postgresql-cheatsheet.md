# PostgreSQL Cheatsheet

## Operadores JSONB

```sql
-- Operadores de acesso
data->'key'           -- retorna JSONB
data->>'key'          -- retorna texto
data#>'{a,b}'         -- caminho aninhado → JSONB
data#>>'{a,b}'        -- caminho aninhado → texto
data->0               -- elemento de array por índice

-- Contenção
'{"a":1}'::jsonb @> '{"a":1}'::jsonb   -- esquerda contém direita
'{"a":1}'::jsonb <@ '{"a":1}'::jsonb   -- esquerda está contida na direita

-- Existência de chave
data ? 'key'           -- chave existe
data ?| ARRAY['a','b'] -- qualquer chave existe
data ?& ARRAY['a','b'] -- todas as chaves existem

-- Modificação
data || '{"c":3}'::jsonb              -- mesclar/concatenar
data - 'key'                          -- deletar chave
data - ARRAY['a','b']                 -- deletar múltiplas chaves
data #- '{a,b}'                       -- deletar caminho aninhado
jsonb_set(data, '{a,b}', '"new"')     -- definir valor aninhado
jsonb_insert(data, '{arr,0}', '99')   -- inserir em array

-- Iteração
jsonb_each(data)                   -- conjunto de (key text, value jsonb)
jsonb_each_text(data)              -- conjunto de (key text, value text)
jsonb_object_keys(data)            -- conjunto de chaves
jsonb_array_elements(data)         -- expande array em linhas
jsonb_array_length(data)           -- comprimento do array

-- Consultas
SELECT * FROM products WHERE attributes @> '{"color":"red"}';
SELECT * FROM logs WHERE payload->>'status' = 'error';
SELECT * FROM events WHERE data ? 'urgent';

-- Índice para consultas de contenção
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
CREATE INDEX idx_products_attrs_ops ON products USING GIN (attributes jsonb_path_ops);
-- jsonb_path_ops é menor e mais rápido apenas para @> (sem operadores ?)
```

---

## Full-Text Search

```sql
-- tsvector — documento pré-processado
to_tsvector('english', 'The quick brown fox')
-- → 'brown':3 'fox':4 'quick':2

-- tsquery — expressão de busca
to_tsquery('english', 'quick & fox')       -- E lógico
to_tsquery('english', 'quick | slow')      -- OU lógico
to_tsquery('english', '!quick & fox')      -- NÃO lógico
plainto_tsquery('english', 'quick fox')    -- frase → quick & fox
phraseto_tsquery('english', 'quick fox')   -- adjacente → quick <-> fox
websearch_to_tsquery('english', '"quick fox" -slow') -- estilo Google

-- Operador de correspondência
to_tsvector('english', title) @@ to_tsquery('english', 'database')

-- Ranking
ts_rank(vector, query)         -- pontuação de relevância baseada em frequência
ts_rank_cd(vector, query)      -- densidade de cobertura (considera proximidade)
ts_headline('english', text, query)  -- destaca termos correspondentes

-- Coluna gerada para indexação (PostgreSQL 12+)
ALTER TABLE articles ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (
    setweight(to_tsvector('english', coalesce(title,'')), 'A') ||
    setweight(to_tsvector('english', coalesce(body,'')), 'B')
  ) STORED;

CREATE INDEX idx_articles_fts ON articles USING GIN (search_vector);

-- Consulta
SELECT id, title, ts_rank(search_vector, query) AS rank
FROM articles, to_tsquery('english', 'postgres & index') AS query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

---

## Window Functions (Avançado)

```sql
-- Sintaxe: function() OVER (PARTITION BY ... ORDER BY ... ROWS/RANGE ...)

-- Ranking
ROW_NUMBER() OVER (PARTITION BY dept ORDER BY salary DESC)   -- rank único
RANK()       OVER (PARTITION BY dept ORDER BY salary DESC)   -- lacunas em empates
DENSE_RANK() OVER (PARTITION BY dept ORDER BY salary DESC)   -- sem lacunas
NTILE(4)     OVER (ORDER BY salary)                          -- quartil
PERCENT_RANK() OVER (ORDER BY salary)                        -- 0.0 a 1.0

-- Deslocamento
LAG(salary, 1, 0)  OVER (PARTITION BY dept ORDER BY hire_date)  -- linha anterior
LEAD(salary, 1, 0) OVER (PARTITION BY dept ORDER BY hire_date)  -- próxima linha
FIRST_VALUE(salary) OVER (PARTITION BY dept ORDER BY salary DESC)
LAST_VALUE(salary)  OVER (PARTITION BY dept ORDER BY salary DESC
                           ROWS BETWEEN UNBOUNDED PRECEDING AND UNBOUNDED FOLLOWING)

-- Agregações como window functions
SUM(amount) OVER (PARTITION BY user_id ORDER BY created_at)  -- total acumulado
AVG(amount) OVER (PARTITION BY user_id
                  ROWS BETWEEN 6 PRECEDING AND CURRENT ROW)  -- média móvel de 7 dias
COUNT(*) OVER (PARTITION BY category)                        -- tamanho do grupo

-- Windows nomeadas (DRY)
SELECT
  salary,
  AVG(salary) OVER w AS dept_avg,
  RANK()       OVER w AS dept_rank
FROM employees
WINDOW w AS (PARTITION BY department ORDER BY salary DESC);

-- Top-N por grupo (caso de uso clássico)
WITH ranked AS (
  SELECT *, ROW_NUMBER() OVER (PARTITION BY category ORDER BY created_at DESC) AS rn
  FROM products
)
SELECT * FROM ranked WHERE rn <= 3;
```

---

## Particionamento

```sql
-- Particionamento por intervalo de data
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

-- Partição padrão (captura qualquer coisa sem correspondência)
CREATE TABLE events_default PARTITION OF events DEFAULT;

-- Particionamento por lista de valores
CREATE TABLE orders (
  id      bigint,
  region  text,
  amount  numeric
) PARTITION BY LIST (region);

CREATE TABLE orders_us PARTITION OF orders FOR VALUES IN ('us-east', 'us-west');
CREATE TABLE orders_eu PARTITION OF orders FOR VALUES IN ('eu-west', 'eu-north');

-- Particionamento por hash para distribuição uniforme
CREATE TABLE sessions (id bigint, data jsonb) PARTITION BY HASH (id);
CREATE TABLE sessions_0 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 0);
CREATE TABLE sessions_1 PARTITION OF sessions FOR VALUES WITH (MODULUS 4, REMAINDER 1);
-- ... sessions_2, sessions_3

-- Índices em tabela particionada (cria em todas as partições)
CREATE INDEX ON events (user_id, created_at);

-- Anexar/desanexar partições (arquivamento sem downtime)
ALTER TABLE events DETACH PARTITION events_2022_q1;
ALTER TABLE events ATTACH PARTITION events_2025_q1
  FOR VALUES FROM ('2025-01-01') TO ('2025-04-01');
```

---

## Configuração de Replicação

```ini
# postgresql.conf — servidor primário
wal_level = replica            # mínimo para replicação
max_wal_senders = 5            # número de conexões de replicação
wal_keep_size = 256MB          # WAL retido para standbys
synchronous_commit = on        # 'off' para assíncrono (mais rápido, risco de perda de dados)
# synchronous_standby_names = 'standby1'  # para replicação síncrona

# pg_hba.conf — permitir conexões de replicação
host replication replicator 10.0.0.2/32 scram-sha-256

# Criar usuário de replicação
CREATE ROLE replicator WITH REPLICATION LOGIN PASSWORD 'secret';

# No standby: recovery.conf (PG < 12) ou postgresql.conf (PG >= 12)
primary_conninfo = 'host=10.0.0.1 user=replicator password=secret'
hot_standby = on               # permite queries de leitura no standby

# Replicação lógica (para replicação em nível de tabela, entre versões)
-- No publisher:
CREATE PUBLICATION my_pub FOR TABLE orders, products;
-- No subscriber:
CREATE SUBSCRIPTION my_sub
  CONNECTION 'host=publisher dbname=mydb user=replicator password=secret'
  PUBLICATION my_pub;
```

---

## Views pg_stat Úteis

```sql
-- Queries ativas e eventos de espera
SELECT pid, state, wait_event_type, wait_event, query, now() - query_start AS duration
FROM pg_stat_activity
WHERE state != 'idle'
ORDER BY duration DESC;

-- Matar query de longa duração
SELECT pg_cancel_backend(pid);   -- envia SIGINT (cancela query, mantém conexão)
SELECT pg_terminate_backend(pid); -- envia SIGTERM (mata conexão)

-- Estatísticas de acesso a tabelas (taxa de cache hit deve ser > 99%)
SELECT relname,
  heap_blks_read, heap_blks_hit,
  round(heap_blks_hit::numeric / nullif(heap_blks_hit + heap_blks_read, 0) * 100, 2) AS hit_ratio
FROM pg_statio_user_tables
ORDER BY heap_blks_read DESC;

-- Uso de índices
SELECT schemaname, tablename, indexname, idx_scan, idx_tup_read, idx_tup_fetch
FROM pg_stat_user_indexes
ORDER BY idx_scan DESC;

-- Índices não utilizados (candidatos a remoção)
SELECT schemaname, tablename, indexname
FROM pg_stat_user_indexes
WHERE idx_scan = 0
AND indexrelname NOT LIKE 'pg_toast%';

-- Bloat de tabela (tuplas mortas)
SELECT relname, n_live_tup, n_dead_tup,
  round(n_dead_tup::numeric / nullif(n_live_tup + n_dead_tup, 0) * 100, 2) AS dead_ratio,
  last_autovacuum, last_autoanalyze
FROM pg_stat_user_tables
ORDER BY dead_ratio DESC;

-- Esperas de lock
SELECT blocked.pid, blocked.query, blocking.pid AS blocking_pid, blocking.query AS blocking_query
FROM pg_stat_activity blocked
JOIN pg_stat_activity blocking ON blocking.pid = ANY(pg_blocking_pids(blocked.pid))
WHERE blocked.cardinality(pg_blocking_pids(blocked.pid)) > 0;

-- Tamanho do banco de dados
SELECT datname, pg_size_pretty(pg_database_size(datname)) FROM pg_database;

-- Tamanhos de tabela + índice
SELECT tablename,
  pg_size_pretty(pg_total_relation_size(tablename::regclass)) AS total,
  pg_size_pretty(pg_relation_size(tablename::regclass)) AS apenas_tabela,
  pg_size_pretty(pg_total_relation_size(tablename::regclass) - pg_relation_size(tablename::regclass)) AS indices
FROM pg_tables WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(tablename::regclass) DESC;
```

---

## VACUUM / ANALYZE

```sql
-- Vacuum manual (libera espaço de tuplas mortas, não devolve ao OS)
VACUUM table_name;

-- VACUUM FULL (reescreve tabela, devolve espaço ao OS, bloqueia tabela)
VACUUM FULL table_name;

-- Atualiza estatísticas para o planejador
ANALYZE table_name;

-- Combinado
VACUUM ANALYZE table_name;

-- Saída detalhada
VACUUM VERBOSE ANALYZE table_name;

-- Ajuste do autovacuum (por tabela)
ALTER TABLE high_churn_table SET (
  autovacuum_vacuum_scale_factor = 0.01,    -- vacuum quando 1% está morto (padrão 20%)
  autovacuum_analyze_scale_factor = 0.005,  -- analyze quando 0.5% mudou
  autovacuum_vacuum_cost_delay = 2          -- reduz throttling de I/O (ms)
);

-- Verificar se autovacuum está rodando
SHOW autovacuum;
SELECT * FROM pg_stat_activity WHERE query LIKE 'autovacuum%';
```

---

## Extensões Comuns

```sql
-- uuid-ossp: gerar UUIDs
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
SELECT uuid_generate_v4();          -- UUID aleatório
SELECT uuid_generate_v1mc();        -- baseado em tempo, preservando privacidade

-- pgcrypto: hashing e criptografia
CREATE EXTENSION IF NOT EXISTS pgcrypto;
SELECT crypt('password', gen_salt('bf', 12));      -- hash bcrypt
SELECT crypt('password', stored_hash) = stored_hash; -- verificar
SELECT encode(digest('data', 'sha256'), 'hex');    -- SHA-256

-- pg_trgm: busca fuzzy e similaridade
CREATE EXTENSION IF NOT EXISTS pg_trgm;
SELECT similarity('hello', 'helo');                -- 0.44
SELECT 'hello' % 'helo';                           -- verdadeiro se similaridade > threshold
CREATE INDEX idx_trgm ON articles USING GIN (title gin_trgm_ops);
-- Suporta queries LIKE/ILIKE no índice:
SELECT * FROM articles WHERE title ILIKE '%postgr%';

-- pg_stat_statements: rastreamento de performance de queries
CREATE EXTENSION IF NOT EXISTS pg_stat_statements;
-- Adicione ao postgresql.conf: shared_preload_libraries = 'pg_stat_statements'
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY total_exec_time DESC LIMIT 20;
pg_stat_statements_reset(); -- reinicia contadores

-- btree_gist: índices GiST em tipos escalares (para constraints de exclusão)
CREATE EXTENSION IF NOT EXISTS btree_gist;
-- Impede double-booking:
ALTER TABLE reservations ADD CONSTRAINT no_overlap
  EXCLUDE USING GIST (room_id WITH =, tstzrange(start_at, end_at) WITH &&);

-- hstore: chave-valor em uma coluna (prefira JSONB para código novo)
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

# Modo de pool: session | transaction | statement
# transaction = recomendado para apps sem estado
pool_mode = transaction

# Tamanhos de pool
default_pool_size = 25       # conexões por par db/user
max_client_conn = 1000       # total de conexões de clientes
reserve_pool_size = 5        # conexões extras para emergência
reserve_pool_timeout = 3     # segundos antes de usar pool de reserva

# Timeouts
server_idle_timeout = 600
client_idle_timeout = 0
query_timeout = 0

# Ressalvas do modo transaction — estas funcionalidades quebram com pooling por transaction:
# SET statements, advisory locks, LISTEN/NOTIFY, tabelas temporárias
# Use pooling por session se precisar delas
```

```bash
# Monitorar PgBouncer
psql -p 5432 -U pgbouncer pgbouncer -c "SHOW POOLS;"
psql -p 5432 -U pgbouncer pgbouncer -c "SHOW STATS;"
psql -p 5432 -U pgbouncer pgbouncer -c "SHOW CLIENTS;"
```

---

## Referência Rápida de Otimização de Queries

```sql
-- Planos de execução
EXPLAIN SELECT ...;                  -- plano estimado
EXPLAIN ANALYZE SELECT ...;          -- plano real + tempo (executa a query!)
EXPLAIN (ANALYZE, BUFFERS) SELECT ...; -- inclui info de cache hit

-- Forçar índice (para testes)
SET enable_seqscan = off;
-- Lembre de resetar: SET enable_seqscan = on;

-- Índice parcial (indexa subconjunto de linhas)
CREATE INDEX idx_active_users ON users (email) WHERE active = true;

-- Índice de expressão
CREATE INDEX idx_lower_email ON users (lower(email));
SELECT * FROM users WHERE lower(email) = 'alice@example.com'; -- usa índice

-- Índice de cobertura (INCLUDE — evita busca no heap para queries indexadas)
CREATE INDEX idx_orders_user ON orders (user_id) INCLUDE (total, created_at);
SELECT total, created_at FROM orders WHERE user_id = 1; -- index-only scan
```
