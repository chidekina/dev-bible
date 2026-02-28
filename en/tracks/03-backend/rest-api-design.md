# REST API Design

## 1. What & Why

REST (Representational State Transfer) is an architectural style for distributed hypermedia systems, defined by Roy Fielding in his 2000 dissertation. In practice, "REST API" refers to an HTTP API that follows a set of conventions making it predictable, cacheable, and easy to consume by any client.

Good REST API design is a multiplier: a well-designed API is consumed correctly the first time, requires minimal documentation, fails loudly and informatively, and evolves without breaking existing clients. A poorly designed one generates support tickets, incorrect integrations, and painful versioning migrations.

The goal of this file is not just syntax rules but the reasoning behind each decision — so you can apply judgment when the rules conflict.

---

## 2. Core Concepts

### Resources, Not Actions

REST models the API as a collection of **resources** (nouns), not a collection of operations (verbs).

```
# Wrong — RPC-style, action-oriented
GET  /getUser?id=42
POST /createUser
POST /deleteUser?id=42
POST /activateUserAccount

# Correct — resource-oriented
GET    /users/42
POST   /users
DELETE /users/42
POST   /users/42/activations  (sub-resource representing the activation state)
```

**Naming conventions:**
- Use **plural nouns**: `/users`, `/orders`, `/products`
- Use **kebab-case** for multi-word resources: `/product-categories`
- Avoid verbs in paths; the HTTP method conveys the action
- Nested resources express relationships: `/users/42/orders` = orders belonging to user 42
- Keep nesting to **max 2 levels deep** — deeper nesting couples client to internal structure

```
# Acceptable
GET /users/42/orders
GET /orders/99/items

# Too deep — avoid
GET /users/42/orders/99/items/7/reviews
# Better: GET /items/7/reviews or GET /reviews?itemId=7
```

### HTTP Methods and Their Semantics

| Method | Safe | Idempotent | Common Use |
|--------|------|------------|------------|
| GET | Yes | Yes | Retrieve resource(s) |
| HEAD | Yes | Yes | Same as GET but no body (check existence, ETags) |
| POST | No | No | Create resource, trigger action |
| PUT | No | Yes | Replace resource entirely |
| PATCH | No | No* | Partial update |
| DELETE | No | Yes | Remove resource |

*PATCH is not inherently idempotent (e.g., increment a counter by 1), but many implementations make it so.

**Safe** means the request does not change server state — clients and caches can freely repeat it.
**Idempotent** means repeating the request N times produces the same result as doing it once — safe for retries.

**PUT vs PATCH:**

```json
// PUT /users/42 — replace the entire resource
// If fields are omitted, they are set to null/default
{
  "name": "Alice",
  "email": "alice@example.com",
  "role": "admin"
}

// PATCH /users/42 — partial update; only provided fields change
{
  "email": "newalice@example.com"
}
// name and role remain unchanged
```

> 💡 Use PUT when the client constructs the full resource representation. Use PATCH for field-level updates. PATCH is more common in real APIs because clients rarely want to send every field.

---

## 3. How It Works

### Status Codes: The API's Vocabulary

Correct status codes are the first thing clients check. Using 200 for errors is one of the most harmful API anti-patterns.

```
2xx — Success
  200 OK              General success, body contains result
  201 Created         Resource created; include Location header pointing to it
  204 No Content      Success, no body (DELETE, some PATCH responses)

3xx — Redirection
  301 Moved Permanently  Resource has a new permanent URL
  304 Not Modified       ETag/Last-Modified match; use cached version

4xx — Client Errors
  400 Bad Request         Malformed syntax, missing required fields
  401 Unauthorized        Authentication required or failed (misleading name — means "unauthenticated")
  403 Forbidden           Authenticated but not authorized for this resource
  404 Not Found           Resource does not exist
  405 Method Not Allowed  HTTP method not supported for this endpoint
  409 Conflict            State conflict (e.g., duplicate email, edit conflict)
  410 Gone                Resource existed but has been permanently deleted
  422 Unprocessable Entity Syntactically valid but semantically invalid (validation errors)
  429 Too Many Requests   Rate limit exceeded

5xx — Server Errors
  500 Internal Server Error Unexpected server failure
  502 Bad Gateway           Upstream service returned invalid response
  503 Service Unavailable   Overloaded or in maintenance
  504 Gateway Timeout       Upstream service timed out
```

