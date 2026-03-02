# Performance Challenges

> Hands-on performance engineering exercises covering profiling, database optimization, caching, frontend rendering, and infrastructure tuning. Each exercise reflects a real production performance problem. Stack assumed: Node.js 20+, TypeScript, Fastify, Prisma, React, Redis. Target skill level: intermediate to advanced.

---

## Exercise 1 — Profile a Slow Node.js Endpoint with clinic.js (Medium)

**Scenario:** A `GET /reports/summary` endpoint takes 3–8 seconds. You have no idea why. Use clinic.js to profile it and identify the bottleneck.

**Requirements:**
- Install `clinic` globally and profile the endpoint under load using `clinic doctor`.
- Identify whether the bottleneck is: CPU-bound computation, I/O waiting, event loop blocking, or memory pressure.
- Produce a `clinic.js` flamegraph or bubbleprof report and annotate the top 3 hotspots.
- Fix the identified bottleneck and measure the before/after response time.
- Document the root cause and fix in a short write-up (3–5 sentences).

**Acceptance Criteria:**
- [ ] `clinic doctor -- node dist/server.js` runs without errors and produces an HTML report.
- [ ] The HTML report clearly identifies the bottleneck type (I/O, CPU, etc.).
- [ ] The fix reduces p99 response time by at least 50%.
- [ ] The write-up names the specific function/module responsible.
- [ ] Load during profiling uses `autocannon -c 10 -d 30 http://localhost:3000/reports/summary`.

**Hints:**
1. Common culprits: synchronous `JSON.parse` on large payloads, missing `await` on DB calls (starving the event loop), blocking `fs.readFileSync` in hot paths, n+1 queries.
2. `clinic flame` gives a CPU flamegraph — best for CPU-bound issues.
3. `clinic bubbleprof` visualizes async operations — best for I/O waits.
4. After finding the hotspot, check: can the computation be moved offline (pre-computed)? Can I/O be parallelized with `Promise.all`?

---

## Exercise 2 — Fix N+1 Queries in a Prisma Schema (Medium)

**Scenario:** A `GET /posts` endpoint lists blog posts with their authors. Each page load triggers 21 SQL queries (1 for posts + 1 per post for author). Fix it.

**Given code (N+1 problem):**
```typescript
async function getPosts(page: number) {
  const posts = await prisma.post.findMany({
    skip: (page - 1) * 20,
    take: 20,
    orderBy: { createdAt: 'desc' },
  });

  return Promise.all(
    posts.map(async post => ({
      ...post,
      author: await prisma.user.findUnique({ where: { id: post.authorId } }),
    }))
  );
}
```

**Requirements:**
- Rewrite `getPosts` to issue at most 2 SQL queries for any page size.
- Do not change the function's return type.
- Add Prisma query logging to count queries before and after the fix.
- Write a test that asserts the fixed version calls `prisma.user.findUnique` zero times.

**Acceptance Criteria:**
- [ ] Prisma query log shows 1–2 queries per call (not 21).
- [ ] The response shape is identical to the original (posts array with nested `author` objects).
- [ ] Fix uses Prisma's `include` or `select` — no raw SQL.
- [ ] A test mocks `prisma` and verifies `findUnique` is never called.
- [ ] `EXPLAIN ANALYZE` shows no sequential scan on the `users` table per post.

**Hints:**
1. Fix: use `include: { author: true }` in `findMany` — Prisma issues a single JOIN query.
2. Alternatively, use `select` to fetch only needed fields: `select: { id: true, title: true, author: { select: { id: true, name: true } } }`.
3. Prisma query logging: `new PrismaClient({ log: ['query'] })`. Count `query` events in tests.
4. For very large result sets, `include` may still be slow — consider a raw SQL query with a manual JOIN for performance-critical endpoints.

---

## Exercise 3 — Implement Cursor-Based Pagination (Medium)

**Scenario:** Your API uses offset pagination (`LIMIT x OFFSET y`). At page 500 with 20 items/page, the DB scans 10,000 rows to skip. Replace it with cursor-based pagination.

**Requirements:**
- Replace `GET /posts?page=N&limit=20` with `GET /posts?cursor=<id>&limit=20`.
- The cursor is the `id` of the last item returned on the previous page (opaque to the client — base64-encode it).
- Response format: `{ data: Post[], nextCursor: string | null }`. `nextCursor` is `null` on the last page.
- The query must use `WHERE id > cursor` — not `OFFSET`.
- Backward pagination is not required.

**Acceptance Criteria:**
- [ ] `GET /posts` (no cursor) returns the first 20 posts and a `nextCursor`.
- [ ] `GET /posts?cursor=<nextCursor>` returns the next 20 posts with a different `nextCursor`.
- [ ] The last page returns `nextCursor: null`.
- [ ] `EXPLAIN ANALYZE` shows an index scan using the primary key — no sequential scans.
- [ ] A test verifies that paginating through all 100 items (5 pages × 20) yields each item exactly once.

