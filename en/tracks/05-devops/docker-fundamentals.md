# Docker Fundamentals

## Overview

Docker is a containerization platform that packages applications and their dependencies into portable, isolated units called containers. Unlike virtual machines, containers share the host OS kernel, making them lightweight, fast to start, and consistent across environments.

The core promise of Docker: **"works on my machine" becomes "works everywhere."** You define the exact environment your application needs — OS libraries, runtime version, configuration — and Docker guarantees that environment is identical in development, CI, staging, and production.

This chapter covers Docker from first principles: images, containers, the build system, networking, volumes, and the mental models you need to use Docker effectively in production.

---

## Prerequisites

- Linux command line basics (see `01-foundations/linux-shell.md`)
- Basic understanding of processes and filesystems
- Node.js/TypeScript familiarity for the examples

Install Docker:
```bash
# Ubuntu/Debian
curl -fsSL https://get.docker.com | sh
sudo usermod -aG docker $USER
newgrp docker

# Verify
docker --version
docker run hello-world
```

---

## Core Concepts

### Images vs Containers

An **image** is a read-only template — a layered filesystem snapshot with metadata. A **container** is a running instance of an image, with a writable layer on top.

```
Image (read-only layers, bottom to top):
  Layer 4: COPY . .          ← your app code
  Layer 3: RUN npm install   ← node_modules
  Layer 2: FROM node:22      ← Node.js runtime
  Layer 1: FROM debian:slim  ← OS base (pulled from Docker Hub)

Container = Image layers + writable layer + PID namespace + network namespace
```

Multiple containers can run from the same image simultaneously. Each gets its own isolated writable layer, but all share the read-only image layers. This is why containers start in milliseconds — no full OS boot, no copying layers.

### The Union Filesystem

Docker uses overlay filesystems (OverlayFS on modern Linux). Each instruction in a Dockerfile creates a new read-only layer. Layers are content-addressed (SHA256) and cached — if the layer content hasn't changed, Docker reuses it during builds.

```
Layer caching rules:
- FROM        → cached unless base image digest changed
- RUN         → cached if the command string is identical
- COPY/ADD    → cached if file content checksum is identical
- ARG         → invalidates all subsequent layers if value changes
```

Understanding cache invalidation is critical for fast builds. Once a layer is invalidated, all subsequent layers are rebuilt.

### The Dockerfile

A Dockerfile is an ordered script that defines how to build an image. Each instruction becomes a layer:

```dockerfile
# Minimal Node.js Dockerfile
FROM node:22-alpine

WORKDIR /app

# Copy manifests first — cached unless package.json changes
COPY package*.json ./

# Install deps — cached if manifests haven't changed
RUN npm ci --only=production

# Copy application code — always re-runs on code changes
COPY src/ ./src/

EXPOSE 3000

USER node

CMD ["node", "src/index.js"]
```

### Namespaces and Cgroups

Containers are not VMs — they are Linux processes with restrictions:

| Mechanism | What it isolates |
|-----------|-----------------|
| PID namespace | Process tree (container can't see host processes) |
| Network namespace | Network interfaces, routing, ports |
| Mount namespace | Filesystem view |
| UTS namespace | Hostname and domain name |
| User namespace | UID/GID mapping |
| Cgroups | CPU, memory, disk I/O limits |
| Seccomp | Allowed system calls |

### Container Lifecycle

```
docker create → Created
docker start  → Running ←→ Paused (docker pause/unpause)
docker stop   → Stopped (SIGTERM + 10s grace + SIGKILL)
docker kill   → Stopped (immediate SIGKILL)
docker rm     → Removed
```

```bash
docker run nginx             # create + start (most common)
docker run -d nginx          # detached (background)
docker run --rm nginx        # remove container when it exits
docker stop mycontainer      # graceful shutdown
docker rm mycontainer        # remove stopped container
docker rm -f mycontainer     # force stop + remove
```

---

## Hands-On Examples

### Example 1: Containerizing a Node.js API

Project structure:
```
myapi/
  src/
    index.ts
  package.json
  tsconfig.json
  Dockerfile
  .dockerignore
```

`src/index.ts`:
```typescript
import Fastify from 'fastify';

const app = Fastify({ logger: true });

app.get('/health', async () => ({
  status: 'ok',
  uptime: process.uptime(),
  version: process.env.APP_VERSION ?? 'dev',
}));

app.get('/users', async () => [
  { id: 1, name: 'Alice' },
  { id: 2, name: 'Bob' },
]);

const start = async () => {
  await app.listen({ port: 3000, host: '0.0.0.0' });
};

start().catch(console.error);
```

`.dockerignore` (always create this before your first COPY):
```
node_modules
dist
.git
*.log
.env
.env.*
coverage
.nyc_output
*.test.ts
*.spec.ts
```

`Dockerfile` using multi-stage build:
```dockerfile
# ─── Stage 1: Build ───────────────────────────────────────────────
FROM node:22-alpine AS builder

WORKDIR /app

COPY package*.json tsconfig.json ./
RUN npm ci

COPY src/ ./src/
RUN npm run build

# ─── Stage 2: Production ──────────────────────────────────────────
FROM node:22-alpine AS production

# Security: non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

COPY package*.json ./
RUN npm ci --only=production && npm cache clean --force

COPY --from=builder /app/dist ./dist

RUN chown -R appuser:appgroup /app

USER appuser

EXPOSE 3000

# Orchestrators use this to decide when the container is healthy
HEALTHCHECK --interval=30s --timeout=3s --start-period=5s --retries=3 \
  CMD wget --quiet --tries=1 --spider http://localhost:3000/health || exit 1

CMD ["node", "dist/index.js"]
```

Build and run:
```bash
# Build
docker build -t myapi:latest .

# Run with resource limits
docker run -d \
  --name myapi \
  -p 3000:3000 \
  -e APP_VERSION=1.0.0 \
  --memory=256m \
  --cpus=0.5 \
  --restart=unless-stopped \
  myapi:latest

# Verify
curl http://localhost:3000/health

# Logs
docker logs -f myapi
```

### Example 2: Environment Variables

```bash
# Inject single variable
docker run -e DATABASE_URL="postgres://user:pass@host/db" myapi:latest

# Inject from file (one KEY=VALUE per line, no quotes needed)
docker run --env-file .env.production myapi:latest

# Pass through from host environment
export DATABASE_URL="postgres://..."
docker run -e DATABASE_URL myapi:latest   # passes $DATABASE_URL value
```

Never bake secrets into images — they end up in `docker history` and registries.

### Example 3: Volumes

```bash
# Named volume — Docker manages the location on host
docker volume create appdata
docker run -v appdata:/app/data myapi:latest

# Bind mount — exact host path mapped into container
docker run -v $(pwd)/uploads:/app/uploads myapi:latest

# Read-only bind mount — container cannot modify
docker run -v $(pwd)/config:/app/config:ro myapi:latest

# tmpfs — in-memory, not persisted, ideal for secrets/temp files
docker run --tmpfs /tmp:size=100m myapi:latest
```

Inspect volumes:
```bash
docker volume ls
docker volume inspect appdata
docker volume rm appdata
```

### Example 4: Networking

```bash
# Create isolated user-defined network
docker network create backend

# Containers on the same network resolve each other by service name
docker run -d --name postgres --network backend \
  -e POSTGRES_PASSWORD=secret postgres:16-alpine

docker run -d --name myapi --network backend \
  -p 3000:3000 \
  -e DATABASE_URL="postgres://postgres:secret@postgres:5432/app" \
  myapi:latest

# postgres is reachable inside myapi by hostname "postgres"
```

Network types:
| Mode | Description |
|------|-------------|
| `bridge` | Default. Isolated NAT network per container |
| User-defined bridge | Custom network; containers can resolve by name |
| `host` | Shares host network stack. No isolation, but faster |
| `none` | No networking at all |

---

## Common Patterns & Best Practices

### Multi-Stage Builds

Multi-stage builds produce small production images by discarding build tools:

```dockerfile
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build && npm run test

FROM node:22-alpine AS production
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY --from=builder /app/dist ./dist
USER node
CMD ["node", "dist/index.js"]
```

Result: builder image ~1.2 GB, production image ~120 MB.

### Layer Order for Maximum Cache Hits

```dockerfile
# BAD — code change invalidates npm install
COPY . .
RUN npm ci

# GOOD — npm install cached until package.json changes
COPY package*.json ./
RUN npm ci
COPY . .
```

The rule: copy things that change less frequently first.

### Non-Root Users

```dockerfile
# Alpine
RUN addgroup -S app && adduser -S app -G app
USER app

# Debian/Ubuntu
RUN groupadd --system app && useradd --system --gid app --no-create-home app
USER app
```

### HEALTHCHECK

Always define a healthcheck. Orchestrators (Docker Compose, Swarm, Kubernetes) use it to know when a container is ready and to restart unhealthy ones:

```dockerfile
HEALTHCHECK --interval=30s --timeout=5s --start-period=10s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1
```

### Image Tagging

```bash
# Tie image tags to git refs for traceability
GIT_SHA=$(git rev-parse --short HEAD)
docker build \
  -t myapi:${GIT_SHA} \
  -t myapi:latest \
  .
```

### Resource Limits

```bash
docker run \
  --memory=512m \              # hard memory limit — OOM killed if exceeded
  --memory-reservation=256m \ # soft limit for scheduling
  --cpus=1.0 \                 # 1.0 = one full CPU core
  --pids-limit=100 \           # prevents fork bombs
  myapi:latest
```

---

## Anti-Patterns to Avoid

### Running as Root

A compromised container running as root gives an attacker root-equivalent capabilities. Always drop to a non-root user before CMD.

### Secrets in Images

```dockerfile
# BAD — visible in docker history, pushed to registry
ENV DATABASE_URL="postgres://admin:secret@prod-db/app"

# GOOD — injected at runtime, never stored in image
# docker run -e DATABASE_URL=... or --env-file
```

### Pinning to `latest`

```yaml
# BAD — unpredictable, can break on any deploy
image: postgres:latest

# GOOD — deterministic
image: postgres:16.2-alpine
```

### Fat Images

```dockerfile
# BAD
RUN apt-get install -y build-essential curl wget vim git

# GOOD — install only what the container needs at runtime
RUN apt-get install -y --no-install-recommends curl \
  && rm -rf /var/lib/apt/lists/*
```

### No .dockerignore

Without `.dockerignore`, `COPY . .` sends `node_modules`, `.git`, `.env`, and logs to the build daemon, inflating build context and potentially leaking secrets.

### PID 1 Signal Problem

Node.js does not handle SIGTERM well when it's PID 1. Use `tini`:

```dockerfile
RUN apk add --no-cache tini
ENTRYPOINT ["/sbin/tini", "--"]
CMD ["node", "dist/index.js"]
```

Or handle signals in your application:
```typescript
process.on('SIGTERM', () => {
  server.close(() => {
    console.log('Graceful shutdown complete');
    process.exit(0);
  });
});
```

---

## Debugging & Troubleshooting

### Get a Shell Inside a Running Container

```bash
docker exec -it myapi sh      # alpine (no bash)
docker exec -it myapi bash    # debian/ubuntu
docker exec myapi env         # print environment
docker exec myapi cat /etc/hosts
```

### Read Logs

```bash
docker logs myapi               # all logs
docker logs myapi --tail 50     # last 50 lines
docker logs myapi -f            # follow in real time
docker logs myapi --since 10m   # last 10 minutes
docker logs myapi 2>&1 | grep ERROR
```

### Inspect Metadata

```bash
docker inspect myapi                                    # full JSON
docker inspect myapi | jq '.[0].State'                 # state info
docker inspect myapi | jq '.[0].HostConfig.Memory'     # memory limit
docker inspect myapi | jq '.[0].NetworkSettings'       # network info
```

### Debug a Stopped Container

```bash
docker ps -a                          # list all including stopped
docker logs <container-id>            # last output before crash
docker inspect <id> | jq '.[0].State.ExitCode'

# Common exit codes:
# 1   — application error (check logs)
# 126 — command not executable (permission denied)
# 127 — command not found (wrong CMD path)
# 137 — SIGKILL / OOM killed (increase --memory)
# 143 — SIGTERM (clean stop)
```

### Analyze Image Layers

```bash
docker history myapi:latest            # layer sizes
docker history --no-trunc myapi:latest # full commands

# Install dive for detailed layer inspection
# https://github.com/wagoodman/dive
dive myapi:latest
```

### Reclaim Disk Space

```bash
docker system df              # show disk usage
docker system prune           # remove stopped containers + dangling images
docker system prune -a        # also remove unused images
docker volume prune           # remove unused volumes
```

---

## Real-World Scenarios

### Scenario 1: Development with Hot Reload

```dockerfile
# Dockerfile.dev
FROM node:22-alpine
WORKDIR /app
RUN npm install -g tsx
COPY package*.json ./
RUN npm install
# Code is bind-mounted at runtime — don't COPY src
CMD ["tsx", "watch", "src/index.ts"]
```

```bash
docker run -it --rm \
  -v $(pwd)/src:/app/src:ro \
  -p 3000:3000 \
  myapi:dev
```

Changes to `src/` on the host are immediately reflected in the container without rebuilding.

### Scenario 2: One-Off Commands (Migrations)

```bash
# Run migration then exit
docker run --rm \
  -e DATABASE_URL="$DATABASE_URL" \
  myapi:latest \
  node dist/migrate.js

# Open a REPL against production database (read-only replica)
docker run --rm -it \
  -e DATABASE_URL="$REPLICA_URL" \
  myapi:latest \
  node -e "require('./dist/repl')"
```

### Scenario 3: Build-Time Arguments for Multiple Environments

```dockerfile
ARG APP_ENV=production
ENV APP_ENV=${APP_ENV}

RUN if [ "$APP_ENV" = "staging" ]; then \
      cp config.staging.json config.json; \
    else \
      cp config.production.json config.json; \
    fi
```

```bash
docker build --build-arg APP_ENV=staging -t myapi:staging .
docker build --build-arg APP_ENV=production -t myapi:prod .
```

### Scenario 4: Passing NPM Credentials Securely

```dockerfile
# syntax=docker/dockerfile:1.4
FROM node:22-alpine AS builder
RUN --mount=type=secret,id=npmrc,target=/root/.npmrc \
    npm install
```

```bash
docker build --secret id=npmrc,src=$HOME/.npmrc .
```

The `.npmrc` is mounted only during the `RUN` step and never stored in any layer.

---

## Further Reading

- [Dockerfile Best Practices — Docker Docs](https://docs.docker.com/develop/develop-images/dockerfile_best-practices/)
- [dive — image layer explorer](https://github.com/wagoodman/dive)
- [tini — minimal init for containers](https://github.com/krallin/tini)
- [Docker Security Cheat Sheet — OWASP](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [BuildKit documentation](https://docs.docker.com/build/buildkit/)

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Images | Layered, read-only, content-addressed snapshots. Shared between containers. |
| Containers | Running image + writable layer + isolated Linux namespaces |
| Dockerfile | Ordered instructions. Order determines cache efficiency. |
| Multi-stage builds | Keep production images small — build tools stay in builder stage |
| Non-root users | Always run as non-root in production |
| HEALTHCHECK | Define one so orchestrators can manage container lifecycle |
| Resource limits | Always set `--memory` and `--cpus` in production |
| .dockerignore | Create it before your first COPY |
| Secrets | Inject at runtime — never bake into image layers |
| PID 1 | Use tini or handle SIGTERM explicitly to enable graceful shutdown |

Docker removes "works on my machine" by making the machine part of the artifact. Master layer caching, keep production images minimal, and never run as root.