> ⚠️ Never return 200 with `{ "status": "error" }` in the body. Status codes in the HTTP layer are checkable without parsing the body. Many HTTP clients, proxies, and monitoring tools rely on them.

### Error Response Format: RFC 7807 Problem Details

RFC 7807 defines a standard JSON error format. Adopting it makes your API interoperable with any client that understands the standard.

```json
{
  "type": "https://api.example.com/errors/validation-error",
  "title": "Validation Error",
  "status": 422,
  "detail": "The request body contains invalid fields.",
  "instance": "/api/users",
  "errors": [
    { "field": "email", "message": "Must be a valid email address" },
    { "field": "age", "message": "Must be a positive integer" }
  ]
}
```

Fields:
- `type` — URI that identifies the error type (dereferenceable = machine-readable docs)
- `title` — human-readable summary (same for all instances of this error type)
- `status` — HTTP status code (redundant but useful for logging)
- `detail` — human-readable explanation for this specific occurrence
- `instance` — URI of the specific request that caused the error

### API Versioning Strategies

```
Strategy 1: URL path versioning (most common)
  /v1/users
  /v2/users

  Pros: Explicit, easy to route, easy to test in browser
  Cons: Technically not RESTful (version is not part of resource identity)

Strategy 2: Accept header versioning
  GET /users
  Accept: application/vnd.myapi.v2+json

  Pros: Cleaner URLs, correct semantically
  Cons: Harder to test in browser/curl, harder to cache correctly

Strategy 3: Query parameter
  GET /users?version=2

  Pros: Simple to implement
  Cons: Easy to omit, pollutes query strings, proxy caching issues

Strategy 4: Custom header
  GET /users
  API-Version: 2024-01-01

  Pros: Date-based versioning is clear about when changes happened
  Cons: Non-standard, requires documentation
```

**Recommendation:** URL path versioning (`/v1/`) for public APIs (predictable, easy to document). Header versioning for internal APIs where URL cleanliness matters. Date-based headers (Stripe style) for APIs with incremental breaking changes.

### Pagination

**Offset/Limit pagination:**

```
GET /posts?offset=0&limit=20   → first page
GET /posts?offset=20&limit=20  → second page
GET /posts?offset=40&limit=20  → third page
```

Problems:
- Inconsistent results when items are inserted/deleted between pages (rows shift)
- Performance degrades at large offsets (e.g., `OFFSET 100000` in PostgreSQL scans 100,000 rows before returning 20)

**Cursor-based pagination (preferred for large datasets):**

```
GET /posts?limit=20
→ returns items + nextCursor

GET /posts?limit=20&cursor=eyJpZCI6IjEwMCJ9
→ returns next page
```

