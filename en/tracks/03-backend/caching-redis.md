# Caching & Redis

## 1. What & Why

Caching is the practice of storing a copy of data in a faster storage layer to serve future requests more quickly. A database query that takes 50ms might serve from cache in 0.1ms — a 500x speedup. Caching is one of the highest-leverage performance optimizations available, and also one of the most dangerous if done incorrectly.

Redis (Remote Dictionary Server) is the industry-standard caching layer: single-threaded, in-memory, sub-millisecond latency, with rich data structures that enable far more than simple key-value caching. Redis is used for sessions, rate limiting, distributed locks, leaderboards, pub/sub, and background job queues.

The most famous quote in computer science: "There are only two hard things in Computer Science: cache invalidation and naming things." — Phil Karlton. This file addresses both.

---

## 2. Core Concepts

### Cache Strategies

**Cache-Aside (Lazy Loading) — most common:**

The application code manages the cache. On cache miss, the app reads from DB and populates the cache.

```
Read flow:
  1. Check cache for key
  2. If HIT: return cached value
  3. If MISS: read from DB, write to cache with TTL, return value

Write flow:
  1. Write to DB
  2. Invalidate or update cache key
```

Pros: only requested data is cached; cache failures are not catastrophic (app reads from DB). Cons: initial requests always miss (cold start); inconsistency window between DB write and cache invalidation.

**Write-Through:**

Every write goes to both cache and DB simultaneously.

```
Write flow:
  1. Write to cache
  2. Write to DB (synchronously)
```

Pros: cache is always consistent with DB; no inconsistency window. Cons: every write has double latency; cache fills with data that may never be read again (wasteful).

**Write-Behind (Write-Back):**

Writes go to cache first; DB writes are async (buffered).

```
Write flow:
  1. Write to cache immediately (return to client)
  2. Async: batch-write cache changes to DB
```

Pros: very fast writes; DB write batching reduces load. Cons: data loss risk if cache crashes before DB write; complex to implement correctly.

**Read-Through:**

Cache sits in front of DB; on miss, the cache itself fetches from DB.

```
Read flow:
  1. App reads from cache
  2. Cache miss: cache fetches from DB, stores, returns
  3. App always talks to cache only
```

Pros: transparent to application code. Cons: requires cache to have DB access credentials; less flexible.

> 💡 Cache-aside is the right default for most applications. Use write-through when consistency is critical. Write-behind only for very high write throughput where some data loss is acceptable (analytics, metrics).

---

## 3. How It Works — Redis Data Types

### Strings

```typescript
import Redis from 'ioredis';
const redis = new Redis(process.env.REDIS_URL!);

// Basic cache with TTL
await redis.setex(`user:${id}`, 300, JSON.stringify(userData)); // 5 min TTL
const cached = await redis.get(`user:${id}`);
const user = cached ? JSON.parse(cached) : null;

// Atomic counter (INCR is atomic — no race conditions)
const views = await redis.incr(`post:${postId}:views`);
await redis.incrby(`user:${userId}:credits`, 50);
await redis.decrby(`user:${userId}:credits`, 10);

// GET and SET atomically (atomically swap value, get old)
const prev = await redis.getset(`config:maintenance`, 'true');

// SET with options
await redis.set('lock:resource', 'owner-id', 'EX', 30, 'NX'); // EX=TTL, NX=only if not exists
```

### Hashes

```typescript
// Store objects without JSON serialization overhead
await redis.hset(`session:${sessionId}`, {
  userId: user.id,
  role: user.role,
  ip: req.ip,
  createdAt: Date.now().toString(),
});
await redis.expire(`session:${sessionId}`, 3600);

const session = await redis.hgetall(`session:${sessionId}`);
// { userId: '...', role: 'admin', ip: '...', createdAt: '...' }

// Increment a single hash field atomically
await redis.hincrby(`stats:${date}`, 'pageViews', 1);
await redis.hincrby(`stats:${date}`, 'uniqueVisitors', 1);
```

