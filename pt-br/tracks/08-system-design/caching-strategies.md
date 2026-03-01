# Caching Strategies

## Visão Geral

Caching armazena o resultado de uma operação custosa para que requisições subsequentes possam usar o resultado armazenado em vez de repetir o trabalho. É uma das técnicas de performance com maior retorno disponíveis — um cache bem posicionado pode reduzir a carga no banco de dados em 90% e cortar os tempos de resposta em ordens de magnitude. Este capítulo cobre os quatro padrões principais de caching, estratégias de TTL e evição, e como implementá-los em uma aplicação Node.js/TypeScript usando Redis.

---

## Pré-requisitos

- Node.js e TypeScript
- Entendimento básico de Redis (key-value store)
- Track 03 — Backend: bancos de dados e APIs REST

---

## Conceitos Fundamentais

### Cache hit vs cache miss

```
Requisição → Cache? → Sim (hit) → Retorna valor em cache (rápido)
                   → Não (miss) → Busca na fonte → Armazena no cache → Retorna valor
```

**Hit rate** é o percentual de requisições servidas do cache. Um hit rate de 90% significa que 10% das requisições vão ao banco de dados. Um hit rate de 99% significa que 1% vai.

### Os quatro padrões principais de caching

| Padrão | Quando usar | Comportamento na escrita | Complexidade |
|--------|-------------|------------------------|--------------|
| Cache-aside (lazy) | Workloads com muitas leituras | App atualiza o cache após escrita no DB | Baixa |
| Read-through | Igual ao cache-aside, mas o cache gerencia o carregamento | Cache busca do DB no miss | Média |
| Write-through | Dados precisam estar sempre consistentes | Escreve no cache e no DB simultaneamente | Média |
| Write-behind | Muitas escritas, tolera breve inconsistência | Escreve no cache; flush assíncrono para o DB | Alta |

---

## Exemplos Práticos

### Cache-aside (o padrão mais comum)

O código da aplicação verifica o cache, carrega do DB no miss e popula o cache.

```typescript
// src/lib/cache.ts
import { createClient } from 'redis';

const redis = createClient({ url: process.env.REDIS_URL });
await redis.connect();

export async function getOrSet<T>(
  key: string,
  loader: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  // 1. Tenta o cache
  const cached = await redis.get(key);
  if (cached !== null) {
    return JSON.parse(cached) as T;
  }

  // 2. Cache miss — carrega da fonte
  const value = await loader();

  // 3. Armazena no cache com TTL
  await redis.setEx(key, ttlSeconds, JSON.stringify(value));

  return value;
}

export async function invalidate(key: string): Promise<void> {
  await redis.del(key);
}

export async function invalidatePattern(pattern: string): Promise<void> {
  // Use com cuidado em keyspaces grandes — SCAN é preferível a KEYS
  const keys = await redis.keys(pattern);
  if (keys.length > 0) {
    await redis.del(keys);
  }
}
```

```typescript
// src/services/product.service.ts
import { getOrSet, invalidate } from '../lib/cache.js';
import { db } from '../lib/db.js';

export async function getProduct(id: string) {
  return getOrSet(
    `product:${id}`,
    () => db.product.findUnique({ where: { id } }),
    300 // cache por 5 minutos
  );
}

export async function updateProduct(id: string, data: Partial<Product>) {
  const updated = await db.product.update({ where: { id }, data });

  // Invalida a entrada do cache para que a próxima leitura busque dados frescos
  await invalidate(`product:${id}`);

  return updated;
}
```

### Cache write-through

Toda escrita vai ao cache e ao banco de dados simultaneamente:

```typescript
export async function writeThrough<T extends { id: string }>(
  keyPrefix: string,
  id: string,
  data: T,
  dbWriter: (data: T) => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  // Escreve no DB
  const saved = await dbWriter(data);

  // Escreve no cache imediatamente (o cache fica sempre atualizado após uma escrita)
  await redis.setEx(`${keyPrefix}:${id}`, ttlSeconds, JSON.stringify(saved));

  return saved;
}

// Uso
const product = await writeThrough(
  'product',
  newProduct.id,
  newProduct,
  (p) => db.product.create({ data: p }),
  300
);
```