The cursor encodes a stable position (typically the last item's ID or timestamp), not a numeric offset.

**Response envelope:**

```json
{
  "data": [...],
  "meta": {
    "nextCursor": "eyJpZCI6IjEwMCJ9",
    "hasMore": true,
    "totalCount": 4200
  }
}
```

> 💡 Avoid returning `totalCount` when it requires a full table scan. Either omit it, compute it separately on demand, or use approximate counts (`SELECT reltuples FROM pg_class`).

### Filtering, Sorting, Sparse Fieldsets

```
# Filtering
GET /users?status=active&role=admin&createdAfter=2024-01-01

# Sorting (- prefix for descending)
GET /posts?sort=-createdAt,title

# Sparse fieldsets (GraphQL-lite for REST)
GET /users?fields=id,name,email

# Combined
GET /orders?status=pending&sort=-createdAt&limit=10&fields=id,total,customerId
```

---

## 4. Code Examples (TypeScript / Fastify)

### Zod Request Validation

```typescript
import Fastify from 'fastify';
import { z } from 'zod';

const app = Fastify({ logger: true });

// Schemas
const CreateUserBody = z.object({
  name: z.string().min(1).max(100),
  email: z.string().email(),
  role: z.enum(['user', 'admin']).default('user'),
  age: z.number().int().positive().optional(),
});

const UserParams = z.object({
  id: z.string().uuid(),
});

const ListUsersQuery = z.object({
  status: z.enum(['active', 'inactive']).optional(),
  limit: z.coerce.number().int().min(1).max(100).default(20),
  cursor: z.string().optional(),
  sort: z.string().optional(),
});

type CreateUserBody = z.infer<typeof CreateUserBody>;
```

### GET List with Cursor Pagination

```typescript
app.get('/v1/users', async (req, reply) => {
  const query = ListUsersQuery.safeParse(req.query);
  if (!query.success) {
    return reply.status(422).send({
      type: 'https://api.example.com/errors/validation',
      title: 'Validation Error',
      status: 422,
      detail: 'Invalid query parameters',
      errors: query.error.errors.map((e) => ({
        field: e.path.join('.'),
        message: e.message,
      })),
    });
  }

  const { limit, cursor, status, sort } = query.data;

  // Decode cursor (base64-encoded JSON with last seen id)
  let cursorId: string | undefined;
  if (cursor) {
    try {
      const decoded = JSON.parse(Buffer.from(cursor, 'base64url').toString('utf8'));
      cursorId = decoded.id;
    } catch {
      return reply.status(400).send({
        type: 'https://api.example.com/errors/invalid-cursor',
        title: 'Invalid Cursor',
        status: 400,
        detail: 'The provided cursor is malformed or expired.',
      });
    }
  }

  const users = await userService.list({ limit: limit + 1, cursorId, status, sort });

  const hasMore = users.length > limit;
  const items = hasMore ? users.slice(0, limit) : users;

  const nextCursor = hasMore
    ? Buffer.from(JSON.stringify({ id: items[items.length - 1].id })).toString('base64url')
    : null;

  return reply.status(200).send({
    data: items,
    meta: {
      nextCursor,
      hasMore,
      count: items.length,
    },
  });
});
```

### POST Create with 201 + Location Header

```typescript
app.post('/v1/users', async (req, reply) => {
  const body = CreateUserBody.safeParse(req.body);
  if (!body.success) {
    return reply.status(422).send({
      type: 'https://api.example.com/errors/validation',
      title: 'Validation Error',
      status: 422,
      detail: 'The request body contains invalid fields.',
      errors: body.error.errors.map((e) => ({
        field: e.path.join('.'),
        message: e.message,
      })),
    });
  }

  try {
    const user = await userService.create(body.data);

    return reply
      .status(201)
      .header('Location', `/v1/users/${user.id}`)
      .send({ data: user });
  } catch (err) {
    if (err instanceof DuplicateEmailError) {
      return reply.status(409).send({
        type: 'https://api.example.com/errors/conflict',
        title: 'Conflict',
        status: 409,
        detail: `A user with email ${body.data.email} already exists.`,
      });
    }
    throw err; // Let Fastify's error handler handle unexpected errors
  }
});
```

### PATCH Partial Update

```typescript
const UpdateUserBody = z.object({
  name: z.string().min(1).max(100).optional(),
  email: z.string().email().optional(),
  role: z.enum(['user', 'admin']).optional(),
}).refine(
  (data) => Object.keys(data).length > 0,
  { message: 'At least one field must be provided' }
);

app.patch('/v1/users/:id', async (req, reply) => {
  const params = UserParams.safeParse(req.params);
  if (!params.success) {
    return reply.status(400).send({
      type: 'https://api.example.com/errors/validation',
      title: 'Invalid ID',
      status: 400,
      detail: 'User ID must be a valid UUID.',
    });
  }

  const body = UpdateUserBody.safeParse(req.body);
  if (!body.success) {
    return reply.status(422).send({ /* RFC 7807 */ });
  }

  const user = await userService.update(params.data.id, body.data);
  if (!user) {
    return reply.status(404).send({
      type: 'https://api.example.com/errors/not-found',
      title: 'Not Found',
      status: 404,
      detail: `User ${params.data.id} does not exist.`,
    });
  }

  return reply.status(200).send({ data: user });
});
```

### DELETE with 204 No Content

```typescript
app.delete('/v1/users/:id', async (req, reply) => {
  const params = UserParams.safeParse(req.params);
  if (!params.success) {
    return reply.status(400).send({ /* validation error */ });
  }

  const deleted = await userService.delete(params.data.id);
  if (!deleted) {
    return reply.status(404).send({ /* not found */ });
  }

  return reply.status(204).send(); // No body on success
});
```

### Idempotency Keys for POST

```typescript
// Critical for payment APIs — POST /payments with idempotency key
// Same key = same result; safe to retry on network failure

app.post('/v1/payments', async (req, reply) => {
  const idempotencyKey = req.headers['idempotency-key'];
  if (!idempotencyKey || typeof idempotencyKey !== 'string') {
    return reply.status(400).send({
      type: 'https://api.example.com/errors/missing-idempotency-key',
      title: 'Missing Idempotency Key',
      status: 400,
      detail: 'The Idempotency-Key header is required for payment requests.',
    });
  }

  // Check if we already processed this key
  const cached = await redis.get(`idempotency:${idempotencyKey}`);
  if (cached) {
    return reply
      .status(200)
      .header('Idempotency-Replayed', 'true')
      .send(JSON.parse(cached));
  }

  const result = await paymentService.charge(req.body);

  // Store result with 24h TTL
  await redis.setex(
    `idempotency:${idempotencyKey}`,
    86400,
    JSON.stringify(result)
  );

  return reply.status(201).send(result);
});
```

### HATEOAS Example

```json
// Response from GET /v1/orders/99
{
  "data": {
    "id": "99",
    "status": "pending",
    "total": 149.99,
    "_links": {
      "self": { "href": "/v1/orders/99" },
      "cancel": { "href": "/v1/orders/99/cancellations", "method": "POST" },
      "pay": { "href": "/v1/payments", "method": "POST" },
      "customer": { "href": "/v1/users/42" }
    }
  }
}
```

> 💡 HATEOAS allows clients to navigate the API without hardcoding URLs — they follow links from response to response. It's rarely implemented fully in practice but the concept is useful: include relevant action URLs in responses so clients do not need out-of-band knowledge of your URL structure.

---

## 5. Common Mistakes & Pitfalls

**Using GET for operations with side effects:**

```
# Wrong — GET must be safe
GET /api/users/42/deactivate
GET /api/emails/send?to=user@example.com

# Correct
POST /api/users/42/deactivations
POST /api/emails
```

**Wrong status codes:**

```typescript
// Wrong — 200 for not-found hides the error from monitoring
res.status(200).json({ error: 'User not found' });

// Wrong — 500 for a client's bad input
res.status(500).json({ error: 'Email already taken' }); // should be 409

// Correct
res.status(404).json({ type: '...', title: 'Not Found', status: 404 });
res.status(409).json({ type: '...', title: 'Conflict', status: 409 });
```

**Leaking internal implementation in error messages:**

```json
// Wrong — exposes DB schema and internal errors
{
  "error": "ERROR:  duplicate key value violates unique constraint \"users_email_key\""
}

// Correct — abstract to domain-level message
{
  "type": "https://api.example.com/errors/conflict",
  "title": "Conflict",
  "status": 409,
  "detail": "An account with this email address already exists."
}
```

**Deep nesting of resources:**

```
# Avoid — couples client to internal structure
GET /companies/5/departments/12/employees/88/projects/3

# Better — use query parameters for filtering
GET /projects?employeeId=88
GET /employees/88/projects
```

**Inconsistent pluralization:**

```
# Wrong — mixed singular/plural
GET /user        vs GET /orders
GET /product/5   vs GET /carts/5/items

# Correct — always plural
GET /users
GET /orders
GET /products/5
GET /carts/5/items
```

---

## 6. When to Use / Not Use

**REST is a good choice when:**
- Building a public API with diverse consumers (mobile, web, third parties)
- Simple CRUD operations dominate the use case
- HTTP caching is important (CDN, browser caching)
- The API surface is stable and evolves slowly
- Consumers need to browse the API without SDK documentation

**Consider alternatives when:**
- Multiple clients need very different data shapes (→ GraphQL)
- High-frequency bidirectional communication (→ WebSockets/gRPC)
- Tight internal service communication with strict schemas (→ gRPC)
- Binary protocol performance is required (→ gRPC, MessagePack)
- Real-time event streaming (→ Server-Sent Events, Kafka + HTTP gateway)

---

## 7. Real-World Scenario

**Problem:** An e-commerce API has an `/api/order/checkout` POST endpoint that charges the customer's card. When the mobile app experiences a network timeout, it retries — resulting in double charges.

**Root cause:** The endpoint is not idempotent. On retry, a new charge is created.

**Solution: Idempotency keys with Redis-backed deduplication:**

```typescript
// Client sends: POST /v1/checkouts
// Header: Idempotency-Key: <uuid generated per checkout attempt>

// Server checks: have we already processed this key?
app.post('/v1/checkouts', async (req, reply) => {
  const key = req.headers['idempotency-key'] as string;
  if (!key) return reply.status(400).send(/* missing key error */);

  // Check for existing result
  const existing = await redis.get(`checkout:idempotency:${key}`);
  if (existing) {
    return reply
      .status(200)
      .header('Idempotency-Replayed', 'true')
      .send(JSON.parse(existing));
  }

  // Lock to prevent concurrent requests with same key
  const lock = await redis.set(
    `checkout:lock:${key}`,
    '1',
    'EX', 30,
    'NX' // set only if not exists
  );
  if (!lock) {
    return reply.status(409).send({
      type: 'https://api.example.com/errors/concurrent-request',
      title: 'Concurrent Request',
      status: 409,
      detail: 'A request with this idempotency key is already being processed.',
    });
  }

  try {
    const order = await checkoutService.process(req.body);
    const response = { data: order };

    // Cache result for 24 hours
    await redis.setex(`checkout:idempotency:${key}`, 86400, JSON.stringify(response));

    return reply.status(201).send(response);
  } finally {
    await redis.del(`checkout:lock:${key}`);
  }
});
```

**Result:** Retries return the same result without creating additional charges. The customer's payment is safe.

---

## 8. Interview Questions

**Q1: What are the six REST constraints defined by Fielding?**

(1) **Client-Server** — UI separated from data storage; independent evolution. (2) **Stateless** — each request contains all information needed; server stores no client context between requests. (3) **Cacheable** — responses must define themselves as cacheable or not. (4) **Uniform Interface** — resource identification in requests, manipulation through representations, self-descriptive messages, HATEOAS. (5) **Layered System** — client cannot tell whether it is connected directly to the server or through intermediaries. (6) **Code on Demand (optional)** — servers can transfer executable code (JavaScript) to clients.

**Q2: What is the difference between PUT and PATCH?**

PUT replaces the entire resource — if you omit a field, it is cleared. PATCH applies a partial update — only the provided fields change. PUT is idempotent (same payload = same result); PATCH is not inherently idempotent (e.g., PATCH to increment a counter). Use PUT when the client has the full resource; use PATCH for field-level updates, which is more common.

**Q3: What is the difference between cursor-based and offset-based pagination?**

Offset pagination (`?offset=20&limit=10`) skips N rows in the database — simple but inconsistent when rows are inserted/deleted between pages, and slow at large offsets (the DB must scan and discard offset rows). Cursor pagination uses a stable pointer to the last seen item (typically its ID or timestamp) — consistent regardless of concurrent inserts, and efficient because it uses an indexed WHERE clause rather than OFFSET. Use cursor pagination for large datasets or real-time data; offset for small admin lists where simplicity matters.

**Q4: How do you version a REST API? What are the trade-offs?**

URL path (`/v1/`): explicit, easy to route and test, widely understood, but technically includes version in resource identity. Header (`Accept: application/vnd.api.v2+json`): semantically correct, cleaner URLs, but harder to test and requires careful cache configuration. Query param (`?version=2`): simplest to implement but pollutes URLs and is easy to accidentally omit. My default: URL path versioning for public APIs because it is the most predictable for consumers.

**Q5: What is idempotency and why does it matter in APIs?**

Idempotency means applying an operation N times produces the same result as applying it once. GET, PUT, DELETE are idempotent by definition. POST is not — calling POST /payments twice creates two payments. Idempotency matters because networks fail and clients retry. For non-idempotent POST operations, implement idempotency keys: the client generates a unique key per operation; the server caches the result by key, returning the cached result on retries instead of re-executing.

**Q6: What is the difference between 401 and 403?**

401 Unauthorized (misleadingly named) means the request lacks valid authentication credentials — the server does not know who you are. 403 Forbidden means the server knows who you are but you do not have permission to access the resource. Practical test: 401 = "please log in first"; 403 = "you're logged in but not allowed here."

**Q7: What is RFC 7807 and why should you use it?**

RFC 7807 "Problem Details for HTTP APIs" defines a standard JSON error format with fields: `type` (URI identifying the error category), `title` (human-readable summary), `status` (HTTP status code), `detail` (specific explanation), `instance` (URI of the specific error occurrence). Using it makes errors machine-readable and interoperable — clients that understand RFC 7807 can handle errors from any compliant API without custom parsing.

**Q8: What is HATEOAS and is it used in practice?**

HATEOAS (Hypermedia as the Engine of Application State) means that API responses include links to related actions and resources, allowing clients to navigate the API without hardcoded URLs. Example: a GET /orders/99 response includes `_links.cancel` and `_links.pay`. In theory it enables API evolution without client changes. In practice, few production APIs implement full HATEOAS because it increases payload size and clients rarely follow links dynamically. The concept is still valuable: include the most relevant action URLs in responses to reduce client coupling to URL structure.

---

## 9. Exercises

**Exercise 1: Design a REST API for a blog**

Design the full URL structure for a blog with: users, posts, comments, tags, and likes. Define endpoints for: listing posts (with filtering by tag and author), creating a post, updating a post, deleting a post, adding a comment, getting comments for a post, liking a post. Specify the HTTP method and expected status codes for each.

*Hint:* Consider: `/posts`, `/posts/:id`, `/posts/:id/comments`, `/posts/:id/likes`. Should tags be a sub-resource or a query param? Why?

**Exercise 2: Implement cursor pagination**

Given a PostgreSQL table `posts(id UUID, title TEXT, created_at TIMESTAMPTZ)`, implement:
- A Fastify route `GET /posts` with `?limit` and `?cursor` query params
- Cursor encodes `{ id, createdAt }` as base64url JSON
- SQL that uses `WHERE (created_at, id) < ($cursor_created_at, $cursor_id)` for stable ordering
- Response envelope with `data`, `meta.nextCursor`, `meta.hasMore`

*Hint:* The tuple comparison `(a, b) < (c, d)` in PostgreSQL is equivalent to `a < c OR (a = c AND b < d)` — useful when timestamps can be equal.

**Exercise 3: Implement RFC 7807 error handler**

Build a Fastify error handler plugin that:
- Catches Zod validation errors and returns 422 with per-field error details
- Catches known domain errors (NotFoundError, ConflictError, ForbiddenError) and maps them to correct status codes
- Catches unknown errors and returns 500 without leaking stack traces
- Always returns RFC 7807 format

*Hint:* Use `fastify.setErrorHandler()`. Check `error instanceof ZodError` and use `error.errors` for field details.

---

## 10. Further Reading

- [Fielding's original REST dissertation (Chapter 5)](https://www.ics.uci.edu/~fielding/pubs/dissertation/rest_arch_style.htm)
- [RFC 7807 — Problem Details for HTTP APIs](https://www.rfc-editor.org/rfc/rfc7807)
- [RFC 9457 — Updated Problem Details (2023)](https://www.rfc-editor.org/rfc/rfc9457)
- [HTTP semantics RFC 9110](https://www.rfc-editor.org/rfc/rfc9110)
- [Stripe API — excellent real-world REST design](https://stripe.com/docs/api)
- [GitHub REST API — versioning + pagination](https://docs.github.com/en/rest)
- [Fastify documentation](https://fastify.dev/docs/latest/)
- [Zod documentation](https://zod.dev)
- Book: *REST API Design Rulebook* by Mark Masse
- Book: *Building Microservices* by Sam Newman (Chapter on APIs)
