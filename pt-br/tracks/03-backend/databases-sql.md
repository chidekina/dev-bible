# Bancos de Dados: SQL

## 1. O Que é e Por Que Usar

Bancos de dados relacionais dominaram o armazenamento de dados por mais de 40 anos porque oferecem uma combinação poderosa: uma linguagem de consulta flexível (SQL), garantias fortes de consistência (ACID) e um ecossistema maduro. O PostgreSQL em particular cresceu para se tornar a escolha padrão para a maioria dos aplicativos backend porque combina SQL com recursos avançados — JSONB, window functions, full-text search e extensibilidade — que eliminam a necessidade de muitos bancos de dados especializados.

Entender SQL profundamente — não apenas CRUD, mas índices, planos de execução, transações e níveis de isolamento — separa engenheiros que constroem sistemas que funcionam em produção daqueles que produzem demos que quebram sob carga.

---

## 2. Conceitos Fundamentais

### Propriedades ACID

ACID define as garantias que uma transação de banco de dados deve fornecer:

**Atomicidade** — Uma transação é tudo ou nada. Se qualquer etapa falhar, a transação inteira é revertida. Nenhum estado parcial é confirmado no banco.

```sql
BEGIN;
UPDATE accounts SET balance = balance - 100 WHERE id = 1;
UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- Se o segundo UPDATE falhar, o primeiro também é revertido
COMMIT;
```

**Consistência** — Uma transação leva o banco de um estado válido para outro. Restrições (foreign keys, check constraints, unique constraints) são aplicadas. Uma transação que viola a consistência é abortada.

**Isolamento** — Transações concorrentes parecem executar serialmente. Cada transação trabalha com um snapshot consistente dos dados. O grau de isolamento é configurável (veja Níveis de Isolamento).

**Durabilidade** — Uma vez confirmados, os dados sobrevivem a crashes. O PostgreSQL alcança isso com Write-Ahead Logging (WAL): as mudanças são escritas em um arquivo de log WAL antes de serem aplicadas aos arquivos de dados. Em caso de crash, o WAL é reproduzido na reinicialização.

---

### Normalização

A normalização reduz redundância de dados e melhora a integridade organizando tabelas seguindo formas normais.

**1FN (Primeira Forma Normal):**
- Cada coluna contém valores atômicos (indivisíveis).
- Sem grupos repetidos ou arrays em colunas.
- Cada linha é unicamente identificável.

```sql
-- Viola 1FN: múltiplos valores em uma coluna
CREATE TABLE orders (
  id INT,
  customer_name TEXT,
  items TEXT  -- "apple, banana, cherry" ← NÃO atômico
);

-- 1FN compatível: linhas separadas por item
CREATE TABLE order_items (
  order_id INT,
  item_name TEXT,
  quantity INT
);
```

**2FN (Segunda Forma Normal):** Deve estar em 1FN + sem dependências parciais em uma chave primária composta. Toda coluna não-chave deve depender da CHAVE INTEIRA.

```sql
-- Viola 2FN: customer_name depende apenas de customer_id, não de (customer_id, product_id)
CREATE TABLE order_products (
  customer_id INT,
  product_id INT,
  customer_name TEXT,   -- ← dependência parcial em customer_id apenas
  quantity INT,
  PRIMARY KEY (customer_id, product_id)
);

-- 2FN compatível: extrai customer para sua própria tabela
CREATE TABLE customers (id INT PRIMARY KEY, name TEXT);
CREATE TABLE order_products (
  customer_id INT REFERENCES customers(id),
  product_id INT,
  quantity INT,
  PRIMARY KEY (customer_id, product_id)
);
```

**3FN (Terceira Forma Normal):** Deve estar em 2FN + sem dependências transitivas. Colunas não-chave não devem depender de outras colunas não-chave.

```sql
-- Viola 3FN: zip_code determina city (dependência transitiva)
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name TEXT,
  zip_code TEXT,
  city TEXT  -- ← depende de zip_code, não de id
);

-- 3FN compatível
CREATE TABLE zip_codes (zip TEXT PRIMARY KEY, city TEXT);
CREATE TABLE employees (
  id INT PRIMARY KEY,
  name TEXT,
  zip_code TEXT REFERENCES zip_codes(zip)
);
```

