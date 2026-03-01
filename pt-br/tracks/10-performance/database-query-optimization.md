# Otimização de Queries no Banco de Dados

## Visão Geral

O banco de dados é o gargalo de performance mais comum em aplicações web. Queries lentas, indexes ausentes, padrões de query N+1 e joins ineficientes podem transformar uma API rápida em uma que dá timeout sob carga. Este capítulo cobre como diagnosticar queries lentas com EXPLAIN ANALYZE, como projetar indexes eficazes, como eliminar queries N+1 e os padrões de otimização mais comuns no Postgres.

---

## Pré-requisitos

- Fundamentos de SQL (SELECT, JOIN, WHERE, ORDER BY)
- Prisma ou outro ORM
- Track 08: Escalabilidade de Banco de Dados (para contexto mais amplo)

---

## Conceitos Fundamentais

### Como o Postgres executa queries

Quando você executa uma query, o query planner do Postgres avalia múltiplas estratégias de execução e escolhe a de menor custo estimado. As escolhas do planner — sequential scan vs. index scan, hash join vs. nested loop — determinam a performance. `EXPLAIN ANALYZE` mostra o plano escolhido e o tempo real de execução.

### Os três principais causadores de lentidão

1. **Indexes ausentes** — sequential scan em uma tabela grande
2. **Queries N+1** — uma query para buscar uma lista e depois uma query por item
3. **Queries não seletivas** — cláusulas WHERE que não reduzem o conjunto de resultados o suficiente

---

## Exemplos Práticos

### EXPLAIN ANALYZE

```sql
-- Encontrar queries lentas
SELECT query, calls, mean_exec_time, total_exec_time, rows
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Analisar uma query específica
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT o.*, u.name, u.email
FROM orders o
JOIN users u ON u.id = o.user_id
WHERE o.status = 'pending'
  AND o.created_at > NOW() - INTERVAL '7 days'
ORDER BY o.created_at DESC;
```

Exemplo de saída:

```
Sort  (cost=2847.23..2848.12 rows=356 width=248) (actual time=125.432..125.891 rows=356 loops=1)
  Sort Key: o.created_at DESC
  Sort Method: quicksort  Memory: 185kB
  ->  Hash Join  (cost=...) (actual time=45.123..124.012 rows=356 loops=1)
        ...
        ->  Seq Scan on orders o  (cost=0.00..8423.50 rows=356 width=...)
              Filter: (status = 'pending' AND created_at > ...)
              Rows Removed by Filter: 129644     ← 130K linhas varridas, 356 retornadas
Planning Time: 2.3 ms
Execution Time: 126.2 ms
```

O `Seq Scan` com `Rows Removed by Filter: 129644` indica um index ausente — o Postgres varreu todas as 130K linhas para encontrar 356.

### Criando indexes eficazes

```sql
-- Index de coluna única para queries de igualdade
CREATE INDEX CONCURRENTLY idx_orders_status ON orders (status);

-- Index composto para WHERE multi-coluna + ORDER BY
-- Colunas do WHERE primeiro, depois as do ORDER BY
CREATE INDEX CONCURRENTLY idx_orders_status_created
  ON orders (status, created_at DESC);

-- Index parcial — indexa apenas linhas que atendem uma condição
-- Se você só consulta orders pending/processing (não as históricas)
CREATE INDEX CONCURRENTLY idx_orders_active
  ON orders (created_at DESC)
  WHERE status IN ('pending', 'processing');

-- Index para pesquisas com LIKE de prefixo
CREATE INDEX CONCURRENTLY idx_users_email_prefix
  ON users (email text_pattern_ops);  -- habilita LIKE 'prefix%'

-- Index GIN para busca full-text
ALTER TABLE products ADD COLUMN search_vector tsvector
  GENERATED ALWAYS AS (to_tsvector('english', name || ' ' || description)) STORED;

CREATE INDEX idx_products_search ON products USING GIN (search_vector);
```

```typescript
// Equivalente no schema do Prisma
model Order {
  id        String   @id @default(cuid())
  status    String
  createdAt DateTime @default(now())
  userId    String

  // Index composto
  @@index([status, createdAt(sort: Desc)])
}
```

Após adicionar o index, execute novamente o EXPLAIN ANALYZE:

```
Index Scan using idx_orders_status_created on orders o
  (cost=0.43..12.34 rows=356 width=...) (actual time=0.05..2.34 rows=356 loops=1)
  Index Cond: (status = 'pending')
  Filter: (created_at > ...)
Execution Time: 3.1 ms   ← 40x mais rápido
```

### Eliminando queries N+1

O padrão N+1: uma query para buscar uma lista e depois uma query adicional por item.

**N+1 no Prisma (ruim):**

```typescript
// Isso faz 1 + N queries (1 para buscar orders, N para buscar cada usuário)
const orders = await db.order.findMany({ where: { status: 'pending' } });

for (const order of orders) {
  // N queries dentro do loop
  const user = await db.user.findUnique({ where: { id: order.userId } });
  console.log(`${user.name}: ${order.total}`);
}
```

**Corrigido com `include` (eager loading):**

```typescript
// 2 queries: uma para orders + um JOIN para users
const orders = await db.order.findMany({
  where: { status: 'pending' },
  include: { user: true },  // JOIN em uma única query ou query em batch
});

for (const order of orders) {
  console.log(`${order.user.name}: ${order.total}`); // nenhuma query adicional
}
```

**Detectando N+1 com logging do Prisma:**

