# API Gateways

## Overview

An API gateway is a server that acts as the single entry point for all client requests to a system. It handles cross-cutting concerns — authentication, rate limiting, routing, request transformation, logging, and SSL termination — so that backend services do not need to implement them individually. In microservice architectures, the gateway is essential. In monolith architectures, it still provides value as a reverse proxy and security layer. This chapter covers what gateways do, how to build lightweight gateway logic in Fastify, and how to configure production-grade gateways.

---

## Prerequisites

- HTTP fundamentals (headers, methods, proxying)
- Basic understanding of microservices concepts
- Track 07 — Security: authentication and rate limiting concepts

---

## Core Concepts

### What an API gateway does

```
Client → [API Gateway] → Service A
                      → Service B
                      → Service C
```

**Responsibilities of the gateway:**

| Responsibility | Details |
|---------------|---------|
| Routing | Map `/api/users` to `users-service`, `/api/orders` to `orders-service` |
| Authentication | Verify JWTs or API keys before forwarding requests |
| Rate limiting | Throttle requests per client or per IP |
| Load balancing | Distribute requests across instances of a service |
| SSL termination | Handle HTTPS; forward HTTP to internal services |
| Request transformation | Modify headers, body, or URL before forwarding |
| Response transformation | Aggregate, filter, or reshape responses |
| Caching | Cache responses at the gateway level |
| Observability | Centralized logging, metrics, tracing |

### Gateway vs reverse proxy vs load balancer

- **Reverse proxy** (Nginx): routes requests, terminates TLS, serves static files. Simple.
- **Load balancer** (Nginx upstream, AWS ALB): distributes traffic. Focused on availability.
- **API gateway** (Kong, AWS API Gateway, custom Fastify): all of the above, plus auth, rate limiting, transformation. Application-aware.

---

## Hands-On Examples

### Simple gateway in Fastify with @fastify/http-proxy

```bash
npm install @fastify/http-proxy
```

```typescript
// src/gateway.ts
import Fastify from 'fastify';
import proxy from '@fastify/http-proxy';
import helmet from '@fastify/helmet';
import rateLimit from '@fastify/rate-limit';
import jwt from '@fastify/jwt';

const gateway = Fastify({ logger: true });

await gateway.register(helmet);
await gateway.register(jwt, { secret: process.env.JWT_SECRET! });

// Rate limiting — applied globally before routing
await gateway.register(rateLimit, {
  max: 100,
  timeWindow: '1 minute',
  keyGenerator: (request) => request.headers.authorization ?? request.ip,
});

// Authentication middleware — applied to protected routes
async function requireAuth(request: any, reply: any) {
  try {
    await request.jwtVerify();
  } catch {
    return reply.status(401).send({ error: 'Unauthorized' });
  }
}

// Route: users service (requires auth)
await gateway.register(proxy, {
  upstream: 'http://users-service:3001',
  prefix: '/api/users',
  preHandler: requireAuth,
  rewritePrefix: '/users',
  // Adds X-User-Id header so the upstream service does not re-verify the JWT
  replyOptions: {
    rewriteRequestHeaders: (request, headers) => ({
      ...headers,
      'x-user-id': (request as any).user?.id,
      'x-user-role': (request as any).user?.role,
    }),
  },
});

// Route: public search endpoint (no auth required)
await gateway.register(proxy, {
  upstream: 'http://search-service:3002',
  prefix: '/api/search',
  rewritePrefix: '/search',
});

// Route: orders service (requires auth + admin)
await gateway.register(proxy, {
  upstream: 'http://orders-service:3003',
  prefix: '/api/orders',
  preHandler: [requireAuth, async (request: any, reply: any) => {
    if (request.user.role !== 'admin') {
      return reply.status(403).send({ error: 'Forbidden' });
    }
  }],
  rewritePrefix: '/orders',
});

await gateway.listen({ port: 8080, host: '0.0.0.0' });
```

### Rate limiting strategies

```typescript
// Per-user rate limit (authenticated routes)
await gateway.register(rateLimit, {
  max: 1000,
  timeWindow: '1 hour',
  keyGenerator: (request) => {
    // Authenticated: rate limit by user ID
    const user = (request as any).user;
    if (user) return `user:${user.id}`;
    // Unauthenticated: rate limit by IP
    return `ip:${request.ip}`;
  },
  errorResponseBuilder: (request, context) => ({
    code: 'RATE_LIMIT_EXCEEDED',
    error: 'Too Many Requests',
    message: `Rate limit exceeded. Try again in ${context.after}`,
    retryAfter: context.after,
  }),
});

// Tiered rate limits by subscription plan
const RATE_LIMITS = {
  free: { max: 100, window: '1 hour' },
  pro: { max: 5000, window: '1 hour' },
  enterprise: { max: 100000, window: '1 hour' },
};

gateway.addHook('preHandler', async (request, reply) => {
  const user = (request as any).user;
  const tier = user?.tier ?? 'free';
  const limits = RATE_LIMITS[tier as keyof typeof RATE_LIMITS];

  // Check custom rate limit store (Redis)
  const key = `ratelimit:${user?.id ?? request.ip}`;
  const count = await redis.incr(key);
  if (count === 1) await redis.expire(key, 3600); // set TTL on first request

  if (count > limits.max) {
    return reply.status(429).send({
      error: 'Rate limit exceeded',
      limit: limits.max,
      reset: await redis.ttl(key),
    });
  }

  reply.header('X-RateLimit-Limit', limits.max);
  reply.header('X-RateLimit-Remaining', Math.max(0, limits.max - count));
});
```

