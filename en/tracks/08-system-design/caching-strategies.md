# Caching Strategies

## Overview

Caching stores the result of an expensive operation so subsequent requests can use the stored result instead of repeating the work. It is one of the highest-leverage performance techniques available — a well-placed cache can reduce database load by 90% and cut response times by orders of magnitude. This chapter covers the four main caching patterns, TTL and eviction strategies, and how to implement them in a Node.js/TypeScript application using Redis.

---

## Prerequisites

- Node.js and TypeScript
- Basic understanding of Redis (key-value store)
- Track 03 — Backend: databases and REST APIs

---

## Core Concepts

### Cache hit vs cache miss

```
Request → Cache? → Yes (hit) → Return cached value (fast)
                 → No (miss) → Fetch from source → Store in cache → Return value
```

**Hit rate** is the percentage of requests served from cache. A 90% hit rate means 10% of requests hit the database. A 99% hit rate means 1% do.

### The four main caching patterns

| Pattern | When to use | Write behavior | Complexity |
|---------|-------------|---------------|------------|
| Cache-aside (lazy) | Read-heavy workloads | App updates cache after DB write | Low |
| Read-through | Same as cache-aside, but cache manages loading | Cache fetches from DB on miss | Medium |
| Write-through | Data must always be consistent | Write to cache and DB simultaneously | Medium |
| Write-behind | Write-heavy, can tolerate brief inconsistency | Write to cache; async flush to DB | High |

---

## Hands-On Examples

### Cache-aside (the most common pattern)

The application code checks the cache, loads from DB on a miss, and populates the cache.

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
  // 1. Try cache
  const cached = await redis.get(key);
  if (cached !== null) {
    return JSON.parse(cached) as T;
  }

  // 2. Cache miss — load from source
  const value = await loader();

  // 3. Store in cache with TTL
  await redis.setEx(key, ttlSeconds, JSON.stringify(value));

  return value;
}

export async function invalidate(key: string): Promise<void> {
  await redis.del(key);
}

export async function invalidatePattern(pattern: string): Promise<void> {
  // Use with caution on large keyspaces — SCAN is preferred over KEYS
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
    300 // cache for 5 minutes
  );
}

export async function updateProduct(id: string, data: Partial<Product>) {
  const updated = await db.product.update({ where: { id }, data });

  // Invalidate the cache entry so next read fetches fresh data
  await invalidate(`product:${id}`);

  return updated;
}
```

### Write-through cache

Every write goes to the cache and the database simultaneously:

```typescript
export async function writeThrough<T extends { id: string }>(
  keyPrefix: string,
  id: string,
  data: T,
  dbWriter: (data: T) => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  // Write to DB
  const saved = await dbWriter(data);

  // Write to cache immediately (cache is always fresh after a write)
  await redis.setEx(`${keyPrefix}:${id}`, ttlSeconds, JSON.stringify(saved));

  return saved;
}

// Usage
const product = await writeThrough(
  'product',
  newProduct.id,
  newProduct,
  (p) => db.product.create({ data: p }),
  300
);
```

### Write-behind (write-back) cache

Writes go to the cache immediately; a background process flushes to the database:

```typescript
import { Queue, Worker } from 'bullmq';

const writeQueue = new Queue('db-writes', { connection: redis });

export async function writeBehind(key: string, data: unknown, ttl: number): Promise<void> {
  // 1. Write to cache immediately — client gets fast response
  await redis.setEx(key, ttl, JSON.stringify(data));

  // 2. Queue async write to DB
  await writeQueue.add('persist', { key, data });
}

