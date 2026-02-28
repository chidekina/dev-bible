# CDN

## Overview

A Content Delivery Network (CDN) is a geographically distributed network of servers (edge nodes) that cache and serve content from locations close to users. Instead of every user hitting your origin server in São Paulo, a user in Tokyo gets content from an edge node in Tokyo. This reduces latency, reduces origin server load, and improves resilience. CDNs serve static assets, but modern CDNs also handle dynamic content, API responses, image optimization, and edge computing. This chapter covers CDN fundamentals, cache invalidation, and practical configuration.

---

## Prerequisites

- HTTP caching headers (Cache-Control, ETags)
- Basic DNS understanding
- Track 07 — Security: HTTPS/TLS concepts

---

## Core Concepts

### How CDN caching works

```
First request (cache miss):
User (Tokyo) → CDN Edge (Tokyo) → Origin (São Paulo) → Response
                ↑ CDN stores response

Subsequent requests (cache hit):
User (Tokyo) → CDN Edge (Tokyo) → Cached response (fast — no origin hit)
```

**Cache hit ratio:** the percentage of requests served from cache. Goal: > 90% for static assets.

### Types of content

| Content type | CDN strategy | Typical TTL |
|-------------|-------------|-------------|
| Immutable assets (JS, CSS with hash in filename) | Cache forever | 1 year |
| Images, fonts | Long cache | 30 days |
| HTML pages | Short cache or no cache | 0–5 minutes |
| API responses | Depends on data freshness | 0 seconds to minutes |
| User-specific data | Usually not cached | N/A |

### Push vs Pull CDN

**Pull CDN (most common):** CDN fetches from origin on the first request, then caches. Zero setup — just configure DNS.

**Push CDN:** You explicitly upload assets to the CDN. Useful for large files (videos, backups) where you want to pre-position content before anyone requests it.

---

## Hands-On Examples

### Configuring Cache-Control headers in Fastify

```typescript
// src/plugins/cache-headers.ts
import { FastifyPluginAsync } from 'fastify';

const cacheHeaders: FastifyPluginAsync = async (fastify) => {
  // Static assets served by Fastify (usually handled by Nginx/CDN instead)
  fastify.register(import('@fastify/static'), {
    root: new URL('../public', import.meta.url).pathname,
  });

  // Set caching headers per route type
  fastify.addHook('onSend', async (request, reply) => {
    const url = request.url;

    // Immutable static assets (hashed filenames: app.abc123.js)
    if (/\.(js|css|woff2|woff|ttf)$/.test(url) && /[a-f0-9]{8,}/.test(url)) {
      reply.header('Cache-Control', 'public, max-age=31536000, immutable');
      return;
    }

    // Images — long cache
    if (/\.(png|jpg|jpeg|webp|avif|svg|ico|gif)$/.test(url)) {
      reply.header('Cache-Control', 'public, max-age=2592000'); // 30 days
      return;
    }

    // HTML pages — short cache with revalidation
    if (reply.getHeader('Content-Type')?.toString().includes('text/html')) {
      reply.header('Cache-Control', 'public, max-age=60, stale-while-revalidate=300');
      return;
    }

    // API responses — not cached by default
    if (url.startsWith('/api/')) {
      reply.header('Cache-Control', 'no-store');
      return;
    }
  });
};
```

### Cache-Control for API responses

```typescript
// Public, cacheable API responses (product catalog, public content)
fastify.get('/api/products', async (request, reply) => {
  const products = await getProducts();

  // Cache at CDN and browser for 5 minutes; serve stale for 1 hour while revalidating
  reply.header('Cache-Control', 'public, max-age=300, stale-while-revalidate=3600');

  return products;
});

// User-specific data — never cache
fastify.get('/api/me', { preHandler: [requireAuth] }, async (request) => {
  // implicitly: reply.header('Cache-Control', 'no-store')
  return getUserProfile(request.user.id);
});

// ETag-based conditional caching
fastify.get('/api/products/:id', async (request, reply) => {
  const product = await getProduct(request.params.id);
  const etag = `"${product.updatedAt.getTime()}"`;

  reply.header('ETag', etag);
  reply.header('Cache-Control', 'public, max-age=60');

  // If client sends If-None-Match and ETags match, return 304
  if (request.headers['if-none-match'] === etag) {
    return reply.status(304).send();
  }

  return product;
});
```

### Cache invalidation in Cloudflare

```typescript
// Purge specific URLs from Cloudflare cache after content changes
async function purgeCloudflareCache(urls: string[]): Promise<void> {
  const response = await fetch(
    `https://api.cloudflare.com/client/v4/zones/${process.env.CLOUDFLARE_ZONE_ID}/purge_cache`,
    {
      method: 'POST',
      headers: {
        Authorization: `Bearer ${process.env.CLOUDFLARE_API_TOKEN}`,
        'Content-Type': 'application/json',
      },
      body: JSON.stringify({ files: urls }),
    }
  );

  if (!response.ok) {
    throw new Error(`Cloudflare purge failed: ${response.statusText}`);
  }
}