### Lists

```typescript
// Queue (FIFO): RPUSH to enqueue, LPOP to dequeue
await redis.rpush('job:queue', JSON.stringify(job));
const item = await redis.lpop('job:queue');

// Stack (LIFO): RPUSH to push, RPOP to pop
await redis.rpush('undo:stack', JSON.stringify(action));
const last = await redis.rpop('undo:stack');

// Activity feed: keep last N items
await redis.lpush(`feed:${userId}`, JSON.stringify(activity));
await redis.ltrim(`feed:${userId}`, 0, 99); // keep only 100 most recent
const feed = await redis.lrange(`feed:${userId}`, 0, 19); // first 20

// Blocking pop (for job workers — blocks until item available)
const [queue, value] = await redis.blpop('job:queue', 0); // 0 = block indefinitely
```

### Sets

```typescript
// Unique collections — O(1) add, remove, contains
await redis.sadd(`post:${postId}:likers`, userId);
await redis.srem(`post:${postId}:likers`, userId);
const liked = await redis.sismember(`post:${postId}:likers`, userId); // 0 or 1
const likeCount = await redis.scard(`post:${postId}:likers`);

// Set operations: union, intersection, difference
const commonFollowers = await redis.sinter(`user:1:followers`, `user:2:followers`);
const allFollowers = await redis.sunion(`user:1:followers`, `user:2:followers`);
```

### Sorted Sets

```typescript
// Leaderboard: O(log n) add and rank lookup
await redis.zadd('leaderboard:season5', score, userId);

// Top 10 players (highest score first)
const top10 = await redis.zrevrange('leaderboard:season5', 0, 9, 'WITHSCORES');
// ['userId1', '9850', 'userId2', '9720', ...]

// Rank of a specific player (0-indexed, higher = better)
const rank = await redis.zrevrank('leaderboard:season5', userId);
// null if not in set; 0 = first place

// Players around a user (for "near you" leaderboard)
const userRank = await redis.zrevrank('leaderboard:season5', userId);
const nearby = await redis.zrevrange('leaderboard:season5',
  Math.max(0, userRank - 2), userRank + 2, 'WITHSCORES');

// Rate limiting with sorted set (sliding window)
const now = Date.now();
const windowStart = now - 60_000; // 1 minute window
await redis.zremrangebyscore(`rate:${userId}`, 0, windowStart);
const count = await redis.zcard(`rate:${userId}`);
if (count >= 100) throw new RateLimitError();
await redis.zadd(`rate:${userId}`, now, `${now}-${Math.random()}`);
await redis.expire(`rate:${userId}`, 60);
```

---

## 4. Code Examples (TypeScript / ioredis)

### Cache-Aside Pattern

```typescript
interface CacheOptions {
  ttl?: number; // seconds
  staleWhileRevalidate?: number; // seconds — serve stale while refreshing
}

class CacheService {
  constructor(private redis: Redis) {}

  async getOrSet<T>(
    key: string,
    fetchFn: () => Promise<T>,
    options: CacheOptions = {}
  ): Promise<T> {
    const { ttl = 300 } = options;

    const cached = await this.redis.get(key);
    if (cached !== null) {
      return JSON.parse(cached) as T;
    }

    const value = await fetchFn();
    await this.redis.setex(key, ttl, JSON.stringify(value));
    return value;
  }

  async invalidate(key: string): Promise<void> {
    await this.redis.del(key);
  }

  async invalidatePattern(pattern: string): Promise<void> {
    // SCAN instead of KEYS — KEYS blocks Redis
    let cursor = '0';
    do {
      const [nextCursor, keys] = await this.redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
      cursor = nextCursor;
      if (keys.length > 0) {
        await this.redis.del(...keys);
      }
    } while (cursor !== '0');
  }
}

// Usage in a service
class UserService {
  constructor(private db: Database, private cache: CacheService) {}

  async getUser(id: string): Promise<User> {
    return this.cache.getOrSet(
      `user:${id}`,
      () => this.db.users.findUniqueOrThrow({ where: { id } }),
      { ttl: 300 }
    );
  }

  async updateUser(id: string, data: Partial<User>): Promise<User> {
    const updated = await this.db.users.update({ where: { id }, data });
    await this.cache.invalidate(`user:${id}`); // invalidate after write
    return updated;
  }
}
```

