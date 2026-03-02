# System Design Challenges

> System design katas to sharpen your distributed systems thinking. Each exercise presents a concrete implementation or design problem, not just a vague "design Twitter" prompt. Focus on trade-offs, failure modes, and operational reality. Skill level: intermediate to advanced.
>
> For each kata: spend 20–30 minutes sketching a design before reading the hints. Write down your assumptions explicitly.

---

## Exercise 1 — Distributed Rate Limiter (Token Bucket) (Hard)

**Scenario:** Design and implement a distributed token bucket rate limiter. It must enforce limits across multiple API gateway nodes sharing no local state.

**Requirements:**
- Each client gets 100 tokens per minute. Tokens refill continuously (not in batches at the start of each minute).
- The limiter runs as a shared Redis-backed service. Multiple gateway nodes call it on every request.
- Return the number of remaining tokens and the time until the next token is available in response headers.
- Implement the core algorithm as an atomic Lua script to prevent race conditions.
- Support burst: a client can consume up to 20 tokens in a single request.

**Acceptance Criteria:**
- [ ] Two gateway nodes concurrently consuming the last token result in exactly one success (atomicity).
- [ ] Tokens refill at 100/60 ≈ 1.67 tokens/second (continuous, not per-minute batch).
- [ ] `X-RateLimit-Remaining` and `X-RateLimit-Reset` headers are returned on every response.
- [ ] A client that was throttled can resume as soon as tokens are available (no fixed window penalty).
- [ ] Lua script is unit-testable by injecting a fake `redis.call` function.

**Discussion questions:**
1. Token bucket vs. sliding window: which is fairer under bursty traffic?
2. What happens if the Redis Lua script execution time exceeds 5ms? How do you handle Redis latency spikes?
3. How would you add per-endpoint limits (different buckets for `/api/upload` vs. `/api/read`) without duplicating logic?

**Hints:**
1. Token bucket in Redis: store `{ tokens, lastRefillTimestamp }` as a hash. On each call, compute elapsed time, add `elapsed * refillRate` tokens (capped at max), then subtract requested tokens.
2. Lua ensures atomicity. The script takes: `KEYS[1]` = bucket key, `ARGV[1]` = now (ms), `ARGV[2]` = refill rate, `ARGV[3]` = capacity, `ARGV[4]` = requested tokens.
3. `X-RateLimit-Reset`: compute `(requested - current_tokens) / refillRate` seconds from now.

---

## Exercise 2 — Notification Fanout System (Medium)

**Scenario:** Design a notification fanout service that sends a single notification event to millions of users via push, email, and SMS simultaneously.

**Requirements:**
- A producer service publishes a `NotificationEvent` to a queue with: `{ eventId, recipients: string[], channels: ['push','email','sms'], payload }`.
- Recipients can be up to 1 million users (e.g., a marketing blast).
- Each channel has an independent worker pool.
- User preferences (opted-out channels) must be checked before sending.
- Failed sends are retried up to 3 times per channel per recipient with exponential backoff.

**Acceptance Criteria:**
- [ ] A single event with 1M recipients is fully delivered within 10 minutes.
- [ ] An opted-out user receives no notification on their opted-out channel.
- [ ] A push delivery failure does not affect email delivery for the same recipient.
- [ ] Dead-lettered messages (exhausted retries) are stored and observable.
- [ ] The system handles duplicate `eventId` submissions idempotently.