// Usage: after updating a product
await db.product.update({ where: { id }, data });
await purgeCloudflareCache([
  `https://www.example.com/api/products/${id}`,
  `https://www.example.com/products/${id}`,
]);
```

### Asset naming strategy (immutable caching)

The key to cache-busting without cache invalidation is content-addressed file names:

```typescript
// vite.config.ts
export default {
  build: {
    rollupOptions: {
      output: {
        // Hash in filename — when content changes, filename changes → new cache entry
        entryFileNames: 'assets/[name].[hash].js',
        chunkFileNames: 'assets/[name].[hash].js',
        assetFileNames: 'assets/[name].[hash][extname]',
      },
    },
  },
};

// After build:
// assets/app.a1b2c3d4.js  → Cache-Control: immutable, max-age=31536000
// assets/app.e5f6a7b8.js  → new hash = new URL = no cache conflict
```

### Cloudflare configuration (via Terraform)

```hcl
# Page rules for caching behavior
resource "cloudflare_page_rule" "static_assets" {
  zone_id = var.zone_id
  target  = "*.example.com/assets/*"
  priority = 1

  actions {
    cache_level = "cache_everything"
    edge_cache_ttl = 2592000  # 30 days at edge
    browser_cache_ttl = 31536000
  }
}

resource "cloudflare_page_rule" "api_no_cache" {
  zone_id = var.zone_id
  target  = "*.example.com/api/*"
  priority = 2

  actions {
    cache_level = "bypass"
  }
}
```

### CDN for image optimization

Modern CDNs (Cloudflare Images, Imgix, Cloudinary) can transform images on the fly:

```typescript
// src/lib/image.ts — generate optimized image URLs
const CDN_BASE = 'https://images.example.com';

interface ImageOptions {
  width?: number;
  height?: number;
  format?: 'webp' | 'avif' | 'auto';
  quality?: number;
}

export function optimizedImageUrl(path: string, options: ImageOptions = {}): string {
  const params = new URLSearchParams();
  if (options.width) params.set('w', String(options.width));
  if (options.height) params.set('h', String(options.height));
  if (options.format) params.set('fm', options.format);
  if (options.quality) params.set('q', String(options.quality));

  return `${CDN_BASE}${path}?${params.toString()}`;
}

// Usage in React component
function ProductImage({ product }: { product: Product }) {
  return (
    <picture>
      <source
        srcSet={optimizedImageUrl(product.imageUrl, { width: 800, format: 'avif' })}
        type="image/avif"
      />
      <source
        srcSet={optimizedImageUrl(product.imageUrl, { width: 800, format: 'webp' })}
        type="image/webp"
      />
      <img
        src={optimizedImageUrl(product.imageUrl, { width: 800 })}
        alt={product.name}
        loading="lazy"
        decoding="async"
      />
    </picture>
  );
}
```

---

## Common Patterns & Best Practices

- **Immutable assets with content hashes** — never worry about cache invalidation for JS/CSS
- **Short TTL with `stale-while-revalidate`** — serve cached content while fetching fresh version in background
- **Separate static and API origins** — `cdn.example.com` for assets, `api.example.com` for API (CDN can be configured differently per subdomain)
- **Warm the cache after deployment** — make requests to critical URLs before directing user traffic
- **Monitor cache hit rate** — available in Cloudflare Analytics, Fastly, AWS CloudFront

---

## Anti-Patterns to Avoid

- Caching user-specific responses (profile data, cart contents) at the CDN level — every user would see the same data
- Very long TTLs on non-hashed filenames — when you deploy an update, users see stale content until TTL expires
- Caching 500 responses — a temporary error can get cached and served for hours
- Not setting `Vary` header when content varies by `Accept-Encoding` or `Accept` — CDN may serve compressed content to clients that don't support it

---

## Debugging & Troubleshooting

**"Users are seeing old content after a deploy"**
Your assets do not have content hashes in their filenames. The CDN serves the cached version until TTL expires. Solution: use a build tool that generates hashed filenames, or purge the CDN cache after each deploy.

**"CDN is serving wrong content to different users"**
Check `Vary` headers. If your API serves different responses based on `Authorization` or `Accept-Language`, the CDN must be configured to vary the cache by those headers, or bypass caching entirely for those routes.

**"Cache hit rate is 20% for static assets"**
Check if query strings are causing cache misses (`/styles.css?v=1` and `/styles.css?v=2` are treated as different URLs by many CDNs). Normalize URLs or use filename hashing instead.

---

## Further Reading

- [Cloudflare Cache documentation](https://developers.cloudflare.com/cache/)
- [MDN: Cache-Control](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Cache-Control)
- [web.dev: HTTP caching](https://web.dev/http-cache/)
- Track 10: [HTTP Caching](../10-performance/caching-http.md)
- Track 08: [Caching Strategies](caching-strategies.md)

---

## Summary

CDNs are the single most cost-effective performance improvement for web applications that serve global users. The key to maximizing CDN effectiveness is correct Cache-Control headers: immutable content with hashed filenames gets infinite caching with zero invalidation complexity; dynamic content gets short TTLs or bypasses caching entirely. Cache invalidation should be driven by content hashes (build-time) rather than manual purges (error-prone). Modern CDNs extend beyond static file serving to image optimization, edge compute, and DDoS protection — making them a foundational layer of any serious production architecture.