### Rate Limiting with Sliding Window

```typescript
// Atomic rate limiter using Lua script to prevent race conditions
const RATE_LIMIT_SCRIPT = `
  local key = KEYS[1]
  local now = tonumber(ARGV[1])
  local window = tonumber(ARGV[2])
  local limit = tonumber(ARGV[3])
  local unique = ARGV[4]

  -- Remove expired entries
  redis.call('ZREMRANGEBYSCORE', key, 0, now - window * 1000)

  -- Count current entries
  local count = redis.call('ZCARD', key)

  if count >= limit then
    -- Get oldest entry to calculate reset time
    local oldest = redis.call('ZRANGE', key, 0, 0, 'WITHSCORES')
    local resetAt = tonumber(oldest[2]) + window * 1000
    return {0, count, resetAt}
  end

  -- Add current request
  redis.call('ZADD', key, now, unique)
  redis.call('EXPIRE', key, window)

  return {1, count + 1, 0}
`;

async function rateLimit(
  redis: Redis,
  key: string,
  limit: number,
  windowSecs: number
): Promise<{ allowed: boolean; remaining: number; resetAt: number }> {
  const now = Date.now();
  const unique = `${now}-${Math.random()}`;

  const result = await redis.eval(
    RATE_LIMIT_SCRIPT, 1, key, now, windowSecs, limit, unique
  ) as [number, number, number];

  const [allowed, count, resetAt] = result;
  return {
    allowed: allowed === 1,
    remaining: Math.max(0, limit - count),
    resetAt: resetAt || now + windowSecs * 1000,
  };
}

// Fastify middleware
app.addHook('preHandler', async (req, reply) => {
  const userId = req.user?.id ?? req.ip;
  const { allowed, remaining, resetAt } = await rateLimit(
    redis, `rate:api:${userId}`, 100, 60
  );

  reply.header('X-RateLimit-Limit', '100');
  reply.header('X-RateLimit-Remaining', remaining.toString());
  reply.header('X-RateLimit-Reset', Math.ceil(resetAt / 1000).toString());

  if (!allowed) {
    return reply.status(429).send({
      type: 'https://api.example.com/errors/rate-limited',
      title: 'Too Many Requests',
      status: 429,
      detail: 'Rate limit exceeded. Please retry after the reset time.',
    });
  }
});
```

### Distributed Lock (Redlock)

```typescript
// Simplified Redlock implementation for single-node Redis
// For production multi-node deployments, use the 'redlock' npm package

async function acquireLock(
  redis: Redis,
  resource: string,
  ttlMs: number
): Promise<string | null> {
  const lockId = crypto.randomUUID();
  const key = `lock:${resource}`;

  // SET NX EX: set only if not exists, with expiry
  const result = await redis.set(key, lockId, 'PX', ttlMs, 'NX');
  return result === 'OK' ? lockId : null;
}

async function releaseLock(
  redis: Redis,
  resource: string,
  lockId: string
): Promise<boolean> {
  // Lua script: atomically check ownership + delete
  // Without this, you might delete a lock you don't own (race condition)
  const RELEASE_SCRIPT = `
    if redis.call('GET', KEYS[1]) == ARGV[1] then
      return redis.call('DEL', KEYS[1])
    else
      return 0
    end
  `;
  const result = await redis.eval(RELEASE_SCRIPT, 1, `lock:${resource}`, lockId);
  return result === 1;
}

async function withLock<T>(
  redis: Redis,
  resource: string,
  ttlMs: number,
  fn: () => Promise<T>
): Promise<T> {
  const lockId = await acquireLock(redis, resource, ttlMs);
  if (!lockId) {
    throw new Error(`Could not acquire lock for resource: ${resource}`);
  }
  try {
    return await fn();
  } finally {
    await releaseLock(redis, resource, lockId);
  }
}

// Usage: prevent double-charging a payment
await withLock(redis, `payment:${orderId}`, 30_000, async () => {
  const order = await db.orders.findUnique({ where: { id: orderId } });
  if (order?.status === 'PAID') return; // idempotent
  await paymentProvider.charge(order.amount);
  await db.orders.update({ where: { id: orderId }, data: { status: 'PAID' } });
});
```

