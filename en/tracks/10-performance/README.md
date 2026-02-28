# Track 10 — Performance

Measure before you optimize. Performance is about understanding where time and bytes are spent and making deliberate trade-offs to improve user experience and reduce infrastructure costs.

**Estimated time:** 2–3 weeks

---

## Topics

1. [Web Performance Metrics](web-performance-metrics.md) — Core Web Vitals, LCP, CLS, INP, TTFB and how to measure them
2. [JavaScript Optimization](javascript-optimization.md) — bundle splitting, tree-shaking, lazy loading, avoiding layout thrashing
3. [Database Query Optimization](database-query-optimization.md) — EXPLAIN ANALYZE, indexes, N+1 queries, query batching
4. [HTTP Caching](caching-http.md) — Cache-Control, ETags, stale-while-revalidate, service workers
5. [Profiling Tools](profiling-tools.md) — Chrome DevTools, Node.js --prof, clinic.js, 0x flame graphs
6. [Image Optimization](image-optimization.md) — WebP/AVIF, responsive images, lazy loading, CDN delivery

---

## Prerequisites

- Track 02 — Frontend (React, bundlers)
- Track 03 — Backend (databases, HTTP)
- Track 05 — DevOps (CDN concepts from Track 08 are also helpful)

---

## What You'll Build

- A Lighthouse audit baseline and a before/after report after applying optimizations to a Next.js page
- An index strategy for a Postgres table with slow queries, validated with EXPLAIN ANALYZE
- A Node.js flame graph identifying a hot path in a Fastify handler, with the bottleneck resolved
- An HTTP caching strategy for a REST API with ETags and Cache-Control headers
- An image pipeline using Sharp that converts uploads to WebP with responsive sizes