**Hints:**
1. Query: `WHERE id > decodeCursor(cursor) ORDER BY id ASC LIMIT limit + 1`. Fetch `limit + 1` rows — if you get `limit + 1`, there is a next page; set `nextCursor` to the `id` of item `limit + 1`, then return only the first `limit` items.
2. Cursor encoding: `Buffer.from(String(id)).toString('base64')`. Decode: `parseInt(Buffer.from(cursor, 'base64').toString('utf8'))`.
3. The `ORDER BY id` must match the index — add `CREATE INDEX ON posts(id)` if not already the primary key.
4. Missing cursor (first page): `WHERE id > 0` or omit the `WHERE` clause.

---

## Exercise 4 — Add Redis Caching with Cache-Aside Pattern (Medium)

**Scenario:** A `GET /users/:id` endpoint hits PostgreSQL on every request. The user data changes rarely (at most once per day). Add a Redis cache using the cache-aside pattern.

**Requirements:**
- On cache miss: fetch from PostgreSQL, store in Redis with a 5-minute TTL, return to client.
- On cache hit: return from Redis without touching PostgreSQL.
- On user update (`PUT /users/:id`): invalidate the cache key immediately.
- Cache key format: `user:v1:<id>`. Version the key so you can invalidate all old-format entries by bumping `v1`.
- Add `X-Cache: HIT` or `X-Cache: MISS` response header.

**Acceptance Criteria:**
- [ ] Second request for the same user returns `X-Cache: HIT`.
- [ ] After `PUT /users/:id`, the next `GET /users/:id` returns `X-Cache: MISS` (cache was invalidated).
- [ ] PostgreSQL is never queried when the cache is warm (verified by a DB query counter).
- [ ] Redis key expires after 5 minutes (verified by checking TTL after set).
- [ ] JSON serialization/deserialization is handled transparently — callers receive a typed `User` object, not a raw string.

**Hints:**
1. Cache-aside: `const cached = await redis.get(key); if (cached) return JSON.parse(cached); const user = await db.findUser(id); await redis.setex(key, 300, JSON.stringify(user)); return user;`.
2. Typed deserialization: create a `deserializeUser(raw: string): User` function that parses and validates with Zod.
3. On update: `await redis.del(`user:v1:${id}`)` immediately after the DB update.
4. Test the invalidation: `await userService.updateUser(id, data); const res = await app.inject({ url: `/users/${id}` }); expect(res.headers['x-cache']).toBe('MISS');`.

---

## Exercise 5 — Optimize a React Component with memo/useMemo (Medium)

**Scenario:** A `ProductList` component re-renders on every parent state change, even when its own data has not changed. Profile it with React DevTools and optimize it.

**Given component (simplified):**
```tsx
function ProductList({ products, onAddToCart, discountRate }) {
  const discountedProducts = products.map(p => ({
    ...p,
    finalPrice: p.price * (1 - discountRate),
  }));

  return (
    <ul>
      {discountedProducts.map(product => (
        <ProductCard key={product.id} product={product} onAdd={() => onAddToCart(product.id)} />
      ))}
    </ul>
  );
}
```

**Requirements:**
- Use React DevTools Profiler to measure renders before optimization.
- Apply `useMemo` to memoize `discountedProducts` computation.
- Wrap `ProductCard` with `React.memo` so it skips re-render when its props do not change.
- Stabilize the `onAdd` callback with `useCallback` to prevent `ProductCard` from re-rendering due to a new function reference.
- After optimization, adding an item to the cart must not re-render `ProductCard` components for other products.

**Acceptance Criteria:**
- [ ] React DevTools Profiler shows `ProductCard` components not re-rendering on unrelated state changes.
- [ ] `useMemo` dependency array is correct — `discountedProducts` recomputes only when `products` or `discountRate` changes.
- [ ] `useCallback` is applied to `onAddToCart` in the parent component — not inside `ProductList`.
- [ ] A Vitest test using `@testing-library/react` verifies the component renders once on initial load.
- [ ] No premature optimization: `React.memo` is not applied to every component — only those that are proven to re-render unnecessarily.

**Hints:**
1. `useMemo`: `const discountedProducts = useMemo(() => products.map(...), [products, discountRate]);`.
2. `React.memo`: `export default React.memo(ProductCard)`. It does a shallow comparison of props by default.
3. The `onAdd` prop problem: `() => onAddToCart(product.id)` creates a new function reference on every render, breaking `memo`. Fix: pass `productId` and `onAddToCart` as separate stable props to `ProductCard`.
4. Confirm with the profiler: record a session, click "Add to Cart" on one product, inspect which `ProductCard` components show in the flame graph.

---