### Session Storage

```typescript
// Store user sessions in Redis
interface Session {
  userId: string;
  role: string;
  ip: string;
  createdAt: number;
  lastAccessedAt: number;
}

class SessionStore {
  private SESSION_TTL = 7 * 24 * 3600; // 7 days

  constructor(private redis: Redis) {}

  async create(userId: string, role: string, ip: string): Promise<string> {
    const sessionId = crypto.randomBytes(32).toString('base64url');
    const session: Session = {
      userId, role, ip,
      createdAt: Date.now(),
      lastAccessedAt: Date.now(),
    };

    await this.redis.hset(`session:${sessionId}`, session as any);
    await this.redis.expire(`session:${sessionId}`, this.SESSION_TTL);

    return sessionId;
  }

  async get(sessionId: string): Promise<Session | null> {
    const data = await this.redis.hgetall(`session:${sessionId}`);
    if (!data || Object.keys(data).length === 0) return null;

    // Sliding expiration: extend TTL on every access
    await this.redis.expire(`session:${sessionId}`, this.SESSION_TTL);
    await this.redis.hset(`session:${sessionId}`, 'lastAccessedAt', Date.now());

    return data as unknown as Session;
  }

  async destroy(sessionId: string): Promise<void> {
    await this.redis.del(`session:${sessionId}`);
  }

  // Revoke all sessions for a user (e.g., after password change)
  async destroyAllForUser(userId: string): Promise<void> {
    let cursor = '0';
    do {
      const [next, keys] = await this.redis.scan(cursor, 'MATCH', 'session:*', 'COUNT', 100);
      cursor = next;
      for (const key of keys) {
        const uid = await this.redis.hget(key, 'userId');
        if (uid === userId) await this.redis.del(key);
      }
    } while (cursor !== '0');
  }
}
```

### Pipeline and Transaction

```typescript
// Pipeline: batch multiple commands, reduce round trips (non-atomic)
const pipeline = redis.pipeline();
pipeline.get('key1');
pipeline.get('key2');
pipeline.set('key3', 'value3', 'EX', 300);
pipeline.incr('counter');
const results = await pipeline.exec();
// results: [[null, 'value1'], [null, 'value2'], [null, 'OK'], [null, 1]]

// Transaction (MULTI/EXEC): atomic batch — all succeed or none apply
const multi = redis.multi();
multi.set('account:1:balance', '900');
multi.set('account:2:balance', '1100');
multi.incrby('audit:transfers', 1);
const txResults = await multi.exec(); // returns null if WATCH detected a change
```

---

## 5. Common Mistakes & Pitfalls

**Cache stampede (thundering herd):**

