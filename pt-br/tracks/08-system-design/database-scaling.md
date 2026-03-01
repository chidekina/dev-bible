# Database Scaling

## Visão Geral

À medida que uma aplicação cresce, o banco de dados frequentemente se torna o gargalo. Uma única instância Postgres lida com milhares de queries por segundo em condições normais, mas workloads com muitas escritas, queries analíticas e alta concorrência podem sobrecarregar um único nó. Este capítulo cobre as principais estratégias para escalar bancos de dados: read replicas, connection pooling, sharding e escalonamento vertical — e quando usar cada um.

---

## Pré-requisitos

- Fundamentos de SQL (queries, índices, transações)
- Entendimento básico de conceitos de rede
- Track 03 — Backend: bancos de dados e uso de Prisma/ORM

---

## Conceitos Fundamentais

### Dimensões de escalonamento

**Escalonamento vertical (scale up):** Dê ao banco de dados mais CPU, RAM e armazenamento mais rápido. Simples, sem alterações na aplicação, mas tem limites rígidos.

**Escalonamento horizontal (scale out):** Adicione mais nós de banco de dados. Requer alterações na aplicação e introduz complexidade de sistemas distribuídos, mas não tem teto rígido.

### Escalonamento de leituras vs escritas

A maioria das aplicações web tem muito mais leituras do que escritas (80–95% leituras). Isso significa que frequentemente você pode escalar dramaticamente distribuindo leituras entre múltiplos nós, mesmo que as escritas ainda vão para um único primary.

---

## Exemplos Práticos

### Read replicas com Prisma

```typescript
// src/lib/db.ts
import { PrismaClient } from '@prisma/client';

// Primary — todas as escritas, leituras críticas
export const primaryDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_PRIMARY_URL } },
  log: ['warn', 'error'],
});

// Read replica — volume de leituras
export const replicaDb = new PrismaClient({
  datasources: { db: { url: process.env.DATABASE_REPLICA_URL } },
  log: ['warn', 'error'],
});

// Conveniência: a maioria das leituras deve usar este
export const db = replicaDb;
```

```typescript
// src/services/user.service.ts
import { primaryDb, replicaDb } from '../lib/db.js';

export class UserService {
  // Escritas sempre vão para o primary
  async createUser(data: CreateUserDto) {
    return primaryDb.user.create({ data });
  }

  async updateUser(id: string, data: UpdateUserDto) {
    return primaryDb.user.update({ where: { id }, data });
  }

  // Leituras não críticas podem usar a replica
  async listUsers(page: number, limit: number) {
    return replicaDb.user.findMany({
      skip: (page - 1) * limit,
      take: limit,
      orderBy: { createdAt: 'desc' },
    });
  }

  // Leituras críticas (após uma escrita) vão para o primary
  async getUserById(id: string, mustBeFresh = false) {
    const client = mustBeFresh ? primaryDb : replicaDb;
    return client.user.findUnique({ where: { id } });
  }
}
```

### Connection pooling com PgBouncer

O PostgreSQL tem um limite rígido de conexões (tipicamente 100–500). Cada conexão consome ~5–10MB de memória. Em escala, connection pooling é essencial.

O PgBouncer fica entre sua app e o Postgres, gerenciando um pool de conexões:

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
      POOL_MODE: transaction      # conexão retorna ao pool após cada transação
      MAX_CLIENT_CONN: 1000       # máximo de conexões simultâneas da app
      DEFAULT_POOL_SIZE: 20       # máximo de conexões Postgres reais por database/user
    ports:
      - "5432:5432"

  postgres:
    image: postgres:16
    environment:
      POSTGRES_PASSWORD: secret
```

```bash
# A aplicação conecta ao PgBouncer, não diretamente ao Postgres
DATABASE_URL=postgresql://myapp:secret@pgbouncer:5432/myapp
```

Com o modo `transaction`, sua app pode ter 1000 conexões com o PgBouncer, mas apenas 20 conexões Postgres reais ficam ativas ao mesmo tempo.

**Importante:** No modo transaction, variáveis de sessão com `SET` e prepared statements não persistem entre requisições. Use o modo `session` se precisar dessas funcionalidades (mas o mode session tem pool menos agressivo).

### Estratégia de indexação

Índices são a primeira otimização a buscar antes de escalar o hardware do banco de dados:

```sql
-- Encontra queries lentas (pg_stat_statements)
SELECT query, calls, mean_exec_time, total_exec_time
FROM pg_stat_statements
ORDER BY mean_exec_time DESC
LIMIT 10;

-- Encontra índices faltando (varreduras sequenciais em tabelas grandes)
SELECT schemaname, tablename, seq_scan, seq_tup_read, idx_scan
FROM pg_stat_user_tables
WHERE seq_scan > idx_scan
ORDER BY seq_scan DESC;

-- EXPLAIN ANALYZE para entender a execução da query
EXPLAIN ANALYZE
SELECT * FROM orders
WHERE user_id = '123' AND status = 'pending'
ORDER BY created_at DESC;
```

```typescript
// No schema do Prisma — índice composto para padrão de query comum
model Order {
  id        String   @id @default(cuid())
  userId    String
  status    String
  createdAt DateTime @default(now())

  @@index([userId, status, createdAt(sort: Desc)])
  // Cobre: WHERE userId = ? AND status = ? ORDER BY createdAt DESC
}
```

### Conceitos de sharding

Sharding divide dados entre múltiplas instâncias de banco de dados com base em uma partition key:

```typescript
// Sharding conceitual por ID de usuário
const SHARD_COUNT = 4;