## Exercise 6 — Fix Layout Thrashing in a Given JS Snippet (Medium)

**Scenario:** The following JavaScript code causes layout thrashing — it alternates between reading and writing layout properties, forcing the browser to recalculate layout repeatedly. Fix it.

**Given code (causes layout thrashing):**
```javascript
const items = document.querySelectorAll('.item');

items.forEach(item => {
  const height = item.offsetHeight; // READ — forces layout
  item.style.height = `${height * 2}px`; // WRITE — invalidates layout
});
```

**Requirements:**
- Refactor to batch all reads first, then all writes (read-write separation).
- Measure the performance difference using `performance.mark` and `performance.measure`.
- Verify with Chrome DevTools Performance panel that the "Recalculate Style" events are reduced.
- Apply the same fix to a more complex example: an animated list where each item's `top` is calculated from its previous sibling's height.

**Acceptance Criteria:**
- [ ] Fixed code performs all `offsetHeight` reads in one pass, then all `style.height` writes in a second pass.
- [ ] `performance.measure` shows at least a 5x speedup on a list of 500 items.
- [ ] Chrome DevTools Performance panel shows a single "Layout" event instead of N layout events.
- [ ] A comment in the fixed code explains why the original caused thrashing.
- [ ] The `requestAnimationFrame` API is used if animations are involved.

**Hints:**
1. Batch reads: `const heights = Array.from(items).map(item => item.offsetHeight);`.
2. Batch writes: `items.forEach((item, i) => { item.style.height = `${heights[i] * 2}px`; });`.
3. `performance.mark('start'); /* code */; performance.mark('end'); performance.measure('layout', 'start', 'end');` — read from `performance.getEntriesByName('layout')`.
4. For animations, use `requestAnimationFrame(() => { /* all writes here */ })` to synchronize with the browser's paint cycle.

---

## Exercise 7 — Set Up Core Web Vitals Monitoring (Medium)

**Scenario:** Your React app has poor Lighthouse scores. Set up real user monitoring (RUM) for Core Web Vitals: LCP, CLS, INP (formerly FID), and TTFB.

**Requirements:**
- Use the `web-vitals` library to capture LCP, CLS, INP, and TTFB in real user sessions.
- Report metrics to your analytics endpoint (`POST /analytics/vitals`).
- Set performance budgets: LCP < 2.5s, CLS < 0.1, INP < 200ms.
- Add a CI step using Lighthouse CI (`lhci`) that fails if any Core Web Vital exceeds its budget.
- Display a development-mode overlay that shows live metric values.

**Acceptance Criteria:**
- [ ] `web-vitals` is imported and all four metrics are reported on every page load.
- [ ] `POST /analytics/vitals` receives `{ metric: 'LCP', value: 1234, url: '/dashboard', rating: 'good' }`.
- [ ] Lighthouse CI config (`lighthouserc.js`) defines assertions for LCP, CLS, and INP.
- [ ] Lighthouse CI job fails if LCP > 2500ms (tested with an intentionally slow page).
- [ ] Dev overlay appears only when `NODE_ENV=development` or `?debug=vitals` query param is present.

**Hints:**
1. `web-vitals` usage:
   ```javascript
   import { onLCP, onCLS, onINP, onTTFB } from 'web-vitals';
   [onLCP, onCLS, onINP, onTTFB].forEach(fn => fn(metric => sendToAnalytics(metric)));
   ```
2. `sendToAnalytics`: use `navigator.sendBeacon('/analytics/vitals', JSON.stringify(metric))` — doesn't block page unload.
3. Lighthouse CI: `lhci autorun` in GitHub Actions. Config: `assertions: { 'largest-contentful-paint': ['warn', { maxNumericValue: 2500 }] }`.
4. Dev overlay: a fixed-position `<div>` that subscribes to the same `web-vitals` callbacks and updates its content.

---

## Exercise 8 — Optimize a Dockerfile for Smaller Image Size (Easy)

**Scenario:** A Node.js application's Docker image is 1.2 GB. Reduce it to under 200 MB without breaking functionality.

**Given Dockerfile:**
```dockerfile
FROM node:20

WORKDIR /app

COPY package*.json ./
RUN npm install

COPY . .
RUN npm run build

CMD ["node", "dist/index.js"]
```

**Requirements:**
- Use a multi-stage build with an Alpine-based runtime image.
- Install only production dependencies in the final image (`npm ci --omit=dev`).
- Use `.dockerignore` to exclude unnecessary files from the build context.
- Pin the base image to a specific patch version.
- Verify the final image size with `docker image ls`.

**Acceptance Criteria:**
- [ ] Final image size is under 200 MB (`docker image ls` output).
- [ ] The app still starts and serves requests correctly after the optimization.
- [ ] Multi-stage build: `AS builder` stage builds the app; `AS runtime` stage copies only `dist/` and prod `node_modules`.
- [ ] `.dockerignore` excludes: `node_modules`, `.git`, `*.log`, `.env*`, `test/`, `coverage/`.
- [ ] Base image is pinned (e.g., `node:20.11.1-alpine3.19`), not a floating tag.