```typescript
// Problem: all requests hit the DB simultaneously when a cache key expires
// Imagine 1000 requests/sec and a popular key expires at 14:00:00.000
// All 1000 requests get a cache miss simultaneously → 1000 DB queries

// Fix 1: Lock-based (only one request fetches, others wait)
async function getWithStampedeProtection<T>(
  redis: Redis,
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number
): Promise<T> {
  const cached = await redis.get(key);
  if (cached) return JSON.parse(cached);

  // Try to acquire the "I'm fetching" lock
  const lockKey = `${key}:fetching`;
  const acquired = await redis.set(lockKey, '1', 'EX', 10, 'NX');

  if (acquired) {
    // We got the lock — fetch and populate
    const value = await fetchFn();
    await redis.setex(key, ttl, JSON.stringify(value));
    await redis.del(lockKey);
    return value;
  }

  // Another request is fetching — wait and retry
  await new Promise((r) => setTimeout(r, 50));
  return getWithStampedeProtection(redis, key, fetchFn, ttl);
}

// Fix 2: Probabilistic early expiration (simpler, no locking)
// Refresh the cache before it expires when load is detected
async function getWithEarlyExpiry<T>(
  redis: Redis,
  key: string,
  fetchFn: () => Promise<T>,
  ttl: number,
  beta = 1
): Promise<T> {
  const now = Date.now() / 1000;
  const dataKey = `${key}:data`;
  const expiryKey = `${key}:expiry`;

  const [cached, expiry] = await Promise.all([
    redis.get(dataKey),
    redis.get(expiryKey),
  ]);

  if (cached && expiry) {
    const timeRemaining = parseFloat(expiry) - now;
    const shouldRefresh = -beta * Math.log(Math.random()) * 0 > timeRemaining;
    // Slightly randomized: some requests refresh before expiry to avoid synchronized expiry
    if (!shouldRefresh) return JSON.parse(cached);
  }

  const value = await fetchFn();
  const newExpiry = now + ttl;
  await redis.setex(dataKey, ttl, JSON.stringify(value));
  await redis.setex(expiryKey, ttl, newExpiry.toString());
  return value;
}
```

**Using KEYS command in production:**

```typescript
// WRONG: KEYS blocks Redis for the duration of the scan — O(N) where N = all keys
const keys = await redis.keys('user:*'); // freezes Redis for seconds on large datasets

// CORRECT: use SCAN — non-blocking cursor-based iteration
async function scanKeys(redis: Redis, pattern: string): Promise<string[]> {
  const results: string[] = [];
  let cursor = '0';
  do {
    const [nextCursor, keys] = await redis.scan(cursor, 'MATCH', pattern, 'COUNT', 100);
    cursor = nextCursor;
    results.push(...keys);
  } while (cursor !== '0');
  return results;
}
```

**No TTL on cached values:**

```typescript
// WRONG: cache grows forever, stale data never expires
await redis.set(`user:${id}`, JSON.stringify(user));

// CORRECT: always set TTL
await redis.setex(`user:${id}`, 300, JSON.stringify(user)); // 5 minutes
// OR
await redis.set(`user:${id}`, JSON.stringify(user), 'EX', 300);
```

**Caching mutable data for too long:**

```typescript
// Danger: caching user profile for 1 hour
// User updates their name → old name served for up to 1 hour
// Fix: invalidate on write AND use a shorter TTL as fallback
await redis.setex(`user:${id}`, 60, JSON.stringify(user)); // 60s max staleness

// On update: immediately invalidate
await db.users.update({ where: { id }, data: { name: newName } });
await redis.del(`user:${id}`); // invalidate cache immediately
```

---

## 6. When to Use / Not Use

**Use Redis caching for:**
- Expensive DB queries that are read frequently (user profiles, product details, configuration)
- API responses for public data (news feed, product listings)
- Computed data that is expensive to recalculate (aggregations, recommendation scores)
- Rate limiting, session storage, distributed locks

**Avoid caching for:**
- Highly volatile data where stale reads are unacceptable (bank balances, inventory counts)
- Data that is rarely requested (low hit rate = cache pollution)
- Very small, fast queries (caching overhead may exceed query time)
- Personalized data with high cardinality keys (one key per user per resource = too many keys)

**Redis vs Memcached:**