**Quando desnormalizar:**
- Queries de relatório com muitos JOINs que são read-heavy — o overhead de JOIN pode superar o custo da redundância.
- Agregados pré-computados (ex.: coluna `post_count` na tabela de usuários) para contadores acessados frequentemente.
- Workloads de analytics/OLAP onde leituras dominam e anomalias de atualização são aceitáveis.
- Quando você adiciona desnormalização, documente explicitamente e adicione triggers ou código de aplicação para manter a consistência.

---

### Índices

Um índice é uma estrutura de dados separada que permite ao banco encontrar linhas sem escanear a tabela inteira.

**B-tree (padrão):**
- Árvore balanceada, lookup O(log n).
- Suporta: igualdade (`=`), queries de intervalo (`<`, `>`, `BETWEEN`), `LIKE 'prefixo%'`, `ORDER BY`, `IS NULL`.
- Padrão para a maioria das colunas.

**Hash:**
- Lookup O(1) apenas para igualdade exata.
- NÃO suporta queries de intervalo ou ordenação.
- Raramente escolhido sobre B-tree na prática (B-tree trata igualdade também e suporta mais tipos de query).

**GIN (Generalized Inverted Index):**
- Para containment de arrays, operadores JSONB, full-text search.
- Armazena um mapeamento de elementos/tokens individuais para as linhas que os contêm.

**Índice parcial:**
- Indexa apenas linhas que correspondem a uma condição WHERE.
- Menor que um índice completo; mais eficaz quando a maioria das queries foca em um subconjunto.

```sql
-- Índice parcial: maioria das queries acessa apenas registros não deletados
CREATE INDEX idx_users_email_active
ON users(email)
WHERE deleted_at IS NULL;

-- Índice full-text
CREATE INDEX idx_posts_search ON posts USING GIN(to_tsvector('english', title || ' ' || body));

-- Índice JSONB: permite queries de containment @> rápidas
CREATE INDEX idx_events_metadata ON events USING GIN(metadata);

-- Índice covering: inclui colunas extras para evitar ir ao heap
-- Query: SELECT name, email FROM users WHERE status = 'active'
CREATE INDEX idx_users_status_covering ON users(status) INCLUDE (name, email);
-- Com INCLUDE, o índice contém name e email — a query pode ser satisfeita do
-- índice sozinho (Index Only Scan), sem necessidade de fetch do heap
```

**A ordem das colunas em índice composto importa:**

```sql
-- Índice em (status, created_at)
CREATE INDEX idx_orders ON orders(status, created_at DESC);

-- Usa o índice: status é a coluna líder
SELECT * FROM orders WHERE status = 'pending' ORDER BY created_at DESC;

-- Usa o índice (coluna líder presente)
SELECT * FROM orders WHERE status = 'pending';

-- NÃO usa o índice eficientemente: coluna líder ausente
SELECT * FROM orders WHERE created_at > '2024-01-01';
-- Usaria idx_orders(created_at) se existisse separadamente

-- Regra: o prefixo mais à esquerda do índice composto deve estar na query
```

> Indexe a coluna mais seletiva primeiro em um índice composto para máxima eficácia de filtro. Mas se você quase sempre filtra por uma coluna específica, coloque-a primeiro independentemente — o otimizador usará o índice para essa coluna mesmo que não seja a mais seletiva.

> Índices têm custos: cada escrita (INSERT/UPDATE/DELETE) também deve atualizar todos os índices relevantes. Uma tabela com 15 índices terá escritas mais lentas. Faça profiling antes de adicionar índices especulativamente.

---

### EXPLAIN ANALYZE

A ferramenta mais importante para entender performance de query:

