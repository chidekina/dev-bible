# Backend Challenges

> Practical backend challenges set in a Node.js/TypeScript context using Fastify (or Express where noted). Each scenario reflects a real production situation. Focus on correctness first, then security, then performance. Stack assumed: Node.js 20+, TypeScript, Fastify, Zod, Prisma (or raw SQL), Redis.

---

## Exercise 1 — Design a RESTful Resource API (Easy)

**Scenario:** You are building the backend for a task management app. You need to expose a `tasks` resource with full CRUD operations.

**Requirements:**
- Endpoints: `GET /tasks`, `GET /tasks/:id`, `POST /tasks`, `PUT /tasks/:id`, `DELETE /tasks/:id`.
- A task has: `id`, `title` (required), `description` (optional), `status` (`todo | in_progress | done`), `dueDate` (ISO string, optional), `createdAt`.
- `GET /tasks` supports filtering by `status` and pagination (`page`, `limit` query params, default limit 20, max 100).
- All request bodies are validated with Zod schemas.
- Return appropriate HTTP status codes (201 on create, 204 on delete, 404 when not found, 422 on validation failure).

**Acceptance Criteria:**
- [ ] `POST /tasks` with a missing `title` returns `422` with a structured error body (field-level errors).
- [ ] `GET /tasks?status=invalid` returns `422`, not `500`.
- [ ] `DELETE /tasks/:id` on a non-existent resource returns `404`.
- [ ] `GET /tasks` response includes `{ data: Task[], meta: { page, limit, total } }`.
- [ ] All endpoints have route-level Zod schema registered with Fastify (not ad-hoc validation inside handlers).

**Hints:**
1. Register a Zod schema for each route via `fastify.route({ schema: { body: zodToJsonSchema(taskSchema) } })`.
2. Use a `taskService` layer — routes call service methods, service calls the repository. No business logic in route handlers.
3. For pagination: `SELECT * FROM tasks LIMIT $limit OFFSET ($page - 1) * $limit`. Always run a `COUNT(*)` query for the total.
4. Centralize 404/422 responses in a shared `reply.badRequest` / `reply.notFound` helper or a custom error class.

---

## Exercise 2 — JWT Authentication Flow (Medium)

**Scenario:** Implement a complete auth system: register, login, refresh token, and logout.

