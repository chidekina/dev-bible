# HTTP Caching

## Overview

HTTP caching is the mechanism by which browsers, CDNs, and proxies store responses and reuse them for subsequent requests. When done correctly, it eliminates entire categories of network requests — the user gets the resource from their local cache in milliseconds, and the server never sees the request. This chapter covers the Cache-Control header in detail, ETags, the `stale-while-revalidate` strategy, and how to design a caching strategy for different types of resources.

---

## Prerequisites

- HTTP fundamentals (headers, status codes)
- Understanding of CDNs (Track 08)
- Fastify or Express basics

---

## Core Concepts

### Where caching happens

```
Browser cache → CDN edge cache → Reverse proxy cache → Origin server
```

HTTP headers control behavior at every layer. A response with `Cache-Control: public, max-age=3600` is cached for 1 hour by the browser, the CDN, and any intermediate proxy.

### Cache-Control directives

| Directive | Meaning |
|-----------|---------|
| `public` | Response can be cached by any cache (browser + CDN) |
| `private` | Response can only be cached by the browser (not CDN) |
| `no-store` | Never cache — must always fetch from server |
| `no-cache` | Cache is allowed but must revalidate before using (confusingly named) |
| `max-age=N` | Cache is valid for N seconds from the response time |
| `s-maxage=N` | Override `max-age` for shared caches (CDNs) only |
| `immutable` | Hint that the resource will never change (skip revalidation) |
| `stale-while-revalidate=N` | Serve stale content for up to N seconds while fetching fresh in background |
| `must-revalidate` | Must revalidate once expired (no serving stale) |

---

## Hands-On Examples

### Setting Cache-Control in Fastify

```typescript
// src/plugins/cache-control.ts
import { FastifyPluginAsync } from 'fastify';

const cacheControl: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('onSend', async (request, reply, payload) => {
    const url = request.url;

    // Immutable hashed assets — cache forever, browser can skip revalidation
    if (/\.(js|css|woff2|ttf)$/.test(url) && /[a-f0-9]{8}/.test(url)) {
      reply.header('Cache-Control', 'public, max-age=31536000, immutable');
      return payload;
    }

    // API responses with data that can be slightly stale
    if (url.startsWith('/api/products') && request.method === 'GET') {
      reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=600');
      return payload;
    }

    // Private user data — browser can cache but CDN cannot
    if (url.startsWith('/api/me') || url.startsWith('/api/account')) {
      reply.header('Cache-Control', 'private, max-age=300');
      return payload;
    }

    // Default: no caching for API endpoints
    if (url.startsWith('/api/')) {
      reply.header('Cache-Control', 'no-store');
      return payload;
    }

    return payload;
  });
};
```

### ETag-based conditional caching

ETags let the browser ask "has this changed?" The server returns 304 Not Modified if the content is the same — no body is sent, saving bandwidth:

```typescript
import crypto from 'crypto';

function generateETag(content: unknown): string {
  const hash = crypto
    .createHash('sha256')
    .update(JSON.stringify(content))
    .digest('hex')
    .slice(0, 16);
  return `"${hash}"`;
}

fastify.get('/api/products/:id', async (request, reply) => {
  const { id } = request.params as { id: string };
  const product = await db.product.findUnique({ where: { id } });

  if (!product) return reply.status(404).send({ error: 'Not found' });

  const etag = generateETag(product);
  reply.header('ETag', etag);
  reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');

  // If client sends If-None-Match header with same ETag, send 304
  if (request.headers['if-none-match'] === etag) {
    return reply.status(304).send();
  }

  return product;
});
```

The second request from the browser includes `If-None-Match: "abc123"`. If the product hasn't changed, the server returns `304` with no body — the browser uses its cached copy.

### Last-Modified conditional caching

Alternative to ETags using timestamps:

```typescript
fastify.get('/api/posts/:id', async (request, reply) => {
  const post = await db.post.findUnique({ where: { id: request.params.id } });
  if (!post) return reply.status(404).send();

  const lastModified = post.updatedAt.toUTCString();
  reply.header('Last-Modified', lastModified);
  reply.header('Cache-Control', 'public, max-age=300');

  // Check If-Modified-Since
  const ifModifiedSince = request.headers['if-modified-since'];
  if (ifModifiedSince && new Date(ifModifiedSince) >= post.updatedAt) {
    return reply.status(304).send();
  }

  return post;
});
```

### stale-while-revalidate — serving stale content while refreshing

```
Client requests /api/products
  max-age=60 has passed (cache stale)
  stale-while-revalidate=600 has NOT passed

  → Serve stale response immediately (fast for user)
  → Background: refetch from origin and update cache
```