```sql
EXPLAIN ANALYZE
SELECT u.name, COUNT(p.id) as post_count
FROM users u
LEFT JOIN posts p ON p.author_id = u.id
WHERE u.status = 'active'
GROUP BY u.id, u.name
ORDER BY post_count DESC
LIMIT 10;

-- Saída de exemplo:
-- Limit  (cost=234.50..234.53 rows=10 width=36) (actual time=12.543..12.545 rows=10 loops=1)
--   ->  Sort  (cost=234.50..236.50 rows=800 width=36) (actual time=12.541..12.542 rows=10 loops=1)
--         Sort Key: (count(p.id)) DESC
--         ->  HashAggregate  (cost=192.00..200.00 rows=800 width=36) (actual time=12.300..12.410 rows=812 loops=1)
--               Group Key: u.id
--               ->  Hash Left Join  (cost=45.00..176.00 rows=3200 width=20) (actual time=1.234..10.456 rows=3200 loops=1)
--                     Hash Cond: (p.author_id = u.id)
--                     ->  Seq Scan on posts p  (cost=0.00..64.00 rows=3200 width=8)
--                     ->  Hash  (cost=35.00..35.00 rows=800 width=20)
--                           ->  Index Scan using idx_users_status on users u
-- Planning Time: 0.456 ms
-- Execution Time: 12.678 ms
```

**Lendo a saída:**
- `cost=X..Y`: custo estimado de startup..total (unidades arbitrárias).
- `actual time=X..Y`: milissegundos reais de startup..fim.
- `rows`: linhas estimadas (das estatísticas) vs. linhas reais.
- Grande discrepância entre estimado e real = estatísticas desatualizadas → execute `ANALYZE`.
- **Seq Scan**: lê cada linha da tabela — esperado para tabelas pequenas ou queries de alta seletividade; problemático para tabelas grandes com predicados de baixa seletividade.
- **Index Scan**: busca linhas via índice, faz fetch do heap. Bom.
- **Index Only Scan**: todas as colunas necessárias estão no índice — sem fetch do heap. Melhor para performance de leitura.
- **Bitmap Index Scan**: coleta IDs de linha do índice, depois faz fetch do heap em ordem de página. Usado quando muitas linhas correspondem — mais eficiente que acessos aleatórios repetidos ao heap.

---

### Níveis de Isolamento de Transação

| Nível | Dirty Read | Non-Repeatable Read | Phantom Read |
|-------|------------|---------------------|--------------|
| Read Uncommitted | Possível | Possível | Possível |
| Read Committed | Prevenido | Possível | Possível |
| Repeatable Read | Prevenido | Prevenido | Possível* |
| Serializable | Prevenido | Prevenido | Prevenido |

*O Repeatable Read baseado em MVCC do PostgreSQL também previne phantom reads na maioria dos casos.

**Padrão do PostgreSQL: Read Committed.**

**Fenômenos:**
- **Dirty read**: ler mudanças não confirmadas de outra transação (dados que podem ser revertidos).
- **Non-repeatable read**: ler a mesma linha duas vezes dentro de uma transação retorna valores diferentes (outra transação confirmou uma atualização entre as duas leituras).
- **Phantom read**: uma query de intervalo retorna conjuntos diferentes de linhas em duas execuções dentro da mesma transação (outra transação inseriu/deletou linhas).

```sql
-- Isolamento Serializable: previne todas as anomalias (mais restritivo)
BEGIN ISOLATION LEVEL SERIALIZABLE;
-- ... operações complexas de múltiplas leituras + escritas que devem ser atômicas
COMMIT;

-- No PostgreSQL, SERIALIZABLE usa SSI (Serializable Snapshot Isolation)
-- que tem muito menos overhead que o locking tradicional
-- mas pode gerar falhas de serialização (requer retry)
```

---

## 3. Como Funciona — Padrões Comuns de SQL

### Soft Delete

```sql
ALTER TABLE users ADD COLUMN deleted_at TIMESTAMPTZ;

-- Soft delete
UPDATE users SET deleted_at = NOW() WHERE id = $1;

-- Queries excluem registros soft-deleted
SELECT * FROM users WHERE deleted_at IS NULL;

-- Índice parcial para performance
CREATE INDEX idx_users_active ON users(email) WHERE deleted_at IS NULL;

-- Restaurar
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

-- Função de trigger
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

-- Anexa às tabelas
CREATE TRIGGER users_audit AFTER INSERT OR UPDATE OR DELETE ON users
  FOR EACH ROW EXECUTE FUNCTION audit_trigger();
```