**Discussion questions:**
1. How do you fan out to 1M recipients without loading all user records into memory at once?
2. Should user preference checks happen before or after enqueuing per-recipient jobs? What are the trade-offs?
3. How do you rate-limit outbound channels (e.g., Twilio's 100 SMS/sec limit) without losing messages?

**Hints:**
1. Fan-out pattern: the intake service splits the recipient list into batches of 1,000 and enqueues one job per batch. Each batch worker expands, checks preferences, and enqueues per-channel jobs.
2. Preference check at the batch level: fetch preferences for all 1,000 users in one DB query (`WHERE user_id IN (...)`) before enqueueing per-channel jobs.
3. Channel rate limiting: use a separate per-channel queue with a concurrency cap that matches the provider's rate limit.

---

## Exercise 3 — Job Queue with Retry and Dead-Letter (Medium)

**Scenario:** Build a production-grade job queue system with retry logic, exponential backoff, and a dead-letter queue for failed jobs.

**Requirements:**
- Jobs have types, payloads, priority levels (`high`, `normal`, `low`), and a `maxAttempts` setting.
- On failure, jobs are retried with exponential backoff: `delay = baseDelay * 2^(attempt - 1)` (base: 1 second, max: 5 minutes).
- After `maxAttempts`, the job moves to a dead-letter queue (DLQ) and triggers an alert.
- The queue must support at-least-once delivery (no silent drops).
- Expose a `GET /jobs/:id` endpoint to inspect status, attempts, and failure reasons.

**Acceptance Criteria:**
- [ ] A job that fails 3 times with `maxAttempts=3` ends up in the DLQ after the 3rd failure.
- [ ] Retry delays are approximately 1s, 2s, 4s (exponential) — not fixed intervals.
- [ ] Restarting a worker mid-job does not result in the job being silently dropped.
- [ ] DLQ entries include: `jobId`, `jobType`, `failureReason`, `attempts`, `lastAttemptAt`.
- [ ] A test simulates a worker crash (process.exit) and verifies the job is re-queued.

**Discussion questions:**
1. What is the difference between at-least-once and exactly-once delivery? When is each appropriate?
2. How do you prevent a bad job (e.g., malformed payload) from blocking a queue indefinitely?
3. When should a job be sent to the DLQ immediately (without retries)?

**Hints:**
1. BullMQ handles most of this: `{ attempts: maxAttempts, backoff: { type: 'exponential', delay: 1000 } }`.
2. Worker crash recovery: BullMQ uses a Redis lock per job. If a worker dies, the lock expires and another worker picks up the job.
3. DLQ: listen for BullMQ's `'failed'` event on the queue after `maxAttempts` exhausted. Move job to a separate `dead-letter` queue and send alert.

---

## Exercise 4 — Read-Heavy Leaderboard with Caching (Medium)

**Scenario:** Design a leaderboard for a mobile game with 5 million daily active users. The leaderboard shows the top 100 players and the rank of the requesting player.

**Requirements:**
- Players earn points via in-game actions. Points are updated in real time (up to 10,000 updates/second).
- `GET /leaderboard/top100`: returns the top 100 players with rank, username, and score.
- `GET /leaderboard/me`: returns the requesting player's rank and score.
- Top 100 query must respond in < 50ms at any scale.
- Player rank must be accurate within 5 seconds of a score update.

**Acceptance Criteria:**
- [ ] Top 100 is served from a Redis sorted set, not a DB query.
- [ ] Score updates are written to both PostgreSQL (source of truth) and Redis (sorted set) in the same operation.
- [ ] `ZREVRANK` is used for efficient rank lookup — no manual counting.
- [ ] A player's rank changes are reflected in `GET /leaderboard/me` within 5 seconds.
- [ ] Load test: 10,000 concurrent `GET /leaderboard/top100` requests are served in < 50ms p99.

**Discussion questions:**
1. What happens to the Redis sorted set if the Redis instance is restarted? How do you rebuild it?
2. How do you handle ties (two players with the same score)? Does rank order matter?
3. At 10,000 updates/second, is writing to both Redis and PostgreSQL synchronously sustainable? What is the alternative?

**Hints:**
1. Redis sorted set: `ZADD leaderboard <score> <userId>`. `ZREVRANGE leaderboard 0 99 WITHSCORES` for top 100. `ZREVRANK leaderboard <userId>` for player rank (0-indexed).
2. Rebuild on restart: a startup script that runs `SELECT user_id, score FROM scores ORDER BY score DESC` and bulk-loads with `ZADD` pipeline.
3. For high write throughput: write to Redis synchronously (fast, in-memory), write to PostgreSQL asynchronously via a BullMQ job. Accept eventual consistency for the DB.

---

## Exercise 5 — Multi-Tenant SaaS Schema with Row-Level Security (Hard)

**Scenario:** Design a PostgreSQL schema for a multi-tenant SaaS application. Tenants (organizations) must be strictly isolated — no data leakage between tenants is acceptable.

**Requirements:**
- All tenant-scoped tables have a `tenant_id UUID NOT NULL` column.
- PostgreSQL Row Level Security (RLS) policies enforce isolation at the database layer (not just application layer).
- An admin user (role: `superadmin`) can query across all tenants.
- The application connects as a single DB user (`app_user`) and sets `app.tenant_id` on the connection before each query.
- Migrations must include RLS policy creation (reproducible, not manual).

**Acceptance Criteria:**
- [ ] A query run as `app_user` with `SET app.tenant_id = 'tenant-A'` cannot return rows belonging to `tenant-B`.
- [ ] `superadmin` queries bypass RLS (verified by a direct DB test).
- [ ] The migration that creates the `projects` table also enables RLS and creates the isolation policy.
- [ ] Integration test: two tenants, three projects each — verify cross-tenant isolation via the API.
- [ ] The application layer never manually filters by `tenant_id` in service methods — RLS handles it.

**Discussion questions:**
1. What are the performance implications of RLS at scale? How does it affect query plans?
2. What happens if a developer forgets to set `app.tenant_id` before a query? How do you make this a safe-by-default mistake?
3. How do you handle tenant-agnostic tables (e.g., `countries`, `currencies`) with RLS enabled?

**Hints:**
1. RLS policy: `CREATE POLICY tenant_isolation ON projects FOR ALL TO app_user USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
2. Enable: `ALTER TABLE projects ENABLE ROW LEVEL SECURITY; ALTER TABLE projects FORCE ROW LEVEL SECURITY;`.
3. Safe default: set `app.tenant_id = ''` as the default for `app_user`. An empty string never matches a real UUID, so a forgotten `SET` results in empty results (safe) rather than leaking data.
4. Superadmin bypass: create a separate DB role `superadmin_user` with `BYPASSRLS` attribute.

---

## Exercise 6 — Circuit Breaker Pattern (Medium)

**Scenario:** Your service calls an external payment API. When the payment API is degraded, your service cascades failures and becomes slow. Implement a circuit breaker to fail fast and recover gracefully.

**Requirements:**
- Implement a `CircuitBreaker` class with three states: `CLOSED` (normal), `OPEN` (failing fast), `HALF_OPEN` (testing recovery).
- Threshold: open after 5 consecutive failures within a 30-second window.
- Recovery: after 60 seconds in `OPEN` state, transition to `HALF_OPEN`. Allow one probe request. If it succeeds, close the circuit. If it fails, return to `OPEN`.
- When `OPEN`, calls immediately throw a `CircuitOpenError` without hitting the external service.
- Emit events on state transitions for observability.

**Acceptance Criteria:**
- [ ] After 5 consecutive failures, the circuit opens and subsequent calls fail immediately.
- [ ] While `OPEN`, zero network requests are made to the external service (verified by a mock).
- [ ] After the recovery timeout, exactly one probe request is allowed through.
- [ ] A successful probe closes the circuit; a failed probe resets the recovery timer.
- [ ] State transitions emit `{ from, to, timestamp }` events that a logger or metrics system can consume.

**Discussion questions:**
1. Should the circuit breaker be per-instance or a shared singleton? What are the trade-offs in a multi-node deployment?
2. What is the difference between a circuit breaker and a retry? When should you use both together?
3. How do you test a circuit breaker without introducing flakiness in your test suite?

**Hints:**
1. State machine: `CLOSED → OPEN` (on threshold); `OPEN → HALF_OPEN` (on timeout); `HALF_OPEN → CLOSED` (on success) or `HALF_OPEN → OPEN` (on failure).
2. Store: `failureCount`, `lastFailureTime`, `state`, `halfOpenProbeAllowed` (boolean).
3. Use `EventEmitter` for state transition events. Consumers can log, send to Datadog, or trigger alerts.
4. In tests: use fake timers (`vi.useFakeTimers`) to advance the recovery timeout without actually waiting 60 seconds.

---

## Exercise 7 — Idempotent Payment API (Hard)

**Scenario:** Your payment API must be idempotent: if a client retries a request (due to a network timeout), it must not double-charge.

**Requirements:**
- `POST /payments` accepts an `Idempotency-Key` header (UUID, client-generated).
- If the key has been seen before and the request completed, return the cached response (same status code + body).
- If the key is currently being processed, return `409 Conflict` — do not process a second time.
- Idempotency keys expire after 24 hours.
- The key must be tied to the authenticated user — keys are not shared across users.

**Acceptance Criteria:**
- [ ] Sending `POST /payments` twice with the same key and same user returns identical responses.
- [ ] The payment processor (Stripe, etc.) is only called once per idempotency key.
- [ ] A second request arriving while the first is processing returns `409`, not a duplicate charge.
- [ ] Using the same key with a different user returns `403` (key isolation per user).
- [ ] Keys older than 24 hours are treated as new (no cached response returned).

**Discussion questions:**
1. Where do you store idempotency keys — Redis or PostgreSQL? What are the durability trade-offs?
2. What happens if the server crashes after charging but before storing the idempotency response? How do you handle this partial write?
3. Should idempotency keys be enforced at the HTTP layer or the service layer? What are the architectural implications?

**Hints:**
1. Key format in Redis: `idempotency:<userId>:<key>`. Value: `{ status, body }` as JSON. TTL: 86400 seconds.
2. Lock during processing: `SET idempotency:<userId>:<key>:lock 1 NX EX 30`. If the lock exists (409 scenario), return `409`. Release lock with Lua script after storing the result.
3. Partial write recovery: store the payment intent ID from Stripe before the `SET` — if a crash occurs, a recovery job can reconcile by checking Stripe for the intent status.

---

## Exercise 8 — Real-Time Presence System (Medium)

**Scenario:** Design the "who's online" feature for a collaborative document editor. Users must see which team members are currently viewing the same document.

**Requirements:**
- When a user opens a document, they become "present" on that document.
- Presence expires after 30 seconds of inactivity (heartbeat keeps it alive).
- `GET /documents/:id/presence` returns `[{ userId, name, avatarUrl, lastSeen }]` for currently present users.
- When a user's presence changes (joins/leaves), all other present users receive a real-time update via WebSocket.
- Support up to 50 concurrent users per document.

**Acceptance Criteria:**
- [ ] A user closing their browser tab is removed from the presence list within 30 seconds (TTL expiry).
- [ ] Two users on the same document both see a `presence_update` event when a third user joins.
- [ ] Heartbeat endpoint: `POST /documents/:id/heartbeat` resets the 30-second TTL.
- [ ] Horizontal scaling: presence updates on server A reach clients connected to server B (via Redis Pub/Sub).
- [ ] `GET /documents/:id/presence` is served from Redis — no DB query on each call.

**Discussion questions:**
1. What is the trade-off between TTL-based expiry and explicit "user left" events?
2. How do you handle the case where a user's browser crashes (no WebSocket close event)?
3. At 50 users per document, is broadcasting via Redis Pub/Sub a bottleneck? At what scale would you redesign?

**Hints:**
1. Redis key: `presence:<docId>:<userId>` with 30-second TTL. Value: `{ name, avatarUrl }` as JSON. `KEYS presence:<docId>:*` to list all present users (or use a Redis Set for the document).
2. Better pattern: use a Redis Set `presence:doc:<docId>` containing `userId`s, plus individual keys for user metadata. Remove from the set on expiry using keyspace notifications.
3. WebSocket fanout: on heartbeat, publish `{ type: 'presence_join', docId, userId }` to `presence:updates:<docId>`. Each WS server subscribes and delivers to local clients.

---

## Exercise 9 — Cache Invalidation Strategy for a Product Catalog (Medium)

**Scenario:** A product catalog API serves 50,000 requests/second. Product data rarely changes (< 100 updates/day). Design a cache invalidation strategy that keeps data fresh without overloading the origin DB.

**Requirements:**
- `GET /products/:id`: serve from cache with < 10ms latency.
- When a product is updated, all cached versions (in Redis and in the CDN) must be invalidated within 60 seconds.
- Cache layers: CDN (Cloudflare) → Redis (application cache) → PostgreSQL.
- Product data changes trigger invalidation from an internal admin API.
- Handle cache stampede on invalidation (many clients requesting the same key simultaneously after eviction).

**Acceptance Criteria:**
- [ ] `GET /products/123` after an update returns the new data within 60 seconds.
- [ ] A cache invalidation event triggers: Redis key deletion + Cloudflare cache purge via API.
- [ ] Cache stampede is prevented: only one DB query fires per cold key, regardless of concurrency.
- [ ] Cache hit ratio remains above 99% under normal load (no spurious invalidations).
- [ ] Integration test: update a product, wait 5 seconds, verify `GET /products/:id` returns updated data.

**Discussion questions:**
1. TTL-based expiry vs. explicit invalidation on write — what are the consistency trade-offs?
2. How do you handle a cascade: a product update also affects category listings and search results? How do you track all dependent cache keys?
3. If Cloudflare purge API is slow (2–5 second latency), how do you ensure clients don't serve stale data in the meantime?

**Hints:**
1. Invalidation flow: `PUT /admin/products/:id` → update DB → publish `product:updated:<id>` to Redis Pub/Sub → subscriber deletes `product:cache:<id>` + calls Cloudflare purge API.
2. Stampede prevention: probabilistic early expiration, or a Redis lock on cache misses — `SET product:lock:<id> 1 NX EX 5`. If the lock exists, wait briefly and retry from cache.
3. Cloudflare purge: `POST https://api.cloudflare.com/client/v4/zones/<zoneId>/purge_cache` with `{ files: ['https://api.yourdomain.com/products/123'] }`.

---

## Exercise 10 — Webhook Delivery System with Retries (Medium)

**Scenario:** Your platform sends webhooks to customer-defined URLs when events occur. Customers' servers are unreliable — deliveries must be retried reliably.

**Requirements:**
- When an event fires, enqueue a webhook delivery job to the registered endpoint URL.
- Delivery attempts: immediate, then +1m, +5m, +30m, +2h, +8h (6 total attempts).
- If all 6 attempts fail, mark the webhook as permanently failed and notify the customer.
- Record every delivery attempt: `{ timestamp, httpStatus, responseBody, durationMs }`.
- Expose `GET /webhooks/:id/deliveries` to show the attempt history.

**Acceptance Criteria:**
- [ ] A customer endpoint returning `500` triggers a retry after the correct delay.
- [ ] A `200` or `201` response marks the delivery as succeeded — no further retries.
- [ ] After 6 failed attempts, an email is sent to the customer's registered address.
- [ ] Delivery logs are stored in PostgreSQL (not only in the job queue).
- [ ] A test with a mock HTTP server verifies the retry schedule is correct.

**Discussion questions:**
1. Should you retry on `4xx` responses (e.g., `401 Unauthorized`)? What is the reasoning?
2. How do you protect against a slow customer endpoint blocking your worker threads?
3. How do you allow customers to replay a specific failed webhook without re-triggering the original event?

**Hints:**
1. Use BullMQ with custom delay scheduling: instead of built-in exponential backoff, enqueue the next attempt as a new job with a fixed delay after each failure.
2. Store delivery attempts: in the job's `onFailed` and `onCompleted` handlers, insert a row into `webhook_attempts(webhookId, attempt, status, responseStatus, durationMs)`.
3. Timeout per attempt: use `AbortSignal.timeout(5000)` with `fetch` to cap each delivery attempt at 5 seconds.
4. Do not retry on `410 Gone` or `4xx` responses that indicate the endpoint is intentionally rejecting the request.