| Factor | Redis | Memcached |
|--------|-------|-----------|
| Data structures | Rich (strings, hashes, lists, sets, sorted sets, streams) | Strings only |
| Persistence | RDB + AOF | None |
| Clustering | Redis Cluster (native) | Consistent hashing (client-side) |
| Pub/sub | Built-in | Not supported |
| Lua scripting | Supported | Not supported |
| Memory efficiency | Slightly less | Slightly more |
| Choose when | Almost always | Simplest possible cache, memory efficiency critical |

> 💡 Redis has effectively superseded Memcached for new projects. The only reason to choose Memcached today is if you have existing infrastructure or need slightly better memory efficiency for pure string caching at extreme scale.

---

## 7. TTL Strategy

```typescript
// TTL guidelines by data type
const TTL = {
  // Near-real-time (changes frequently)
  stockPrice: 5,           // 5 seconds
  liveScore: 10,           // 10 seconds

  // Short-lived (minutes)
  searchResults: 60,       // 1 minute
  userActivity: 120,       // 2 minutes

  // Medium-lived (hours)
  userProfile: 3600,       // 1 hour
  productDetails: 1800,    // 30 minutes

  // Long-lived (day+)
  staticConfig: 86400,     // 24 hours
  countryList: 86400 * 7,  // 1 week

  // Session-based
  authSession: 1800,       // 30 minutes idle timeout
  refreshToken: 86400 * 30, // 30 days
};

// Jitter: add randomness to prevent synchronized expiry
function ttlWithJitter(baseTtl: number, jitterPercent = 10): number {
  const jitter = baseTtl * (jitterPercent / 100);
  return Math.floor(baseTtl + (Math.random() * 2 - 1) * jitter);
}

// Instead of all user profiles expiring at the same time
await redis.setex(`user:${id}`, ttlWithJitter(3600), JSON.stringify(user));
// Some expire at 3240s, some at 3760s — no synchronized stampede
```

---

## 8. Eviction Policies

When Redis reaches its memory limit (`maxmemory`), it evicts keys based on the configured policy:

```
# redis.conf
maxmemory 512mb
maxmemory-policy allkeys-lru

Policies:
  noeviction:      Write error when memory full. Use for: primary data store (no eviction acceptable)
  allkeys-lru:     Evict least recently used across ALL keys. Use for: pure cache (every key is expendable)
  volatile-lru:    LRU eviction only from keys WITH TTL. Use for: mix of cache + persistent data
  allkeys-lfu:     Evict least frequently used. Use for: cache with power-law access (some keys 100x hotter)
  volatile-lfu:    LFU only from keys with TTL.
  allkeys-random:  Random eviction. Rarely useful.
  volatile-ttl:    Evict key with shortest remaining TTL. Use for: prioritizing long-lived keys
  volatile-random: Random eviction from keys with TTL.
```

---

## 9. Real-World Scenario

**Problem:** A news website serves 50,000 requests/minute on a viral story. The "get article with comment count + like count + author details" query takes 80ms from PostgreSQL. Under this load, the DB is at 100% CPU.

**Solution: Cache-aside with invalidation**

```typescript
class ArticleService {
  constructor(private db: PrismaClient, private redis: Redis) {}

  async getArticle(slug: string): Promise<ArticleDTO | null> {
    const cacheKey = `article:${slug}`;
    const cached = await this.redis.get(cacheKey);

    if (cached) {
      // Cache hit: 0.3ms
      return JSON.parse(cached);
    }

    // Cache miss: 80ms DB query
    const article = await this.db.article.findUnique({
      where: { slug },
      include: {
        author: { select: { name: true, avatarUrl: true } },
        _count: { select: { comments: true, likes: true } },
      },
    });

    if (!article) return null;

    const dto: ArticleDTO = {
      ...article,
      commentCount: article._count.comments,
      likeCount: article._count.likes,
    };

    // Cache for 5 minutes + jitter
    const ttl = ttlWithJitter(300);
    await this.redis.setex(cacheKey, ttl, JSON.stringify(dto));

    return dto;
  }

  // Invalidate when article is updated
  async updateArticle(id: string, data: Partial<Article>): Promise<Article> {
    const updated = await this.db.article.update({ where: { id }, data });
    await this.redis.del(`article:${updated.slug}`);
    return updated;
  }

  // Invalidate when a new comment is added (comment count changed)
  async addComment(articleId: string, comment: CreateCommentInput): Promise<Comment> {
    const article = await this.db.article.findUniqueOrThrow({ where: { id: articleId } });
    const newComment = await this.db.comment.create({
      data: { ...comment, articleId },
    });
    await this.redis.del(`article:${article.slug}`); // invalidate stale count
    return newComment;
  }
}
```