### Cache write-behind (write-back)

Escritas vão para o cache imediatamente; um processo em background faz o flush para o banco de dados:

```typescript
import { Queue, Worker } from 'bullmq';

const writeQueue = new Queue('db-writes', { connection: redis });

export async function writeBehind(key: string, data: unknown, ttl: number): Promise<void> {
  // 1. Escreve no cache imediatamente — cliente recebe resposta rápida
  await redis.setEx(key, ttl, JSON.stringify(data));

  // 2. Enfileira escrita assíncrona no DB
  await writeQueue.add('persist', { key, data });
}

new Worker('db-writes', async (job) => {
  const { key, data } = job.data;
  // Flush para o banco de dados (extrai tipo de entidade e ID da convenção de chave)
  const [, entity, id] = key.split(':');
  await db[entity].upsert({ where: { id }, create: data, update: data });
}, { connection: redis });
```

Write-behind é complexo — se o worker em background falhar antes do flush, os dados são perdidos. Use apenas para cenários genuinamente write-heavy (contadores de analytics, leaderboards em tempo real).

### Prevenção de cache stampede (dog-piling)

Quando um item em cache expira, muitas requisições simultâneas todas erram o cache e sobrecarregam o banco de dados:

```typescript
// Expiração antecipada probabilística — atualiza antes de expirar para evitar stampede
export async function getWithEarlyExpiry<T>(
  key: string,
  loader: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  const raw = await redis.get(key);

  if (raw !== null) {
    // Probabilisticamente atualiza se estamos nos últimos 10% do TTL
    const ttl = await redis.ttl(key);
    const earlyRefreshThreshold = ttlSeconds * 0.1;

    if (ttl > earlyRefreshThreshold) {
      return JSON.parse(raw) as T;
    }
    // Continua para atualização antecipada
  }

  // Usa um lock distribuído para evitar múltiplas atualizações simultâneas
  const lockKey = `lock:${key}`;
  const locked = await redis.set(lockKey, '1', { NX: true, EX: 5 });

  if (!locked) {
    // Outra requisição está atualizando — retorna dado desatualizado se disponível
    if (raw !== null) return JSON.parse(raw) as T;
    // Aguarda um pouco e tenta novamente
    await new Promise((r) => setTimeout(r, 100));
    return getWithEarlyExpiry(key, loader, ttlSeconds);
  }

  try {
    const value = await loader();
    await redis.setEx(key, ttlSeconds, JSON.stringify(value));
    return value;
  } finally {
    await redis.del(lockKey);
  }
}
```

### Estratégia de TTL por tipo de dado

```typescript
// TTLs diferentes para diferentes requisitos de frescor dos dados
const TTL = {
  USER_SESSION: 3600,          // 1 hora — sensível à segurança
  USER_PROFILE: 300,           // 5 minutos — muda com pouca frequência
  PRODUCT_CATALOG: 600,        // 10 minutos — muda às vezes
  PRODUCT_INVENTORY: 30,       // 30 segundos — muda com frequência
  HOMEPAGE_CONTENT: 3600,      // 1 hora — muda raramente
  SEARCH_RESULTS: 60,          // 1 minuto — staleness aceitável
  RATE_LIMIT_COUNTER: 60,      // janela de 1 minuto
  ONE_TIME_TOKEN: 900,         // 15 minutos — verificação de email, reset de senha
};
```

### Políticas de evição do cache

Configure a política de evição do Redis para quando a memória estiver cheia:

```bash
# Em redis.conf ou via CLI
# LRU (Least Recently Used) — evicta keys não usadas recentemente
CONFIG SET maxmemory-policy allkeys-lru

# LFU (Least Frequently Used) — evicta keys usadas com menos frequência
CONFIG SET maxmemory-policy allkeys-lfu

# noeviction — retorna erro quando a memória estiver cheia (use para dados críticos)
CONFIG SET maxmemory-policy noeviction

# volatile-lru — evicta apenas keys com TTL definido (seguro — nunca evicta keys sem TTL)
CONFIG SET maxmemory-policy volatile-lru

# Define memória máxima
CONFIG SET maxmemory 256mb
```

