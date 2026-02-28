# Load Balancing

## Overview

Load balancing distributes incoming requests across multiple servers to maximize throughput, minimize latency, and avoid overloading any single node. It is the mechanism that allows horizontal scaling — adding more servers to handle more traffic. Without a load balancer, you have a single point of failure and an artificial traffic ceiling. This chapter covers load balancing algorithms, health checks, session affinity, and the difference between layer 4 and layer 7 load balancing.

---

## Prerequisites

- Basic networking (TCP/IP, HTTP)
- Familiarity with Docker and reverse proxies
- Track 05 — DevOps: containers and nginx concepts

---

## Core Concepts

### Layer 4 vs Layer 7

**Layer 4 (Transport):** Load balancing at the TCP/UDP level. The balancer routes packets based on IP address and port, without inspecting HTTP content. Fast, low overhead, but cannot make content-based routing decisions.

**Layer 7 (Application):** Load balancing at the HTTP level. The balancer inspects request content (URL, headers, cookies) and can route based on path, host, or user identity. More flexible, slightly more overhead.

| Feature | Layer 4 | Layer 7 |
|---------|---------|---------|
| Routing basis | IP + port | URL, host, headers |
| TLS termination | Pass-through or terminate | Terminate (and re-encrypt) |
| Sticky sessions | By IP | By cookie |
| Content inspection | No | Yes |
| Examples | AWS NLB, HAProxy (TCP) | Nginx, Traefik, AWS ALB |

### Load balancing algorithms

| Algorithm | How it works | Best for |
|-----------|-------------|----------|
| Round Robin | Requests cycle through servers in order | Homogeneous servers, stateless apps |
| Weighted Round Robin | Like round robin, but some servers get more requests | Heterogeneous server capacity |
| Least Connections | Next request goes to server with fewest active connections | Variable-length requests |
| Least Response Time | Routes to server with lowest latency | Latency-sensitive applications |
| IP Hash | Hash the client IP to determine server | Sticky sessions without cookies |
| Random | Pick a random server | Simple, surprisingly effective |

---

## Hands-On Examples

### Nginx as an HTTP load balancer

```nginx
# /etc/nginx/nginx.conf
http {
    upstream api_servers {
        # Default: round robin
        server api-1:3000;
        server api-2:3000;
        server api-3:3000;

        # Health check (nginx plus feature; for open source, check externally)
        # keepalive 32; — reuse connections to backends
    }

    server {
        listen 80;

        location / {
            proxy_pass http://api_servers;
            proxy_http_version 1.1;
            proxy_set_header Connection "";           # enable keepalive to upstreams
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Timeouts
            proxy_connect_timeout 5s;
            proxy_read_timeout 30s;
        }
    }
}
```

**Least connections algorithm:**

```nginx
upstream api_servers {
    least_conn;
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
}
```

**Weighted round robin:**

```nginx
upstream api_servers {
    server api-1:3000 weight=3;  # gets 3x as many requests
    server api-2:3000 weight=1;
}
```

**IP hash (sticky sessions):**

```nginx
upstream api_servers {
    ip_hash;
    server api-1:3000;
    server api-2:3000;
    server api-3:3000;
    # Same client IP always routes to same server
}
```

### Passive health checks in Nginx

```nginx
upstream api_servers {
    server api-1:3000 max_fails=3 fail_timeout=30s;
    server api-2:3000 max_fails=3 fail_timeout=30s;
    # After 3 failures within 30s, server is removed for 30s
}
```

### Traefik as a load balancer (Docker)

Traefik discovers backend services through Docker labels:

```yaml
# docker-compose.yml
services:
  api:
    image: myapp:latest
    deploy:
      replicas: 3       # 3 instances of the API
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.services.api.loadbalancer.server.port=3000"
      # Default: round robin across all 3 instances

  traefik:
    image: traefik:v3.0
    command:
      - "--providers.docker=true"
      - "--providers.docker.swarmMode=true"  # for Docker Swarm
      - "--entrypoints.web.address=:80"
    ports:
      - "80:80"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
```

### Sticky sessions with cookies (Traefik)

```yaml
labels:
  - "traefik.http.services.api.loadbalancer.sticky=true"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.name=lb_sticky"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.httponly=true"
  - "traefik.http.services.api.loadbalancer.sticky.cookie.secure=true"
```

### Health check endpoint in Node.js

Every backend should expose a health endpoint for the load balancer to probe:

```typescript
// src/routes/health.ts
import { FastifyPluginAsync } from 'fastify';
import { db } from '../lib/db.js';
import { redis } from '../lib/redis.js';

const healthRoute: FastifyPluginAsync = async (fastify) => {
  fastify.get('/health', { logLevel: 'silent' }, async (request, reply) => {
    const checks: Record<string, 'ok' | 'error'> = {};

    // Check database connectivity
    try {
      await db.$queryRaw`SELECT 1`;
      checks.database = 'ok';
    } catch {
      checks.database = 'error';
    }

    // Check Redis connectivity
    try {
      await redis.ping();
      checks.redis = 'ok';
    } catch {
      checks.redis = 'error';
    }

    const healthy = Object.values(checks).every((v) => v === 'ok');

    return reply.status(healthy ? 200 : 503).send({
      status: healthy ? 'healthy' : 'unhealthy',
      checks,
      version: process.env.APP_VERSION ?? 'unknown',
    });
  });

  // Liveness check — is the process running? (no external checks)
  fastify.get('/health/live', { logLevel: 'silent' }, async () => {
    return { status: 'alive' };
  });
};

export default healthRoute;
```

### Simulating load balancing in Node.js (understanding the pattern)

```typescript
// Conceptual implementation — understanding round robin
class RoundRobinBalancer {
  private servers: string[];
  private index = 0;

  constructor(servers: string[]) {
    this.servers = servers;
  }

  next(): string {
    const server = this.servers[this.index % this.servers.length];
    this.index++;
    return server;
  }
}

class LeastConnectionsBalancer {
  private connections = new Map<string, number>();

  constructor(servers: string[]) {
    for (const server of servers) {
      this.connections.set(server, 0);
    }
  }

  acquire(): string {
    let min = Infinity;
    let chosen = '';
    for (const [server, count] of this.connections) {
      if (count < min) { min = count; chosen = server; }
    }
    this.connections.set(chosen, (this.connections.get(chosen) ?? 0) + 1);
    return chosen;
  }

  release(server: string): void {
    this.connections.set(server, Math.max(0, (this.connections.get(server) ?? 1) - 1));
  }
}
```

---

## Common Patterns & Best Practices

- **Stateless backends** — do not store session data in memory; store it in Redis. Stateless servers are interchangeable.
- **Separate liveness and readiness probes** — liveness: "is the process alive?" Readiness: "can it serve traffic?" (checks DB, Redis, etc.)
- **Remove slow servers quickly** — use `max_fails` and `fail_timeout` so one sick server doesn't drag down the whole pool
- **Use keepalive connections to backends** — reduces TCP handshake overhead
- **Monitor backend latency distribution** — p50, p90, p99 from each backend separately to catch degraded nodes
- **Circuit breaker** — stop sending requests to a backend that is returning errors at high rate

---

## Anti-Patterns to Avoid

- Session state stored in application memory — prevents horizontal scaling
- Not having health check endpoints — load balancer cannot detect failed backends
- Routing all traffic to one server "just for testing" and calling that load balancing
- Ignoring the difference between the load balancer being down and a backend being down
- Long-lived sticky sessions without fallback — if the pinned server dies, the session is lost

---

## Debugging & Troubleshooting

**"One server is getting all the traffic"**
Check if sticky sessions (ip_hash or cookie) are routing everything to one node. If so, the distribution is working correctly for the configured algorithm — but perhaps ip_hash is not appropriate here.

**"The load balancer is marking healthy servers as failed"**
Check the health endpoint response time. If it times out (due to a slow DB check), the load balancer considers the server failed. Either speed up the health check or increase the timeout threshold.

**"Traffic spikes overwhelm backends one at a time"**
Least connections or least response time algorithms distribute better under variable request duration. Round robin sends the same number of requests to each server regardless of how long each one takes.

---

## Real-World Scenarios

**Scenario: Deploying 3 API instances behind Nginx on a VPS**

```bash
# Scale up
docker compose up --scale api=3 -d

# Nginx automatically round-robins across all 3 instances
# When one fails max_fails times, Nginx removes it from the pool
# After fail_timeout, Nginx tries it again
```

**Scenario: Zero-downtime deployment with load balancing**

```bash
# Rolling deploy: bring up new instance, verify health, then remove old
# 1. Start new instance (api-v2)
docker run -d --name api-v2 myapp:v2

# 2. Add to upstream, verify it starts receiving traffic
# 3. Remove old instance from upstream
# 4. Stop old instance
docker stop api-v1
```

---

## Further Reading

- [Nginx upstream module documentation](https://nginx.org/en/docs/http/ngx_http_upstream_module.html)
- [Traefik load balancer documentation](https://doc.traefik.io/traefik/routing/services/)
- [AWS ALB vs NLB comparison](https://aws.amazon.com/elasticloadbalancing/features/)
- Track 08: [Database Scaling](database-scaling.md)
- Track 08: [API Gateways](api-gateways.md)

---

## Summary

Load balancing is the enabling technology for horizontal scaling. The algorithm choice matters: round robin works for homogeneous stateless services; least connections is better for variable-duration requests. Health checks are non-negotiable — without them, the balancer keeps sending requests to failed backends. Sticky sessions are a compatibility layer for stateful applications, but the better solution is to externalize state to Redis so every backend is identical. Layer 7 load balancing (Nginx, Traefik) is the right choice for HTTP APIs — it enables TLS termination, content-based routing, and cookie-based stickiness.