```typescript
// CDN and browser handle stale-while-revalidate automatically
// You just set the header:
reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=600');

// Stale for 1 minute → served from cache
// After 1 minute → cache fetches fresh in background, serves stale while waiting
// After 11 minutes (1 + 10) → cache is expired, must block for fresh fetch
```

This is the best strategy for data that can be 1–2 minutes stale without user impact.

### Vary header — caching content-negotiated responses

When the same URL returns different content based on request headers (e.g., Accept, Accept-Language), use `Vary`:

```typescript
fastify.get('/api/products', async (request, reply) => {
  const acceptLanguage = request.headers['accept-language'] ?? 'en';
  const lang = acceptLanguage.startsWith('pt') ? 'pt' : 'en';

  const products = await getLocalizedProducts(lang);

  // Cache separately per language
  reply.header('Vary', 'Accept-Language');
  reply.header('Cache-Control', 'public, max-age=300');

  return products;
});
```

Without `Vary: Accept-Language`, a CDN might serve an English response to a Portuguese user who requested the same URL.

### Cache-busting for HTML pages

HTML pages reference assets by URL. When you deploy new code:
- Assets with hashed names: new hash = new URL = cache miss = new download
- HTML pages: must be refreshed so users get the new asset URLs

```typescript
// HTML pages — short cache, always revalidate
reply.header('Cache-Control', 'public, max-age=0, must-revalidate');
// Or use ETag — browser revalidates, server returns 304 if unchanged
```

### Service Worker caching (offline-first)

```typescript
// public/sw.js
const CACHE_VERSION = 'v2';
const STATIC_CACHE = `static-${CACHE_VERSION}`;
const API_CACHE = `api-${CACHE_VERSION}`;

self.addEventListener('install', (event) => {
  event.waitUntil(
    caches.open(STATIC_CACHE).then((cache) =>
      cache.addAll(['/offline.html', '/styles/main.css', '/scripts/app.js'])
    )
  );
});

self.addEventListener('fetch', (event) => {
  const url = new URL(event.request.url);

  // API responses: network first, cache fallback
  if (url.pathname.startsWith('/api/')) {
    event.respondWith(
      fetch(event.request)
        .then((response) => {
          const clone = response.clone();
          caches.open(API_CACHE).then((cache) => cache.put(event.request, clone));
          return response;
        })
        .catch(() => caches.match(event.request)) // offline fallback
    );
    return;
  }

  // Static assets: cache first
  event.respondWith(
    caches.match(event.request).then((cached) => cached ?? fetch(event.request))
  );
});
```

---

## Common Patterns & Best Practices

- **Immutable assets with hashes** — `max-age=31536000, immutable`; zero-cost cache invalidation
- **HTML pages** — short TTL (0–60s) with ETag for conditional revalidation
- **Public API data** — `stale-while-revalidate` for freshness + performance
- **User data** — `private` so CDN does not cache; browser can cache briefly
- **Authenticated endpoints** — `no-store` or `private` depending on sensitivity
- **CDN purge on deploy** — clear HTML pages after each deployment

---

## Anti-Patterns to Avoid

- `max-age=0` without `must-revalidate` or ETags — cache is useless
- `Cache-Control: public` on user-specific responses — CDN serves one user's data to another
- Long TTL on non-hashed static files — users see stale JS/CSS after deployment
- Not setting `Vary` when responses differ by request header
- `no-cache` when you mean `no-store` — `no-cache` means "cache but revalidate", not "never cache"

---

## Debugging & Troubleshooting

**"The browser is not caching my response"**
Check the response headers in DevTools Network tab. If `Cache-Control: no-store`, the browser discards the response immediately. If there's a `Set-Cookie` on the response, some browsers won't cache it even with `public`.

**"CDN is serving stale content after I deployed"**
The cache TTL has not expired and you haven't purged. Use the CDN's purge API or change the asset URL (add a version parameter or use content hashes).

**"Users are getting different content at the same URL"**
You're missing a `Vary` header. Add `Vary: Accept-Language` or `Vary: Accept-Encoding` so caches store separate copies for different variants.

---

## Further Reading

- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [web.dev: HTTP caching guide](https://web.dev/http-cache/)
- [RFC 9111: HTTP Caching](https://httpwg.org/specs/rfc9111.html)
- Track 08: [CDN](../08-system-design/cdn.md)
- Track 08: [Caching Strategies](../08-system-design/caching-strategies.md)

---

## Summary

HTTP caching is the most impactful performance optimization that requires zero application code changes for static assets. The strategy is simple: immutable assets with content hashes get `max-age=31536000, immutable` (cache forever, no invalidation needed); HTML pages get short TTLs with ETags for conditional revalidation; public API data uses `stale-while-revalidate` for freshness without blocking; user-specific data uses `private` or `no-store`. The most common mistake is either not caching at all (wasted bandwidth, slow loads) or caching incorrectly (`public` on private data, long TTLs on non-hashed files).