**Result:** Cache hit rate reaches 98% within 30 seconds of the story going viral. Database CPU drops from 100% to 4%. Response time for cached requests: 0.5ms. The DB handles only 1,000 requests/minute (2% misses) instead of 50,000.

---

## 10. Interview Questions

**Q1: What is the difference between cache-aside and write-through strategies?**

Cache-aside (lazy loading): the application checks the cache first; on miss, reads from DB and populates the cache. The cache only contains data that has been requested at least once. Write-through: every write goes to both cache and DB synchronously. The cache is always in sync but may contain data that is never read again, and every write has higher latency. Cache-aside is more common because it avoids polluting the cache with unread data; write-through is used when read-after-write consistency is critical.

**Q2: What eviction policy would you use for a session store and why?**

`volatile-lru` or `noeviction`. For a session store, sessions are the primary data — you must not evict them to make room for other keys. Use `volatile-lru` if Redis also serves as a cache (sessions have TTL set; cached items also have TTL; LRU ensures least-recently-used cache items are evicted first). Use `noeviction` if Redis is dedicated to sessions only (Redis returns errors when memory is full rather than silently losing sessions). Never use `allkeys-lru` for a session store — it would evict sessions even when non-session keys exist.

**Q3: What are the strategies for cache invalidation?**

(1) **TTL-based** — simplest; set an expiry and accept that data can be stale for up to TTL seconds. (2) **Event-based invalidation** — when data changes in DB, explicitly delete the cache key (`redis.del(key)`). (3) **Write-through** — cache is always updated on write, never stale. (4) **Cache versioning** — include a version token in the cache key; changing the version effectively invalidates all old keys without deleting them (useful for bulk invalidation). (5) **Tag-based invalidation** — tag cache keys with entity IDs; on entity update, delete all tagged keys (e.g., all pages related to product P).

**Q4: What is a cache stampede and how do you prevent it?**

A cache stampede (thundering herd) occurs when a popular cache key expires and many concurrent requests all get a cache miss simultaneously, causing them all to hit the database at once. Prevention strategies: (1) **Mutex/lock** — first thread to get a miss acquires a Redis lock and fetches; others wait and retry. (2) **Probabilistic early expiration** — some requests start refreshing the cache before it expires, staggered randomly. (3) **Stale-while-revalidate** — serve the stale cached value immediately; asynchronously refresh it in the background. (4) **TTL jitter** — add randomness to TTLs so keys for similar data don't expire simultaneously.

**Q5: What is Redlock and when do you need it?**

Redlock is a distributed locking algorithm for Redis. The basic pattern: `SET key value NX EX ttl` (set only if not exists with expiry). For a single Redis node, this is sufficient for most use cases. Redlock extends this to multiple Redis nodes: a lock is considered acquired if a majority of N nodes (N/2+1) grant it, preventing a single node failure from making the lock unavailable or creating a split-brain. Use distributed locks when you need to ensure only one process executes a critical section at a time (e.g., preventing double payments, scheduling exclusive jobs).

**Q6: How does Redis compare to Memcached for caching?**