### Cursor Pagination

```sql
-- Primeira página (sem cursor)
SELECT id, title, created_at
FROM posts
WHERE status = 'published'
ORDER BY created_at DESC, id DESC
LIMIT 21; -- busca 21, se 21 retornados → hasMore = true

-- Páginas subsequentes usando cursor (created_at + id do último item)
SELECT id, title, created_at
FROM posts
WHERE status = 'published'
  AND (created_at, id) < ($last_created_at, $last_id)
ORDER BY created_at DESC, id DESC
LIMIT 21;

-- Índice para suportar esta query
CREATE INDEX idx_posts_cursor ON posts(status, created_at DESC, id DESC)
WHERE status = 'published';
```

### Optimistic Locking

```sql
ALTER TABLE documents ADD COLUMN version INT NOT NULL DEFAULT 1;

-- Leitura
SELECT id, content, version FROM documents WHERE id = $1;

-- Atualização: inclui version na cláusula WHERE
UPDATE documents
SET content = $new_content, version = version + 1
WHERE id = $1 AND version = $expected_version;

-- Na aplicação: verifica linhas afetadas
-- Se rowCount = 0 → versão incompatível → outro processo modificou o registro → tente novamente
```

---

## 4. Exemplos de Código (TypeScript / PostgreSQL)

### Recursos Específicos do PostgreSQL

```typescript
import { Pool } from 'pg';

const pool = new Pool({ connectionString: process.env.DATABASE_URL });

// Operadores JSONB
// -> retorna valor JSON, ->> retorna texto
// @> operador de containment: o JSONB da esquerda contém o da direita?
const result = await pool.query(`
  SELECT id, metadata->>'plan' as plan_name, metadata->'features' as features
  FROM subscriptions
  WHERE metadata @> '{"status": "active"}'
    AND metadata->>'plan' IN ('pro', 'enterprise')
`);

// UPSERT com ON CONFLICT
await pool.query(`
  INSERT INTO user_preferences (user_id, key, value)
  VALUES ($1, $2, $3)
  ON CONFLICT (user_id, key)
  DO UPDATE SET value = EXCLUDED.value, updated_at = NOW()
  RETURNING *
`, [userId, 'theme', 'dark']);

// CTE (Common Table Expression) — subqueries legíveis e reutilizáveis
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

// RETURNING: recebe as linhas inseridas/atualizadas de volta
const newPost = await pool.query(`
  INSERT INTO posts (title, body, author_id, slug)
  VALUES ($1, $2, $3, $4)
  RETURNING id, title, slug, created_at
`, [title, body, authorId, slug]);
```

### Transações em Node.js

```typescript
async function transferFunds(
  fromId: string,
  toId: string,
  amount: number
): Promise<void> {
  const client = await pool.connect();
  try {
    await client.query('BEGIN');

    // Trava ambas as linhas em ordem consistente (menor id primeiro) para prevenir deadlocks
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
    client.release(); // sempre libera de volta para o pool
  }
}
```

### Connection Pooling com PgBouncer

```
Modos de pooling do PgBouncer:

Session mode:
  - Uma conexão de servidor por conexão de cliente durante toda a sessão
  - Menos restritivo; suporta todos os recursos do PostgreSQL incluindo LISTEN/NOTIFY
  - Economia mínima de conexões

Transaction mode (mais comum):
  - Conexão de servidor atribuída apenas pela duração de uma transação
  - Não pode usar LISTEN/NOTIFY, SET configuration, advisory locks fora de transações
  - 10x-50x mais conexões suportadas por conexão de servidor
  - Melhor para APIs web típicas usando transações curtas

Statement mode:
  - Conexão retornada após cada statement individual
  - Não pode usar transações multi-statement
  - Raramente usado
```

```
# pgbouncer.ini
[databases]
myapp = host=postgres port=5432 dbname=myapp

[pgbouncer]
pool_mode = transaction
max_client_conn = 1000    # máximo de conexões dos apps
default_pool_size = 20    # conexões ao PostgreSQL
min_pool_size = 5
server_idle_timeout = 60
```

---