**Hints:**
1. Multi-stage:
   ```dockerfile
   FROM node:20-alpine AS builder
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci
   COPY . .
   RUN npm run build

   FROM node:20-alpine AS runtime
   WORKDIR /app
   COPY package*.json ./
   RUN npm ci --omit=dev
   COPY --from=builder /app/dist ./dist
   CMD ["node", "dist/index.js"]
   ```
2. Alpine images are ~5 MB vs. ~900 MB for `node:20` Debian. Combined with omitting dev dependencies, expect a 5–10x size reduction.
3. Check what is large in the builder stage: `docker run --rm builder du -sh /app/node_modules` to identify heavy dependencies.
4. If native modules are needed: use `node:20-alpine` with `apk add --no-cache python3 make g++` in the builder stage only.

---

## Exercise 9 — Add HTTP Caching Headers (ETags + Cache-Control) (Medium)

**Scenario:** Your API returns product data that is expensive to compute but changes infrequently. Add HTTP caching headers so browsers and CDNs can cache responses, and support conditional requests to avoid unnecessary data transfer.

**Requirements:**
- `GET /products/:id`: add `Cache-Control: public, max-age=300` and an `ETag` header.
- When the client sends `If-None-Match: <etag>`, respond with `304 Not Modified` if the content has not changed.
- `Cache-Control` for authenticated endpoints must be `private, no-store` — not publicly cacheable.
- For `GET /products` (list): use `Cache-Control: public, max-age=60, stale-while-revalidate=600`.
- Implement ETag generation: MD5 or SHA-1 hash of the response body JSON.

**Acceptance Criteria:**
- [ ] `GET /products/1` response includes `ETag: "abc123"` and `Cache-Control: public, max-age=300`.
- [ ] Sending `If-None-Match: "abc123"` on a subsequent request returns `304` with no body.
- [ ] After the product is updated, the ETag changes and the next conditional request returns `200` with the new data.
- [ ] `GET /profile` (authenticated) returns `Cache-Control: private, no-store`.
- [ ] A test verifies the 304 flow end-to-end using `fastify.inject`.

**Hints:**
1. ETag generation: `const etag = '"' + createHash('md5').update(JSON.stringify(body)).digest('hex') + '"'`. Wrap in double quotes per RFC 7232.
2. Conditional check: `if (req.headers['if-none-match'] === etag) { reply.code(304).send(); return; }`.
3. `stale-while-revalidate`: tells CDNs to serve a stale cached response while fetching a fresh one in the background. Good for list endpoints where slight staleness is acceptable.
4. In Fastify, set headers before `reply.send`: `reply.header('ETag', etag).header('Cache-Control', 'public, max-age=300')`.

---

## Exercise 10 — Implement Lazy Loading for a Heavy Module (Easy)

**Scenario:** Your Node.js app imports a PDF generation library (`pdf-lib`, ~2 MB) at startup, adding 800ms to cold start time. Only 5% of requests actually use PDF generation. Implement lazy loading.

**Requirements:**
- Move the `pdf-lib` import from module top-level to inside the function that uses it.
- The first call to `generatePdf()` triggers the import; subsequent calls reuse the cached module.
- Use dynamic `import()` syntax — not `require()`.
- Measure the cold start time improvement with `process.hrtime.bigint()`.
- Apply the same pattern to two other heavy dependencies in the codebase.

**Acceptance Criteria:**
- [ ] Server startup time is reduced by at least 500ms (measure with `hyperfine node dist/server.js --warmup 3`).
- [ ] `generatePdf()` works correctly on first call and all subsequent calls.
- [ ] The lazy import is cached — `import()` is only called once, not on every `generatePdf()` invocation.
- [ ] TypeScript types are preserved — the imported module is correctly typed even with dynamic import.
- [ ] A comment explains the trade-off: faster startup vs. slightly higher latency on first PDF request.

**Hints:**
1. Lazy import pattern:
   ```typescript
   let pdfLib: typeof import('pdf-lib') | null = null;
   async function generatePdf(data: PdfData): Promise<Buffer> {
     if (!pdfLib) pdfLib = await import('pdf-lib');
     const { PDFDocument } = pdfLib;
     // ...
   }
   ```
2. TypeScript types with dynamic import: `import type { PDFDocument } from 'pdf-lib'` at the top for types only — does not affect the bundle.
3. Measure startup: `hyperfine --warmup 3 'node -e "require(\"./dist/server\")"'` — run before and after to compare.
4. Other candidates for lazy loading: image processing libraries (`sharp`), spreadsheet generators (`exceljs`), QR code generators.