Redis is the right choice for almost all new projects. Reasons: Redis supports rich data structures (hashes, sorted sets, lists) enabling use cases beyond simple caching (leaderboards, rate limiting, sessions, pub/sub); Redis has persistence (RDB, AOF) so data survives restarts; Redis supports Lua scripting for atomic multi-step operations; Redis Cluster provides native sharding. Memcached is simpler, slightly more memory-efficient for pure string caching, and can be better for multi-threaded write workloads. In practice, Redis's versatility wins in almost every scenario.

**Q7: What are sorted sets in Redis and what are typical use cases?**

Sorted sets store unique string members each associated with a float score. Members are ordered by score; ties are broken lexicographically. Operations: `ZADD` (add/update), `ZRANK`/`ZREVRANK` (O(log n) rank lookup), `ZRANGE`/`ZREVRANGE` (get members by rank range), `ZRANGEBYSCORE` (get by score range). Use cases: (1) Leaderboards — `ZADD leaderboard score userId`, `ZREVRANGE leaderboard 0 9` for top 10. (2) Rate limiting — score = timestamp, range queries remove expired entries. (3) Scheduled jobs — score = execution timestamp, workers process jobs whose score <= now(). (4) Autocomplete — score 0 for all entries, range query by lexicographic prefix.

**Q8: How would you implement a fixed-window rate limiter vs a sliding-window rate limiter in Redis?**

**Fixed window:** Increment a counter per user per time window: `INCR rate:userId:minute:1440`. If count > limit, reject. Simple but allows 2x burst at window boundaries (100 requests at 00:59 + 100 at 01:00 = 200 in 2 seconds). **Sliding window with sorted set:** `ZADD rate:userId timestamp uniqueId`, remove entries older than window: `ZREMRANGEBYSCORE rate:userId 0 (now - window)`, count with `ZCARD`. Always exactly counts requests in the last N seconds. The Lua script atomicity is critical — without it, two concurrent requests can both pass the count check before either adds their entry.

---

## 11. Exercises

**Exercise 1: Cache warming**

A product listing page takes 800ms to load from DB on a cold cache. Build a cache warming strategy that:
- Pre-populates the 100 most-viewed products on application startup
- Refreshes them every 10 minutes without causing a stampede
- Uses a Lua script to atomically check TTL and conditionally refresh

*Hint:* Use `TTL` command to check remaining TTL. If < 60 seconds, refresh. Wrap in Lua to avoid race conditions.

**Exercise 2: Multi-tier caching**

Implement a three-tier cache for user profiles:
- L1: in-process Map (no network, microseconds)
- L2: Redis (sub-millisecond)
- L3: PostgreSQL (source of truth)
- Cache populates from L3 → L2 → L1 on miss
- Invalidation propagates from L3 → L2 → L1

*Hint:* L1 needs size limit (LRU eviction) and TTL. Consider using the `lru-cache` npm package.

**Exercise 3: Real-time leaderboard**

Build a leaderboard API using Redis sorted sets:
- `POST /scores` — submit a score for a user
- `GET /leaderboard?limit=10` — get top N users with scores
- `GET /leaderboard/me/:userId` — get user's rank + 2 users above and below ("near you")
- `GET /leaderboard/percentile/:userId` — return user's percentile rank
- Handle concurrent score updates atomically

*Hint:* Use `ZREVRANK` for rank, `ZCARD` for total count, `ZREVRANGE` with offset for "nearby" users.

---

## 12. Further Reading

- [Redis documentation](https://redis.io/docs/)
- [Redis data types guide](https://redis.io/docs/data-types/)
- [Redlock algorithm specification](https://redis.io/docs/manual/patterns/distributed-locks/)
- [Redis persistence documentation](https://redis.io/docs/manual/persistence/)
- [ioredis — TypeScript Redis client](https://github.com/redis/ioredis)
- [Martin Fowler — Cache-Aside Pattern](https://martinfowler.com/patterns/cacheAside.html)
- [Caching Best Practices (AWS)](https://aws.amazon.com/caching/best-practices/)
- [Redis University — free courses](https://university.redis.com/)
- Book: *Redis in Action* by Josiah Carlson