### Request/response transformation

```typescript
// Add correlation ID to every request for tracing
gateway.addHook('onRequest', async (request) => {
  const correlationId = request.headers['x-correlation-id'] ?? crypto.randomUUID();
  request.headers['x-correlation-id'] = correlationId;
});

// Strip internal headers from upstream responses before sending to client
gateway.addHook('onSend', async (request, reply, payload) => {
  reply.removeHeader('x-internal-service-name');
  reply.removeHeader('x-db-query-count');
  return payload;
});
```

### Kong API Gateway configuration

Kong is a production-grade gateway with a plugin ecosystem. Configure via its Admin API:

```bash
# Create a service (backend)
curl -i -X POST http://kong:8001/services \
  --data name=users-service \
  --data url=http://users-service:3001

# Create a route (maps URL to service)
curl -i -X POST http://kong:8001/services/users-service/routes \
  --data paths[]=/api/users

# Add JWT authentication plugin
curl -i -X POST http://kong:8001/services/users-service/plugins \
  --data name=jwt

# Add rate limiting plugin
curl -i -X POST http://kong:8001/services/users-service/plugins \
  --data name=rate-limiting \
  --data config.minute=60 \
  --data config.hour=1000 \
  --data config.policy=redis \
  --data config.redis_host=redis
```

### Nginx as a lightweight gateway

```nginx
http {
  # Backend services
  upstream users_service { server users-service:3001; }
  upstream orders_service { server orders-service:3002; }
  upstream search_service { server search-service:3003; }

  # Rate limiting zone
  limit_req_zone $binary_remote_addr zone=api_limit:10m rate=100r/m;

  server {
    listen 80;

    # Apply rate limit
    limit_req zone=api_limit burst=20 nodelay;

    # Route by URL prefix
    location /api/users {
      auth_request /auth/verify;   # subrequest to verify JWT
      proxy_pass http://users_service;
      proxy_set_header X-Real-IP $remote_addr;
    }

    location /api/orders {
      auth_request /auth/verify;
      proxy_pass http://orders_service;
    }

    location /api/search {
      # Public — no auth
      proxy_pass http://search_service;
    }

    # Internal auth verification endpoint
    location = /auth/verify {
      internal;
      proxy_pass http://auth-service:3000/verify;
      proxy_pass_request_body off;
      proxy_set_header Content-Length "";
      proxy_set_header X-Original-URI $request_uri;
    }
  }
}
```

---

## Common Patterns & Best Practices

- **Centralize auth at the gateway** — downstream services trust `X-User-Id` and `X-User-Role` headers set by the gateway; they do not verify JWTs independently
- **Pass correlation IDs** — add `X-Correlation-ID` to every request for end-to-end tracing across services
- **Log at the gateway level** — one centralized log stream vs. N service log streams
- **Use circuit breakers** — if a backend is unhealthy, stop forwarding requests to it (fail fast)
- **Return consistent error responses** — translate upstream errors into a consistent envelope format

---

## Anti-Patterns to Avoid

- Business logic in the gateway — gateways handle cross-cutting concerns, not domain logic
- Single gateway for all traffic without any fallback — the gateway itself becomes a single point of failure (run multiple instances behind a load balancer)
- Blocking the gateway with slow upstream calls — set aggressive upstream timeouts
- Forwarding internal headers to clients — strip `X-Internal-*` headers from outbound responses

---

## Debugging & Troubleshooting

**"Clients are getting 502 Bad Gateway"**
The upstream service is not responding. Check that the service is running and the gateway has the correct upstream URL. Check upstream logs for errors.

**"Rate limiting is not working"**
If running multiple gateway instances without shared Redis state, each instance has its own counter. Use Redis-backed rate limiting so all instances share a single counter.

**"JWT is invalid according to the downstream service"**
The downstream service should not be verifying JWTs — only the gateway should. Downstream services trust the `X-User-Id` header set by the gateway. Remove JWT verification from downstream services.

---

## Further Reading

- [Kong API Gateway documentation](https://docs.konghq.com/)
- [AWS API Gateway documentation](https://docs.aws.amazon.com/apigateway/)
- [@fastify/http-proxy](https://github.com/fastify/fastify-http-proxy)
- Track 08: [Load Balancing](load-balancing.md)
- Track 07: [Security Headers](../07-security/security-headers.md)

---

## Summary

An API gateway is the natural home for cross-cutting concerns in a multi-service architecture: authentication, rate limiting, routing, SSL termination, and observability. For teams running microservices, a gateway is essential — it prevents every service from reimplementing auth and rate limiting independently. For monolith teams, Nginx or a Fastify gateway handles routing and security at lower complexity. The key principle is that the gateway handles infrastructure concerns and passes identity to services; services focus on business logic.