**Requirements:**
- `POST /auth/register`: creates a user with hashed password (bcrypt, cost 12). Returns `201`.
- `POST /auth/login`: validates credentials, returns `{ accessToken, refreshToken }`. Access token expires in 15 minutes, refresh token in 7 days.
- `POST /auth/refresh`: accepts a valid refresh token in the request body, returns a new access token.
- `POST /auth/logout`: invalidates the refresh token (blacklist in Redis with the token's remaining TTL).
- Protected routes require a valid `Authorization: Bearer <token>` header.

**Acceptance Criteria:**
- [ ] Passwords are never stored in plaintext — verify with a direct DB query.
- [ ] Access tokens are signed with `HS256` using an env-provided `JWT_SECRET`.
- [ ] Refresh tokens are stored in Redis with a TTL of 7 days.
- [ ] After `POST /auth/logout`, the refresh token cannot be used to obtain a new access token.
- [ ] Failed login (wrong password) returns `401`, not `403`.
- [ ] Brute-force protection: after 5 failed logins for the same email within 15 minutes, return `429`.

**Hints:**
1. Use `jsonwebtoken` (or `@fastify/jwt`) for token signing and verification.
2. Store the refresh token in Redis with `SET rt:<userId>:<tokenId> 1 EX 604800`. On refresh, verify the key exists. On logout, delete it.
3. For the brute-force counter: `INCR login_fail:<email>` with `EXPIRE login_fail:<email> 900`. Check the count before validating the password.
4. Never compare passwords with `===` — always use `bcrypt.compare`.

---

## Exercise 3 — Rate Limiting Middleware (Medium)

**Scenario:** Implement a sliding window rate limiter as a Fastify plugin using Redis. It should support per-IP and per-user limits with different thresholds.

**Requirements:**
- Anonymous (unauthenticated) requests: 60 requests per minute per IP.
- Authenticated requests: 300 requests per minute per user ID.
- Return `429 Too Many Requests` with `Retry-After` header when the limit is exceeded.
- Limits are stored and checked in Redis.
- The plugin must be configurable: `{ windowMs, maxRequests, keyPrefix }`.

**Acceptance Criteria:**
- [ ] A burst of 61 anonymous requests within 60 seconds results in the 61st being rejected with `429`.
- [ ] `Retry-After` header value (in seconds) is correct.
- [ ] Authenticated users have a separate, higher limit from anonymous users.
- [ ] Redis failure (connection error) fails open (allow the request) with a logged warning — do NOT fail the entire request.
- [ ] Unit test the rate limiter logic independently of Fastify.

**Hints:**
1. Sliding window with Redis: use a `ZADD` + `ZREMRANGEBYSCORE` + `ZCOUNT` pipeline. Add the current timestamp as both score and member, remove entries older than `windowMs`, count remaining.
2. `ZADD key NX score member` followed by `EXPIRE key windowSeconds` in a Lua script ensures atomicity.
3. Key format: `rl:<keyPrefix>:<identifier>` where identifier is IP or userId.
4. Fail open: wrap all Redis calls in `try/catch`. On error, log with `logger.warn` and call `next()`.

---

## Exercise 4 — Webhook Handler with Signature Verification (Medium)

**Scenario:** Your app receives webhooks from a payment provider (e.g., Stripe-like). Each request includes an `X-Webhook-Signature` header that you must verify before processing.

**Requirements:**
- `POST /webhooks/payment`: receives payment events.
- Signature: HMAC-SHA256 of the raw request body using a shared secret (`WEBHOOK_SECRET` env var). The header format is `sha256=<hex_digest>`.
- If the signature is invalid, return `401`. If the payload structure is invalid, return `422`.
- Process these event types: `payment.succeeded`, `payment.failed`, `payment.refunded`.
- On `payment.succeeded`: update the order status in the DB, send a confirmation email (async, non-blocking).
- Idempotency: if the same event ID is received twice, skip processing and return `200`.

**Acceptance Criteria:**
- [ ] Signature check uses a timing-safe comparison (`crypto.timingSafeEqual`) — not `===`.
- [ ] Raw body is read before any JSON parsing (Fastify parses body by default — configure to preserve raw bytes).
- [ ] The event ID idempotency key is stored in Redis with a 24-hour TTL.
- [ ] Email sending failure does not cause the webhook to return a non-200 response.
- [ ] Each event type is handled by a separate function in a `webhookHandlers` module.

**Hints:**
1. In Fastify: register `addContentTypeParser('application/json', { parseAs: 'buffer' }, ...)` to get the raw body for signature verification, then parse manually.
2. Signature check: `crypto.createHmac('sha256', secret).update(rawBody).digest('hex')`. Compare with `crypto.timingSafeEqual(Buffer.from(expected), Buffer.from(received))`.
3. Idempotency: `SET webhook:<eventId> 1 NX EX 86400`. If `SET` returns `null`, the event was already processed.
4. Fire-and-forget email: call `sendEmail(...)` without `await`. Catch and log any error internally.

---

## Exercise 5 — Database Query Optimization (Medium)

**Scenario:** A reporting endpoint is taking 8–12 seconds to respond. Your job is to diagnose and fix it.

**Starting query (PostgreSQL):**
```sql
SELECT
  u.name,
  u.email,
  COUNT(o.id) AS order_count,
  SUM(o.total_amount) AS lifetime_value,
  MAX(o.created_at) AS last_order_date
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
WHERE u.created_at > NOW() - INTERVAL '1 year'
  AND o.status = 'completed'
GROUP BY u.id
ORDER BY lifetime_value DESC
LIMIT 100;
```

**Requirements:**
- Identify why this query is slow (missing indexes, bad join order, filter placement, etc.).
- Rewrite or optimize it so it runs in under 200ms on a table with 1M users and 5M orders.
- Propose the indexes needed.
- Wrap the result in a materialized view or cache layer if necessary.

**Acceptance Criteria:**
- [ ] `EXPLAIN ANALYZE` shows Seq Scans replaced by Index Scans/Index Only Scans.
- [ ] The `WHERE o.status = 'completed'` filter is applied before the JOIN (push predicate down or use a partial index).
- [ ] At minimum, propose indexes on: `users(created_at)`, `orders(user_id, status)`, and explain the trade-offs.
- [ ] If you introduce a materialized view, explain the refresh strategy (on demand? scheduled?).
- [ ] The fix is tested against a seeded dataset of 1M+ rows.

**Hints:**
1. The query filters on `o.status = 'completed'` but joins first — this means all orders are loaded before filtering. A CTE or subquery that filters orders first can help.
2. `LEFT JOIN` with a `WHERE` clause on the right table turns it into an implicit `INNER JOIN` — make this explicit for clarity and optimizer hints.
3. A partial index `CREATE INDEX ON orders (user_id) WHERE status = 'completed'` dramatically reduces the index size and speeds up the join.
4. For dashboards accessed frequently: a materialized view refreshed every 15 minutes is often the pragmatic answer. `REFRESH MATERIALIZED VIEW CONCURRENTLY` avoids locking.

---

## Exercise 6 — Caching Layer for an Expensive API (Medium)

**Scenario:** Your app fetches currency exchange rates from an external API. The external API has a rate limit of 100 calls/hour. Your app serves 10,000 requests/minute that all need the latest rates.

**Requirements:**
- Implement a caching layer using Redis that serves the cached rates to all clients.
- Rates are cached for 5 minutes (the external API updates every 5 minutes).
- On cache miss: fetch from the external API, store in Redis, return to client.
- On external API failure: serve the last known good rates (stale-while-error), if available. If no cached data exists at all, return `503`.
- Expose a `GET /rates` endpoint and a `POST /rates/refresh` admin endpoint to force-refresh the cache.

**Acceptance Criteria:**
- [ ] Only one request hits the external API even if 1,000 concurrent requests arrive simultaneously on a cache miss (cache stampede prevention).
- [ ] Stale rates are served (with a warning header `X-Cache-Stale: true`) on external API failure.
- [ ] Cache key format: `rates:v1:current`. Changing the data shape bumps the version.
- [ ] `POST /rates/refresh` requires an `Admin-Key` header matching an env variable.
- [ ] A unit test mocks Redis and the external API, verifying the stale fallback behavior.

**Hints:**
1. Cache stampede: use a Redis lock (`SET lock:rates 1 NX EX 10`) before fetching. If the lock is taken, poll (with backoff) until the cache is populated by whoever holds the lock.
2. Stale fallback: use a second Redis key `rates:v1:stale` that is set with a 1-hour TTL. Always update it alongside the primary key. On error, read from the stale key.
3. `GET /rates` response: include `Cache-Control: public, max-age=60` so CDNs and browsers can also cache.
4. The cache key version lets you invalidate all old-format caches without manual flushing — just bump `v1` to `v2`.

---

## Exercise 7 — Background Job Queue (Medium)

**Scenario:** Your app needs to send welcome emails, resize uploaded images, and generate PDF reports asynchronously. These must not block HTTP responses.

**Requirements:**
- Implement a job queue using BullMQ (backed by Redis).
- Three job types: `welcome-email`, `image-resize`, `pdf-report`.
- Each job type has a dedicated worker with its own concurrency setting: email (5), image (2), pdf (1).
- Failed jobs are retried up to 3 times with exponential backoff (1s, 2s, 4s).
- After 3 failures, the job moves to a dead-letter queue (BullMQ "failed" state).
- A `GET /jobs/:id` endpoint returns the job status (waiting, active, completed, failed).

**Acceptance Criteria:**
- [ ] HTTP handler enqueues the job and returns `202 Accepted` with `{ jobId }` immediately.
- [ ] Workers are registered in a separate process (or worker file) — not in the Fastify server process.
- [ ] Exponential backoff is configured via BullMQ's `attempts` and `backoff` options, not a manual `setTimeout`.
- [ ] `GET /jobs/:id` works for all job types regardless of which queue they belong to.
- [ ] A test simulates job failure and verifies retry behavior without needing a real Redis (use `ioredis-mock`).

**Hints:**
1. BullMQ setup: one `Queue` per job type, one `Worker` per job type. Use `{ connection }` from a shared `ioredis` client.
2. Backoff config: `{ attempts: 3, backoff: { type: 'exponential', delay: 1000 } }`.
3. `GET /jobs/:id`: use BullMQ's `Job.fromId(queue, jobId)`. To search across all queues, try each queue in sequence until a match is found.
4. Keep worker code in `src/workers/<type>.worker.ts` files — each file is a standalone `Worker` registration.

---

## Exercise 8 — Multi-Tenant API with Row-Level Security (Hard)

**Scenario:** Build an API where multiple tenants (organizations) share the same database tables but their data is strictly isolated. A user belongs to one tenant and must never see another tenant's data.

**Requirements:**
- Tables: `tenants`, `users` (with `tenantId` FK), `projects` (with `tenantId` FK).
- Each API request is authenticated with a JWT that contains `userId` and `tenantId`.
- All queries automatically scope to the authenticated tenant — no service method should ever need to pass `tenantId` manually.
- Implement using PostgreSQL Row Level Security (RLS) policies.
- Admin users (role: `admin`) can see all tenants' data; regular users see only their tenant.

**Acceptance Criteria:**
- [ ] Direct DB query as `app_user` role with `SET app.tenant_id = X` returns only tenant X's rows.
- [ ] A user from tenant A cannot retrieve a project belonging to tenant B via any API endpoint, even by guessing the `id`.
- [ ] The `tenantId` from the JWT is set on the DB connection via `SET LOCAL app.tenant_id` before each query.
- [ ] RLS policies are created in a migration file (not applied manually).
- [ ] Integration test: two tenants, two users, verify cross-tenant isolation.

**Hints:**
1. In your Fastify `onRequest` hook: after JWT verification, run `SET LOCAL app.tenant_id = '<tenantId>'` on the connection.
2. RLS policy: `CREATE POLICY tenant_isolation ON projects USING (tenant_id = current_setting('app.tenant_id')::uuid)`.
3. Enable RLS: `ALTER TABLE projects ENABLE ROW LEVEL SECURITY; ALTER TABLE projects FORCE ROW LEVEL SECURITY;`
4. The `app_user` DB role must NOT be a superuser — superusers bypass RLS by default.
5. Admin override: create a separate `POLICY admin_override ... USING (current_setting('app.user_role') = 'admin')`.

---

## Exercise 9 — Real-Time Notifications with WebSockets (Medium)

**Scenario:** Build a notification system where users receive real-time alerts when their order status changes.

**Requirements:**
- WebSocket endpoint: `ws://host/ws` — clients connect with a JWT in the `Authorization` query param.
- When an order's status changes (via a separate REST endpoint), all WebSocket clients connected as that order's owner receive a `{ type: "order_update", orderId, status }` message.
- Support horizontal scaling: if the server runs on 3 instances, a status change on instance 1 must notify a client connected to instance 2.
- Clients that disconnect are removed from the connection registry.
- Heartbeat: send a ping every 30 seconds; close connections that don't pong within 10 seconds.

**Acceptance Criteria:**
- [ ] WebSocket clients are stored in a `Map<userId, WebSocket>` in-process.
- [ ] A Redis Pub/Sub channel (`order:updates`) bridges the gap between instances.
- [ ] On order update: publish to Redis; each instance's subscriber checks its local `Map` and sends to connected clients.
- [ ] Disconnected clients are removed from the map in the `'close'` event handler.
- [ ] Load test: 500 concurrent WebSocket connections with no memory leaks after 5 minutes.

**Hints:**
1. Use the `ws` package with Fastify (`@fastify/websocket`). Register the WebSocket plugin then add a `ws: true` route option.
2. JWT in query param: extract from `req.query.token`, verify with `jwt.verify`. Reject with close code `4001` if invalid.
3. Redis Pub/Sub: use two separate `ioredis` connections — one for `subscribe`, one for all other commands (a subscribed connection cannot issue other commands).
4. Heartbeat: use `ws.ping()` on an interval. On `pong` event, reset a `isAlive` flag. Before each ping, check `isAlive` — if false, terminate.

---

## Exercise 10 — Distributed Locking for Idempotent Operations (Hard)

**Scenario:** Your checkout service processes payments. If a user double-clicks "Pay", two requests arrive simultaneously. The second request must not charge the card twice.

**Requirements:**
- Implement a distributed lock using Redis (`SET NX EX`) around the payment processing logic.
- Lock key: `lock:payment:<orderId>`. TTL: 30 seconds (max expected payment duration).
- If the lock cannot be acquired, return `409 Conflict` with `{ message: "Payment already in progress" }`.
- After payment completes (success or failure), the lock is released immediately (not waited for TTL expiry).
- The lock release must be atomic (use a Lua script to avoid releasing another process's lock).

**Acceptance Criteria:**
- [ ] Two simultaneous requests for the same `orderId` result in exactly one successful payment attempt.
- [ ] The lock is released on success, failure, and unexpected exception (use try/finally).
- [ ] The Lua script for release: only delete the key if its value matches the lock token (UUID set at acquire time).
- [ ] If the payment takes longer than 30 seconds (edge case), handle gracefully — log a warning, do not crash.
- [ ] Integration test using real Redis verifies the race condition scenario.

**Hints:**
1. Acquire: `SET lock:payment:<orderId> <uuid> NX EX 30`. Returns `"OK"` on success, `null` if already held.
2. Release Lua script:
   ```lua
   if redis.call("GET", KEYS[1]) == ARGV[1] then
     return redis.call("DEL", KEYS[1])
   else
     return 0
   end
   ```
   Call with `redis.eval(script, 1, lockKey, lockToken)`.
3. The UUID (lock token) ensures you only release your own lock, not one reacquired by another process after your TTL expired.
4. `try { await processPayment() } finally { await releaseLock() }` — never skip the release.

---

## Exercise 11 — API Versioning Strategy (Easy)

**Scenario:** Your public API is in use by 500 external clients. You need to introduce breaking changes to three endpoints without forcing all clients to migrate immediately.

**Requirements:**
- Support two versions simultaneously: `v1` (current) and `v2` (new).
- Versioning via URL path prefix: `/api/v1/...` and `/api/v2/...`.
- The changed endpoints in v2: `GET /users` response shape changes (`firstName`/`lastName` split into a `name` object), and `POST /orders` now requires a `currencyCode` field.
- v1 endpoints continue to work unchanged for at least 6 months.
- Add a `Deprecation` response header on all v1 endpoints.
- Document the migration guide in a route-level JSDoc comment.

**Acceptance Criteria:**
- [ ] `GET /api/v1/users` still returns `{ name: "John Doe" }`.
- [ ] `GET /api/v2/users` returns `{ name: { first: "John", last: "Doe" } }`.
- [ ] `POST /api/v1/orders` accepts a body without `currencyCode`.
- [ ] `POST /api/v2/orders` returns `422` if `currencyCode` is missing.
- [ ] v1 responses include `Deprecation: true` and `Sunset: <date 6 months from now>` headers.

**Hints:**
1. Fastify: register two router prefixes with `fastify.register(v1Routes, { prefix: '/api/v1' })` and `fastify.register(v2Routes, { prefix: '/api/v2' })`.
2. Share the service layer — v1 and v2 routes call the same service methods; the difference is in request parsing and response transformation.
3. Deprecation headers: add via a plugin hook that runs on all v1 routes: `reply.header('Deprecation', 'true').header('Sunset', sunsetDate)`.
4. Sunset date: `new Date(Date.now() + 180 * 24 * 60 * 60 * 1000).toUTCString()`.

---

## Exercise 12 — Structured Logging and Observability (Easy)

**Scenario:** A microservice running in production has no useful logs. Requests fail silently and debugging takes hours. Add structured logging, request tracing, and error monitoring.

**Requirements:**
- Use Pino as the logger (it is built into Fastify).
- Every request logs: `{ method, url, statusCode, responseTime, requestId }`.
- Every error logs: `{ error: { message, stack, name }, requestId, userId (if authenticated) }`.
- Assign a unique `X-Request-ID` to every request (use the header if provided by a gateway, otherwise generate with `crypto.randomUUID()`).
- Add a health check endpoint `GET /health` that returns `{ status: "ok", uptime, version }` — this endpoint must NOT be logged (too noisy).
- Log levels: `info` for requests, `warn` for 4xx, `error` for 5xx.

**Acceptance Criteria:**
- [ ] Every log line is valid JSON (Pino's default output).
- [ ] The `requestId` is the same across all log lines for a given HTTP request.
- [ ] `GET /health` produces no log output.
- [ ] A 500 error log includes the full stack trace.
- [ ] Log level can be changed at runtime via `LOG_LEVEL` env variable without restarting.

**Hints:**
1. Fastify's built-in Pino logger: `fastify({ logger: { level: process.env.LOG_LEVEL ?? 'info' } })`.
2. Request ID: configure `fastify({ genReqId: () => crypto.randomUUID() })` and use `req.id` everywhere.
3. Suppress health check logs: in `fastify.addHook('onSend', ...)`, check `req.url === '/health'` and skip logging. Or configure `disableRequestLogging: true` on that specific route.
4. Log level by status: in `onResponse` hook, use `req.log.warn(...)` for 4xx and `req.log.error(...)` for 5xx, based on `reply.statusCode`.