Para um cache de uso geral, `allkeys-lru` é a política correta. Para um session store onde você nunca quer perder sessões silenciosamente, use `noeviction` e monitore a memória.

---

## Padrões e Boas Práticas

- **Faça cache no nível certo** — cachear um resultado próximo ao usuário (in-memory) é mais rápido do que cachear no Redis, que é mais rápido do que bater no DB. Camadeie seus caches.
- **Use namespacing de chaves** — prefixe chaves por entidade: `user:123`, `product:456`. Isso torna possível a invalidação por padrão.
- **Cache formas serializadas, não objetos** — `JSON.stringify`/`JSON.parse` é barato; passar objetos ricos pelo Redis não é possível.
- **Faça warm-up do cache na inicialização** — para caminhos críticos, pre-popule o cache antes de servir tráfego.
- **Monitore o hit rate** — abaixo de 80% de hit rate, reconsidere o que você está cacheando ou seus TTLs.
- **Cache resultados negativos** — se "usuário não encontrado", cache esse fato por um TTL curto (ex: 30 segundos) para evitar sobrecarregar o DB com entidades inexistentes.

---

## Anti-Padrões a Evitar

- Cachear dados mutáveis e específicos de usuário sem escopar a chave pelo ID do usuário (envenenamento de cache)
- TTLs muito longos em dados que mudam com frequência — usuários veem dados desatualizados, difícil de debugar
- Esquecer de invalidar na escrita — a fonte mais comum de bugs de staleness em cache
- Usar Redis como banco de dados primário — é um cache; dados podem ser evictados a qualquer momento (a menos que persistência esteja configurada)
- Não definir TTL — o cache cresce indefinidamente e eventualmente enche a memória

---

## Debugging e Troubleshooting

**"Usuários estão vendo dados desatualizados após uma atualização"**
Você esqueceu de invalidar o cache na escrita. Verifique todo caminho de escrita por uma chamada correspondente a `redis.del(key)`.

**"Hit rate do cache é 20% — o caching não está ajudando"**
A chave de cache pode ser específica demais (incluindo um timestamp ou um parâmetro único que difere em cada requisição). Normalize as chaves: elimine parâmetros que não afetam o resultado.

**"Memória do Redis está crescendo sem limite"**
Você tem chaves sem TTL definido. Adicione TTLs a todas as chaves de cache e configure `maxmemory` + `maxmemory-policy`.

---

## Cenários do Mundo Real

**Cenário: API de catálogo de produtos com alto tráfego de leitura**

```typescript
fastify.get('/products/:id', async (request, reply) => {
  const { id } = request.params as { id: string };

  const product = await getOrSet(
    `product:${id}`,
    async () => {
      const p = await db.product.findUnique({ where: { id }, include: { category: true } });
      return p ?? null; // cacheia também o resultado null (negative caching)
    },
    TTL.PRODUCT_CATALOG
  );

  if (!product) return reply.status(404).send({ error: 'Product not found' });
  return product;
});

// Na atualização do produto — invalida o cache
fastify.put('/products/:id', { preHandler: [requireAuth, requireRole('admin')] }, async (request) => {
  const { id } = request.params as { id: string };
  const data = UpdateProductSchema.parse(request.body);

  const updated = await db.product.update({ where: { id }, data });
  await invalidate(`product:${id}`);

  return updated;
});
```

---

## Leitura Complementar

- [Padrões de caching do Redis](https://redis.io/docs/manual/patterns/)
- [Estratégias de caching do AWS ElastiCache](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html)
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Capítulo 5
- Track 08: [CDN](cdn.md)
- Track 10: [HTTP Caching](../10-performance/caching-http.md)

---

## Resumo

Caching é uma das ferramentas mais eficazes para melhorar performance e reduzir custos de infraestrutura. Cache-aside é o padrão correto para a maioria das APIs web — verifique o cache primeiro, carregue do DB no miss, armazene com TTL. Write-through mantém o cache consistente com o banco de dados ao custo de escritas levemente mais lentas. Write-behind é poderoso, mas complexo e arriscado. A estratégia de TTL é tão importante quanto o padrão de caching — muito curto e você anula o propósito; muito longo e os usuários veem dados desatualizados. Sempre invalide na escrita, use chaves com namespace e monitore seu hit rate.