```typescript
const db = new PrismaClient({
  log: [
    { level: 'query', emit: 'event' },
  ],
});

let queryCount = 0;
db.$on('query', () => { queryCount++; });

// Testa um route handler
const start = Date.now();
queryCount = 0;
await getOrdersWithUsers(userId);
console.log(`Queries: ${queryCount}, Tempo: ${Date.now() - start}ms`);
// Se queryCount > 3 para uma lista de 100 itens, você tem N+1
```

### Padrão DataLoader para batching

Quando você não pode usar `include` (ex: em um resolver GraphQL), use DataLoader para agrupar lookups individuais em uma única query:

```typescript
import DataLoader from 'dataloader';

const userLoader = new DataLoader<string, User | null>(
  async (userIds) => {
    const users = await db.user.findMany({
      where: { id: { in: userIds as string[] } },
    });

    // DataLoader exige resultados na mesma ordem das chaves
    const userMap = new Map(users.map((u) => [u.id, u]));
    return userIds.map((id) => userMap.get(id) ?? null);
  },
  { cache: true } // cache dentro de uma request
);

// Em resolvers — cada chamada a load() é agrupada em batch
async function resolveOrderUser(order: Order) {
  return userLoader.load(order.userId); // agrupado com todas as outras chamadas no mesmo tick
}
```

### Padrões de otimização de queries

**Selecione apenas as colunas necessárias:**

```typescript
// Ruim — busca todas as colunas incluindo blobs grandes
const users = await db.user.findMany();

// Bom — seleciona apenas o que será exibido
const users = await db.user.findMany({
  select: { id: true, name: true, email: true, role: true },
  // Sem passwordHash, sem profileJson (JSON grande), sem colunas não usadas
});
```

**Paginação — cursor-based vs offset:**

```typescript
// Paginação com offset — O(n) — Postgres varre todas as linhas anteriores
const page3 = await db.product.findMany({
  skip: 200,  // varre 200 linhas apenas para pulá-las
  take: 20,
  orderBy: { createdAt: 'desc' },
});

// Paginação com cursor — O(1) — usa o index diretamente
const afterId = req.query.cursor; // ID do último item da página anterior

const nextPage = await db.product.findMany({
  take: 20,
  skip: afterId ? 1 : 0,
  cursor: afterId ? { id: afterId } : undefined,
  orderBy: { createdAt: 'desc' },
});
```

**Contando de forma eficiente:**

```typescript
// Ruim — busca todas as linhas para contar
const total = (await db.order.findMany({ where: { userId } })).length;

// Bom — query COUNT
const total = await db.order.count({ where: { userId } });

// Bom — busca lista e contagem em paralelo
const [orders, total] = await Promise.all([
  db.order.findMany({ where: { userId }, take: 20, skip: offset }),
  db.order.count({ where: { userId } }),
]);
```

---

## Padrões e Boas Práticas

- **Indexe toda foreign key** — o Postgres não as cria automaticamente
- **Use indexes compostos que correspondam ao seu padrão WHERE + ORDER BY**
- **Crie indexes com CONCURRENTLY** — não bloqueia a tabela durante a criação do index
- **Monitore o log de queries lentas** — configure `log_min_duration_statement = 100` no Postgres (loga queries > 100ms)
- **Use `select` para buscar apenas as colunas necessárias** — evite especialmente campos JSON ou binários grandes
- **Paginação com cursor** para datasets grandes — o offset degrada a partir da página 1000+

---

## Anti-Padrões a Evitar

- Queries N+1 — sempre verifique quantas queries uma request faz
- Buscar todas as linhas para contá-las
- `SELECT *` — busca dados desnecessários, pode incluir colunas grandes
- Indexes em colunas de baixa cardinalidade (status com 3 valores) sem index parcial — o planner pode ignorá-los
- Pular o `EXPLAIN ANALYZE` — otimizar sem profiling é chute

---

## Debugging e Resolução de Problemas

**"O EXPLAIN mostra index scan mas ainda está lento"**
O index existe, mas o volume de dados retornado é grande. Verifique `actual rows` — se 50K linhas são retornadas após um index scan, você pode precisar de uma cláusula WHERE mais seletiva ou de um covering index.

**"Adicionar um index piorou as coisas"**
Indexes têm custo de escrita. Em tabelas com muita escrita, muitos indexes tornam INSERT/UPDATE mais lentos. Remova indexes não usados com `SELECT schemaname, tablename, indexname, idx_scan FROM pg_stat_user_indexes WHERE idx_scan = 0`.

**"O Prisma está fazendo queries extras inesperadas"**
Use `log: ['query']` para ver todas as queries. O Prisma às vezes usa uma query separada para contar relações quando você usa `include` com paginação. Use queries raw ou ajuste a estrutura da query.

---

## Leituras Complementares

- [Use the Index, Luke — guia de indexação SQL](https://use-the-index-luke.com/)
- [Documentação do PostgreSQL sobre EXPLAIN](https://www.postgresql.org/docs/current/sql-explain.html)
- [Prisma: otimização de queries](https://www.prisma.io/docs/guides/performance-and-optimization/query-optimization-performance)
- [DataLoader](https://github.com/graphql/dataloader)
- Track 08: [Escalabilidade de Banco de Dados](../08-system-design/database-scaling.md)

---

## Resumo

A otimização de queries segue um processo simples: identifique queries lentas com `pg_stat_statements`, analise-as com `EXPLAIN ANALYZE`, adicione indexes direcionados e elimine padrões N+1. Os maiores ganhos vêm de: adicionar indexes compostos que correspondam aos seus padrões WHERE + ORDER BY, usar `include` ou DataLoader em vez de queries por item e selecionar apenas as colunas necessárias. A paginação com cursor substitui a paginação com offset em datasets grandes. Sempre meça antes e depois de cada mudança com `EXPLAIN ANALYZE` para confirmar a melhoria.