## 5. Erros Comuns e Armadilhas

**Índices faltando em foreign keys:**

```sql
-- PROBLEMA: toda query em posts que faz JOIN com users fará Seq Scan em posts
-- a não ser que haja um índice na foreign key
CREATE TABLE posts (
  id UUID PRIMARY KEY,
  author_id UUID REFERENCES users(id)
  -- Sem índice em author_id!
);

-- CORREÇÃO
CREATE INDEX idx_posts_author_id ON posts(author_id);
-- O PostgreSQL NÃO indexa automaticamente colunas de foreign key
```

**Queries N+1 no código da aplicação:**

```typescript
// PROBLEMA: 1 query para posts + 1 query por post para autor = N+1
const posts = await db.query('SELECT * FROM posts LIMIT 100');
for (const post of posts) {
  const author = await db.query('SELECT * FROM users WHERE id = $1', [post.author_id]);
}

// CORREÇÃO: JOIN no SQL
const posts = await db.query(`
  SELECT p.*, u.name as author_name, u.avatar_url
  FROM posts p
  JOIN users u ON u.id = p.author_id
  LIMIT 100
`);
```

**Usar OFFSET para páginas grandes:**

```sql
-- LENTO: o banco escaneia e descarta 1.000.000 linhas antes de retornar 20
SELECT * FROM events ORDER BY created_at DESC OFFSET 1000000 LIMIT 20;

-- RÁPIDO: baseado em cursor, usa o índice eficientemente
SELECT * FROM events
WHERE created_at < $cursor_timestamp
ORDER BY created_at DESC
LIMIT 20;
```

**Não tratar falhas de serialização:**

```typescript
// Transações SERIALIZABLE podem falhar com:
// ERROR: could not serialize access due to concurrent update
// Aplicações DEVEM fazer retry neste erro

async function withRetry<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err: any) {
      if (err.code === '40001' && attempt < maxRetries) {
        // 40001 = serialization_failure — seguro para retry
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

## 6. Quando Usar / Não Usar

**Use PostgreSQL (SQL) quando:**
- Relacionamentos complexos entre entidades com integridade referencial
- Transações ACID são necessárias (financeiro, médico, inventário)
- Queries de relatório e analytics ad-hoc
- O modelo de dados é relativamente estável e bem compreendido
- Você precisa de JOINs, agregações, window functions

**Considere NoSQL quando:**
- Schema é verdadeiramente desconhecido ou muda muito frequentemente
- Dados centrados em documentos sem relacionamentos (catálogos de produtos com atributos variáveis)
- Throughput de escrita extremo com padrões de acesso simples (time-series, logs)
- Distribuição global com leituras de baixa latência entre regiões

---

## 7. Cenário Real

**Problema:** A página de listagem de pedidos de uma plataforma de e-commerce demora 12 segundos para carregar para lojistas ativos com 500.000+ pedidos. A query usa paginação `OFFSET/LIMIT`.

**Investigação:**

```sql
EXPLAIN ANALYZE
SELECT o.*, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.merchant_id = $1
  AND o.status = 'completed'
ORDER BY o.created_at DESC
OFFSET 490000 LIMIT 20;

-- Saída mostra: Seq Scan em orders, actual time=11834ms
```

**Causas raiz:**
1. `OFFSET 490000` força o PostgreSQL a escanear 490.020 linhas e descartar a maioria.
2. Sem índice em `(merchant_id, status, created_at)`.

**Correção:**

```sql
-- 1. Adiciona índice composto
CREATE INDEX idx_orders_merchant_status_cursor
ON orders(merchant_id, status, created_at DESC)
WHERE status = 'completed'; -- índice parcial para o caso mais comum

-- 2. Muda para paginação por cursor
SELECT o.id, o.total, o.created_at, c.name as customer_name
FROM orders o
JOIN customers c ON c.id = o.customer_id
WHERE o.merchant_id = $1
  AND o.status = 'completed'
  AND o.created_at < $cursor_timestamp  -- cursor do último item da página anterior