function getShardIndex(userId: string): number {
  // Hash consistente do ID do usuário
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

// Uso
const db = getShardClient(request.user.id);
const orders = await db.order.findMany({ where: { userId: request.user.id } });
```

**Trade-offs do sharding:**
- Queries cross-shard (joins entre shards) são caras ou impossíveis
- Rebalancear dados ao adicionar shards é complexo
- Considere sharding apenas após esgotar read replicas e caching

### CQRS — Separação de Responsabilidades de Comando e Query

Separa os modelos de dados para leituras e escritas:

```typescript
// Modelo de escrita (normalizado, consistência transacional)
export class OrderCommandService {
  async placeOrder(userId: string, items: OrderItem[]) {
    return primaryDb.$transaction(async (tx) => {
      // Verifica estoque
      for (const item of items) {
        const product = await tx.product.findUnique({ where: { id: item.productId } });
        if (!product || product.stock < item.quantity) {
          throw new Error(`Insufficient stock for ${item.productId}`);
        }
      }

      // Cria pedido
      const order = await tx.order.create({
        data: { userId, status: 'pending', items: { create: items } },
      });

      // Decrementa estoque
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

// Modelo de leitura (desnormalizado, otimizado para exibição)
export class OrderQueryService {
  async getUserOrders(userId: string) {
    // Pode ler da replica, ou de um store separado otimizado para leitura
    return replicaDb.orderSummary.findMany({
      where: { userId },
      // orderSummary é uma view desnormalizada ou materialized view
    });
  }
}
```

---

## Padrões e Boas Práticas

- **Adicione read replicas antes de sharding** — mais simples, resolve 90% das necessidades de escalonamento
- **Indexe antes de escalar** — um índice faltando é 100× mais barato de corrigir do que adicionar hardware
- **Connection pooling desde o início** — PgBouncer ou o pool integrado do Prisma evitam esgotamento de conexões
- **Monitore percentis de latência de queries** — p99 importa mais do que p50; outliers lentos degradam a experiência do usuário
- **Use EXPLAIN ANALYZE** em qualquer query que leve > 100ms em produção
- **Evite SELECT \*** — busque apenas as colunas que você precisa; reduz transferência de dados e uso de memória
- **Use materialized views** para agregações custosas que rodam com frequência

---

## Anti-Padrões a Evitar

- Fazer sharding muito cedo — é complexo e você provavelmente não precisa ainda
- Ler de replicas em operações financeiras — o lag pode causar cobranças duplicadas ou verificações incorretas de saldo
- Queries N+1 — carregar registros relacionados um de cada vez em loop (use `include` no Prisma ou DataLoader)
- Não monitorar lag de replicação — se o lag crescer, as leituras da replica se distanciam mais do primary
- Transações de longa duração que bloqueiam linhas por segundos — causa acúmulo de conexões

---

## Debugging e Troubleshooting

**"CPU do banco de dados está em 100% sob carga normal"**
Execute `pg_stat_statements` para encontrar as queries com maior tempo total de execução. O culpado geralmente são 2-3 queries atingindo tabelas grandes sem índices.

**Erro "Too many connections"**
Adicione o PgBouncer. Reduza `max_connections` no seu cliente Prisma se estiver criando muitas instâncias. Cada instância Node.js deve usar 5-10 conexões Postgres, não 100.

**"A replica está 30 segundos atrás do primary"**
Verifique a carga de escrita no primary — escritas em bulk pesadas causam lag de replicação. Verifique também a rede entre primary e replica. Roteie leituras críticas para o primary até o lag se recuperar.

---

## Cenários do Mundo Real

**Cenário: Escalar uma API SaaS de 10K para 1M usuários**

Fase 1 (0–100K usuários): Instância única Postgres, índices adequados, PgBouncer.
Fase 2 (100K–500K): Adiciona 1-2 read replicas, roteia 80% das leituras para replicas.
Fase 3 (500K–1M): Adiciona camada de caching Redis (Track 08: Caching), reduz tráfego de leitura no DB em 90%.
Fase 4 (1M+): Reavalia: bancos separados por serviço, considera sharding de tabelas hot.

---

## Leitura Complementar

- [Guia de tuning de performance do Postgres](https://wiki.postgresql.org/wiki/Performance_Optimization)
- [Use the Index, Luke — guia de indexação SQL](https://use-the-index-luke.com/)
- [Documentação do PgBouncer](https://www.pgbouncer.org/config.html)
- Track 08: [Caching Strategies](caching-strategies.md)
- Track 08: [Teorema CAP](cap-theorem.md)

---

## Resumo

O escalonamento de banco de dados segue uma progressão previsível: otimização de índices primeiro (gratuita), depois connection pooling (barato), depois read replicas (complexidade moderada), depois caching (impacto significativo), depois sharding (caro e complexo). A maioria das aplicações nunca precisa de sharding. As ações de maior retorno são indexação adequada, `EXPLAIN ANALYZE` em queries lentas e adição de uma read replica para o volume de leituras. Connection pooling com PgBouncer ou ferramenta similar é essencial em qualquer escala não trivial.