new Worker('db-writes', async (job) => {
  const { key, data } = job.data;
  // Flush to database (extract entity type and ID from key convention)
  const [, entity, id] = key.split(':');
  await db[entity].upsert({ where: { id }, create: data, update: data });
}, { connection: redis });
```

Write-behind is complex — if the background worker fails before flushing, data is lost. Use it only for truly write-heavy scenarios (analytics counters, real-time leaderboards).

### Cache stampede prevention (dog-piling)

When a cached item expires, many simultaneous requests all miss the cache and hammer the database:

```typescript
// Probabilistic Early Expiration — refresh before expiry to avoid stampede
export async function getWithEarlyExpiry<T>(
  key: string,
  loader: () => Promise<T>,
  ttlSeconds: number
): Promise<T> {
  const raw = await redis.get(key);

  if (raw !== null) {
    // Probabilistically refresh if we're in the last 10% of TTL
    const ttl = await redis.ttl(key);
    const earlyRefreshThreshold = ttlSeconds * 0.1;

    if (ttl > earlyRefreshThreshold) {
      return JSON.parse(raw) as T;
    }
    // Fall through to refresh early
  }

  // Use a distributed lock to prevent multiple refreshes
  const lockKey = `lock:${key}`;
  const locked = await redis.set(lockKey, '1', { NX: true, EX: 5 });

  if (!locked) {
    // Another request is refreshing — return stale data if available
    if (raw !== null) return JSON.parse(raw) as T;
    // Wait a bit and retry
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

### TTL strategy by data type

```typescript
// Different TTLs for different data freshness requirements
const TTL = {
  USER_SESSION: 3600,          // 1 hour — security sensitive
  USER_PROFILE: 300,           // 5 minutes — changes infrequently
  PRODUCT_CATALOG: 600,        // 10 minutes — changes sometimes
  PRODUCT_INVENTORY: 30,       // 30 seconds — changes frequently
  HOMEPAGE_CONTENT: 3600,      // 1 hour — changes rarely
  SEARCH_RESULTS: 60,          // 1 minute — acceptable staleness
  RATE_LIMIT_COUNTER: 60,      // 1 minute window
  ONE_TIME_TOKEN: 900,         // 15 minutes — email verification, password reset
};
```

### Cache eviction policies

Configure Redis eviction policy to handle when memory is full:

```bash
# In redis.conf or via CLI
# LRU (Least Recently Used) — evict keys not recently used
CONFIG SET maxmemory-policy allkeys-lru

# LFU (Least Frequently Used) — evict keys used least often
CONFIG SET maxmemory-policy allkeys-lfu

# noeviction — return error when memory full (use for critical data)
CONFIG SET maxmemory-policy noeviction

# volatile-lru — only evict keys with TTL set (safe — never evicts keys without TTL)
CONFIG SET maxmemory-policy volatile-lru

# Set max memory
CONFIG SET maxmemory 256mb
```

For a general-purpose cache, `allkeys-lru` is the right policy. For a session store where you never want to silently lose sessions, use `noeviction` and monitor memory.

---

## Common Patterns & Best Practices

- **Cache at the right level** — caching a result close to the user (in-memory) is faster than caching in Redis, which is faster than hitting the DB. Layer your caches.
- **Use cache namespacing** — prefix keys by entity: `user:123`, `product:456`. This makes invalidation by pattern possible.
- **Cache serialized forms, not objects** — `JSON.stringify`/`JSON.parse` is cheap; passing rich objects through Redis is not possible.
- **Warm up caches on startup** — for critical paths, pre-populate cache before serving traffic.
- **Monitor hit rate** — below 80% hit rate, reconsider what you're caching or your TTLs.
- **Cache negative results** — if "user not found", cache that fact for a short TTL (e.g., 30 seconds) to prevent hammering the DB for missing entities.

---

## Anti-Patterns to Avoid

- Caching mutable, user-specific data without scoping the key by user ID (cache poisoning)
- Very long TTLs on data that changes frequently — users see stale data, hard to debug
- Forgetting to invalidate on write — the most common source of cache staleness bugs
- Using Redis as a primary database — it is a cache; data can be evicted at any time (unless persistence is configured)
- Not setting a TTL — cache grows indefinitely and eventually fills memory

---

## Debugging & Troubleshooting

**"Users are seeing stale data after an update"**
You forgot to invalidate the cache on write. Check every write path for a corresponding `redis.del(key)` call.

**"Cache hit rate is 20% — caching isn't helping"**
The cache key may be too specific (including a timestamp or a unique parameter that differs every request). Normalize keys: strip parameters that don't affect the result.

**"Redis memory is growing unbounded"**
You have keys with no TTL set. Add TTLs to all cache keys and configure `maxmemory` + `maxmemory-policy`.

---

## Real-World Scenarios

**Scenario: Product catalog API with high read traffic**

```typescript
fastify.get('/products/:id', async (request, reply) => {
  const { id } = request.params as { id: string };

  const product = await getOrSet(
    `product:${id}`,
    async () => {
      const p = await db.product.findUnique({ where: { id }, include: { category: true } });
      return p ?? null; // cache the null result too (negative caching)
    },
    TTL.PRODUCT_CATALOG
  );

  if (!product) return reply.status(404).send({ error: 'Product not found' });
  return product;
});

// On product update — invalidate cache
fastify.put('/products/:id', { preHandler: [requireAuth, requireRole('admin')] }, async (request) => {
  const { id } = request.params as { id: string };
  const data = UpdateProductSchema.parse(request.body);

  const updated = await db.product.update({ where: { id }, data });
  await invalidate(`product:${id}`);

  return updated;
});
```

---

## Further Reading

- [Redis caching patterns](https://redis.io/docs/manual/patterns/)
- [AWS ElastiCache caching strategies](https://docs.aws.amazon.com/AmazonElastiCache/latest/red-ug/Strategies.html)
- [Designing Data-Intensive Applications](https://dataintensive.net/) — Chapter 5
- Track 08: [CDN](cdn.md)
- Track 10: [HTTP Caching](../10-performance/caching-http.md)

---

## Summary

Caching is one of the most effective tools for improving performance and reducing infrastructure cost. Cache-aside is the right default for most web APIs — check cache first, load from DB on miss, store with a TTL. Write-through keeps the cache consistent with the database at the cost of slightly slower writes. Write-behind is powerful but complex and risky. TTL strategy is as important as the caching pattern — too short and you defeat the purpose; too long and users see stale data. Always invalidate on write, use namespaced keys, and monitor your hit rate.