ORDER BY o.created_at DESC
LIMIT 21; -- 21 para detectar hasMore
```

**Resultado:** Tempo de query cai de 12 segundos para 4ms. O índice é utilizado, e o cursor substitui o scan caro de OFFSET.

---

## 8. Perguntas de Entrevista

**Q1: O que é um índice e como um índice B-tree funciona internamente?**

Um índice é uma estrutura de dados separada que armazena um subconjunto ordenado de colunas da tabela com ponteiros para as linhas reais da tabela (heap tuples). Um B-tree é uma árvore auto-balanceada onde cada nó contém múltiplos pares chave-ponteiro. Lookups começam na raiz, seguem ponteiros baseados em comparações de chaves e chegam a um nó folha em O(log n) passos. Nós folha são ligados sequencialmente, habilitando scans de intervalo eficientes. O B-tree do PostgreSQL suporta igualdade, desigualdade, queries de intervalo e ordenação. Escritas devem atualizar a árvore, por isso muitos índices tornam as escritas mais lentas.

**Q2: Explique as quatro propriedades ACID com um exemplo.**

Uma transferência bancária: (1) **Atomicidade** — debitar $100 da conta A e creditar $100 na conta B são uma unidade única — se o crédito falhar, o débito é revertido. (2) **Consistência** — o total de dinheiro no sistema é o mesmo antes e depois (restrição: soma de todos os saldos = constante). (3) **Isolamento** — duas transferências concorrentes não veem estados intermediários uma da outra — uma transferência vê o estado pré-transferência, a outra vê o estado pós-transferência. (4) **Durabilidade** — após COMMIT, a transferência sobrevive a uma queda de energia; WAL garante que pode ser reproduzida na recuperação.

**Q3: Quando você usaria um covering index?**

Quando uma query pode ser satisfeita inteiramente pelo índice sem fazer fetch das linhas reais da tabela (Index Only Scan). Por exemplo: `SELECT name, email FROM users WHERE status = 'active'` — se o índice é `CREATE INDEX ON users(status) INCLUDE (name, email)`, o PostgreSQL pode retornar `name` e `email` diretamente do índice sem um fetch do heap. Isso é especialmente valioso para queries de leitura de alta frequência em tabelas grandes onde o acesso ao heap é o gargalo.

**Q4: Qual nível de isolamento o PostgreSQL usa por padrão e quais fenômenos ele previne?**

O PostgreSQL tem como padrão **Read Committed**. Ele previne dirty reads (ler dados não confirmados de outra transação) mas permite non-repeatable reads (a mesma linha pode retornar valores diferentes dentro de uma transação se outra transação confirmar entre as leituras) e phantom reads (uma query de intervalo pode retornar linhas diferentes). Para a maioria dos aplicativos web, Read Committed é suficiente. Use Repeatable Read ou Serializable para operações financeiras onde consistência através de múltiplas leituras dentro de uma transação é necessária.

**Q5: Qual é a diferença entre dirty read e phantom read?**

Um **dirty read** é ler dados não confirmados de outra transação — você pode ler dados que depois são revertidos. Prevenido em Read Committed e acima. Um **phantom read** é quando uma query que lê um intervalo retorna linhas diferentes em duas execuções dentro da mesma transação porque outra transação inseriu ou deletou linhas naquele intervalo. Exemplo: COUNT(*) WHERE status='pending' retorna 5, depois 7 numa segunda leitura. Prevenido apenas em Serializable (e principalmente em Repeatable Read no PostgreSQL devido ao MVCC).

**Q6: O que é uma CTE e quando você usaria uma?**

Uma CTE (Common Table Expression), escrita com a palavra-chave `WITH`, é um resultado temporário nomeado dentro de uma query. Casos de uso: (1) **Legibilidade** — quebra queries complexas em etapas lógicas nomeadas. (2) **Queries recursivas** — `WITH RECURSIVE` para travessia de árvore/grafo (ex.: hierarquias organizacionais, lista de materiais). (3) **Transformações em múltiplas etapas** — encadeia múltiplos result sets em uma query. No PostgreSQL, CTEs não recursivas podem ser inlineadas na query principal pelo otimizador; use `MATERIALIZED` para forçá-las a serem computadas uma vez.

**Q7: Explique window functions com um exemplo.**

Window functions computam um valor para cada linha usando uma "janela" de linhas relacionadas, sem colapsá-las em uma única linha de agregação. `SUM(score) OVER ()` computa o score total para todas as linhas e o anexa a cada linha. `ROW_NUMBER() OVER (PARTITION BY department ORDER BY salary DESC)` atribui um rank dentro de cada departamento por salário. Funções principais: `ROW_NUMBER`, `RANK`, `DENSE_RANK`, `LAG`, `LEAD`, `FIRST_VALUE`, `LAST_VALUE`, `SUM/AVG/COUNT OVER`. Essenciais para queries de analytics: totais acumulados, percentagens do total, comparações de valor anterior/próximo.

**Q8: Por que a paginação por cursor é melhor que a paginação por offset para grandes datasets?**

Paginação por offset (`OFFSET 10000 LIMIT 20`) requer que o banco escaneie e descarte 10.000 linhas antes de retornar 20 — esse custo cresce linearmente com o número da página. Paginação por cursor usa uma cláusula WHERE que aproveita um índice diretamente: `WHERE created_at < $cursor ORDER BY created_at DESC LIMIT 20` — isso é O(log n) independente de qual página você está. Adicionalmente, a paginação por offset dá resultados inconsistentes quando linhas são inseridas ou deletadas entre carregamentos de página (linhas se deslocam), enquanto a paginação por cursor é estável porque usa posição absoluta.

---

## 9. Exercícios

**Exercício 1: Auditoria de índices**

Dada uma tabela `events(id, user_id, type, payload JSONB, created_at)` com 50 milhões de linhas, escreva os índices ótimos para estes padrões de query:
- `WHERE user_id = $1 AND type = 'click' ORDER BY created_at DESC LIMIT 20`
- `WHERE payload @> '{"campaign_id": "X"}' AND created_at > NOW() - INTERVAL '7 days'`
- `WHERE user_id = $1` com SELECT apenas `id, type, created_at` (sem payload)

Explique por que você escolheu cada tipo de índice e ordem de coluna.

*Dica:* Considere índices compostos, parciais, GIN e covering. Execute `EXPLAIN ANALYZE` em cada padrão.

**Exercício 2: Relatório com window function**

Escreva uma única query SQL (usando CTEs e window functions) que retorna, para cada mês dos últimos 12 meses:
- Receita total
- Número de pedidos
- Crescimento de receita vs mês anterior (como percentagem)
- Receita acumulada no ano
- Receita deste mês como percentagem do total do ano

*Dica:* Use `LAG()` para o mês anterior, `SUM() OVER (PARTITION BY year ORDER BY month)` para YTD.

**Exercício 3: Implementação de optimistic locking**

Implemente um sistema de edição de artigos onde:
- Múltiplos usuários podem editar o mesmo artigo concorrentemente
- Se dois usuários tentam salvar ao mesmo tempo, o segundo salvo falha com um erro de conflito
- O cliente deve reenviar com a versão mais recente para prosseguir
- Escreva o SQL e a camada de service em TypeScript

*Dica:* Adicione coluna `version INT DEFAULT 1`. `UPDATE ... WHERE id = $1 AND version = $expected; RETURNING *` — se 0 linhas retornadas, conflito de versão.

---

## 10. Leitura Complementar

- [Documentação PostgreSQL — EXPLAIN](https://www.postgresql.org/docs/current/using-explain.html)
- [Documentação PostgreSQL — Índices](https://www.postgresql.org/docs/current/indexes.html)
- [Documentação PostgreSQL — Isolamento de transação](https://www.postgresql.org/docs/current/transaction-iso.html)
- [Use the Index, Luke — indexação para desenvolvedores](https://use-the-index-luke.com/)
- [pganalyze — index advisor](https://pganalyze.com/)
- [Documentação do PgBouncer](https://www.pgbouncer.org/config.html)
- [Visualizador de EXPLAIN do Postgres (explain.depesz.com)](https://explain.depesz.com/)
- Livro: *PostgreSQL: Up and Running* por Regina Obe & Leo Hsu
- Livro: *The Art of PostgreSQL* por Dimitri Fontaine
