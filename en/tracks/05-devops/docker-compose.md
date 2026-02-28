# Docker Compose

## Overview

Docker Compose is a tool for defining and running multi-container applications using a single YAML file. Instead of running long `docker run` commands with dozens of flags, you declare your entire stack — API, database, cache, reverse proxy — and manage it with simple commands.

Compose is the bridge between "running a single container" and "running a real application." Real applications have dependencies: a Node.js API needs a Postgres database, a Redis cache, and maybe a background worker. Compose wires all of this together, manages the startup order, handles networking, and makes the whole stack reproducible.

This chapter covers Compose v2 (the modern version, built into Docker CLI as `docker compose`).

---

## Prerequisites

- Docker installed and working (see `docker-fundamentals.md`)
- Basic understanding of Docker images, containers, volumes, and networks

```bash
# Verify Compose v2 is available
docker compose version
# Docker Compose version v2.x.x
```

---

## Core Concepts

### The Compose File

A `docker-compose.yml` (or `compose.yml`) file declares services, volumes, and networks:

```yaml
name: myapp

services:
  api:
    build: .
    ports:
      - "3000:3000"
    environment:
      - DATABASE_URL=postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: secret
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

volumes:
  pgdata:
```

### Core Commands

```bash
docker compose up              # start all services (foreground)
docker compose up -d           # start detached (background)
docker compose up --build      # rebuild images before starting
docker compose down            # stop and remove containers
docker compose down -v         # also remove volumes
docker compose ps              # list running services
docker compose logs api        # logs for one service
docker compose logs -f         # follow all logs
docker compose exec api sh     # shell into running service
docker compose run api node migrate.js  # run one-off command
docker compose restart api     # restart one service
docker compose pull            # pull latest images
```

### Service Definition Keys

```yaml
services:
  myservice:
    # Image source (one of these):
    image: node:22-alpine          # pull from registry
    build: .                       # build from Dockerfile in .
    build:
      context: .
      dockerfile: Dockerfile.prod
      args:
        NODE_ENV: production
      target: production           # multi-stage target

    # Ports: HOST:CONTAINER
    ports:
      - "3000:3000"
      - "127.0.0.1:3000:3000"    # bind to loopback only (safer)

    # Environment
    environment:
      NODE_ENV: production
      PORT: 3000
    env_file:
      - .env.production            # file with KEY=VALUE lines

    # Volumes
    volumes:
      - ./src:/app/src             # bind mount
      - nodemodules:/app/node_modules  # named volume

    # Dependencies
    depends_on:
      db:
        condition: service_healthy  # wait for healthcheck to pass
      redis:
        condition: service_started  # just wait for container start

    # Restart policy
    restart: unless-stopped        # on failure, unless manually stopped

    # Resource limits (Compose v2 + Docker Swarm syntax)
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

    # Networks
    networks:
      - backend
      - frontend
```

### Networking in Compose

By default, Compose creates a single network for your project. All services on that network can reach each other by service name:

```yaml
services:
  api:
    image: myapi:latest
    # Can reach "db" by hostname "db"

  db:
    image: postgres:16-alpine
    # Can reach "api" by hostname "api"
```

Custom networks let you segment traffic:

```yaml
services:
  nginx:
    networks: [frontend, backend]
  api:
    networks: [backend]
  db:
    networks: [backend]
  # nginx can talk to api and db
  # api and db can talk to each other
  # nothing external can reach db directly

networks:
  frontend:
  backend:
    internal: true  # no outbound internet access
```

---

## Hands-On Examples

### Example 1: Full Node.js Stack

```yaml
# compose.yml
name: myapp

services:
  api:
    build:
      context: .
      target: production
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    restart: unless-stopped
    networks:
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}
      POSTGRES_DB: app
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app -d app"]
      interval: 5s
      timeout: 3s
      retries: 10
      start_period: 10s
    restart: unless-stopped
    networks:
      - backend

  cache:
    image: redis:7-alpine
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    restart: unless-stopped
    networks:
      - backend

volumes:
  pgdata:
  redisdata:

networks:
  backend:
```

`.env` file (referenced by `${POSTGRES_PASSWORD}`):
```bash
POSTGRES_PASSWORD=changeme_in_production
```

```bash
docker compose up -d
docker compose logs -f api
curl http://localhost:3000/health
```

### Example 2: Development vs Production Overrides

Use a base file + override files to share common config:

`compose.yml` (base):
```yaml
name: myapp

services:
  api:
    build: .
    environment:
      NODE_ENV: development
    networks:
      - backend

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - backend

networks:
  backend:
```

`compose.override.yml` (development — loaded automatically):
```yaml
services:
  api:
    build:
      target: development
    volumes:
      - ./src:/app/src
    ports:
      - "3000:3000"
      - "9229:9229"    # Node.js debugger
    environment:
      NODE_ENV: development
    command: tsx watch src/index.ts

  db:
    ports:
      - "5432:5432"    # expose to host for DB tools
```

`compose.prod.yml` (production):
```yaml
services:
  api:
    image: myapi:${VERSION:-latest}
    ports:
      - "127.0.0.1:3000:3000"   # only loopback, nginx proxies
    restart: unless-stopped
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'

  db:
    # No port exposure in production
    restart: unless-stopped
```

Usage:
```bash
# Development (compose.yml + compose.override.yml auto-merged)
docker compose up

# Production (explicit file selection)
docker compose -f compose.yml -f compose.prod.yml up -d
```

### Example 3: Database Migrations as Init Container

```yaml
services:
  migrate:
    image: myapi:latest
    command: node dist/migrate.js
    environment:
      DATABASE_URL: postgres://app:secret@db:5432/app
    depends_on:
      db:
        condition: service_healthy
    restart: "no"   # run once and exit
    networks:
      - backend

  api:
    image: myapi:latest
    depends_on:
      migrate:
        condition: service_completed_successfully
    ports:
      - "3000:3000"
    networks:
      - backend

  db:
    image: postgres:16-alpine
    # ... healthcheck config
    networks:
      - backend
```

Compose waits for `migrate` to exit with code 0 before starting `api`.

### Example 4: Multi-Service Development Environment with Tools

```yaml
name: myapp-dev

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile.dev
    volumes:
      - .:/app
      - node_modules:/app/node_modules
    ports:
      - "3000:3000"
      - "9229:9229"
    environment:
      DATABASE_URL: postgres://app:dev@db:5432/app
      REDIS_URL: redis://cache:6379
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: dev
      POSTGRES_DB: app
    ports:
      - "5432:5432"
    volumes:
      - pgdata:/var/lib/postgresql/data
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 5s
      timeout: 3s
      retries: 5

  cache:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  mailpit:
    image: axllent/mailpit:latest
    ports:
      - "1025:1025"    # SMTP
      - "8025:8025"    # Web UI
    environment:
      MP_SMTP_AUTH_ALLOW_INSECURE: true

  pgadmin:
    image: dpage/pgadmin4:latest
    ports:
      - "5050:80"
    environment:
      PGADMIN_DEFAULT_EMAIL: dev@local.dev
      PGADMIN_DEFAULT_PASSWORD: dev

volumes:
  pgdata:
  node_modules:
```

This gives you: API with hot reload, Postgres, Redis, email catching (Mailpit), and a DB GUI (pgAdmin) — all with one `docker compose up`.

---

## Common Patterns & Best Practices

### Always Use Healthchecks

`depends_on` without `condition: service_healthy` only waits for the container to start, not for the service inside to be ready. Postgres takes a few seconds to accept connections after startup.

```yaml
db:
  image: postgres:16-alpine
  healthcheck:
    test: ["CMD-SHELL", "pg_isready -U ${POSTGRES_USER} -d ${POSTGRES_DB}"]
    interval: 5s
    timeout: 3s
    retries: 10
    start_period: 10s  # grace period before first check
```

### Use Named Volumes for Persistent Data

```yaml
volumes:
  pgdata:        # Docker manages the location
  redisdata:

services:
  db:
    volumes:
      - pgdata:/var/lib/postgresql/data
```

Named volumes survive `docker compose down`. Use `docker compose down -v` only when you want to wipe data.

### Separate .env Files per Environment

```
.env              # defaults (safe to commit — no secrets)
.env.local        # local overrides (gitignored)
.env.production   # production values (managed via secrets manager)
```

```yaml
# compose.yml
env_file:
  - .env
  - .env.local    # overrides .env
```

### Profile-Based Conditional Services

```yaml
services:
  api:
    # always runs
    image: myapi:latest

  pgadmin:
    profiles: [tools]       # only runs with --profile tools
    image: dpage/pgadmin4

  mailpit:
    profiles: [tools]
    image: axllent/mailpit
```

```bash
# Normal start — only api
docker compose up -d

# With dev tools
docker compose --profile tools up -d
```

### Pin Image Versions

```yaml
# BAD
image: postgres:latest

# GOOD
image: postgres:16.2-alpine3.19
```

---

## Anti-Patterns to Avoid

### Hardcoding Secrets in compose.yml

```yaml
# BAD — committed to git
environment:
  DATABASE_URL: postgres://root:HARDCODED_SECRET@db/app

# GOOD — from environment or .env file (gitignored)
environment:
  DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db/app
```

### No Healthchecks on Databases

Without healthchecks, `depends_on` is effectively useless for databases — your API starts before Postgres is ready and crashes with "connection refused."

### Storing Node Modules in Bind Mounts

```yaml
# BAD — host node_modules leaks into container or vice versa
volumes:
  - .:/app

# GOOD — anonymous volume for node_modules hides host directory
volumes:
  - .:/app
  - /app/node_modules   # empty anonymous volume masks the bind mount
```

Or even better — use a named volume:
```yaml
volumes:
  - .:/app
  - nodemodules:/app/node_modules

volumes:
  nodemodules:
```

### Running Migrations Inside the API Service on Start

Mixing "run migrations" with "start server" in CMD creates race conditions when you scale to multiple replicas. Use a separate `migrate` service or a pre-deploy hook.

### Using `docker-compose` (v1) Instead of `docker compose` (v2)

v1 (`docker-compose`) is deprecated. v2 is built into the Docker CLI and significantly faster. Use `docker compose` (with a space).

---

## Debugging & Troubleshooting

### Check Service Status

```bash
docker compose ps
docker compose ps --format json | jq '.[] | {name, state, health}'
```

### View Logs

```bash
docker compose logs                     # all services
docker compose logs api                 # single service
docker compose logs api db              # multiple services
docker compose logs -f --tail 50 api    # follow last 50 lines
docker compose logs --since 5m          # last 5 minutes
```

### Exec Into a Service

```bash
docker compose exec api sh
docker compose exec db psql -U app -d app
docker compose exec cache redis-cli
```

### Run One-Off Commands

```bash
# Runs a new container, then removes it
docker compose run --rm api node dist/seed.js
docker compose run --rm api sh -c "node dist/migrate.js && node dist/seed.js"
```

### Inspect Network Connectivity

```bash
# From inside api container, can it reach db?
docker compose exec api sh
# Inside shell:
wget -qO- http://db:5432 || echo "can't reach"
nc -zv db 5432
```

### Reset Everything

```bash
docker compose down -v --remove-orphans  # stop, remove containers+volumes+orphans
docker compose build --no-cache          # force full rebuild
docker compose up -d
```

---

## Real-World Scenarios

### Scenario 1: Zero-Downtime Deployment with Health Checks

```yaml
services:
  api:
    image: myapi:${VERSION}
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 10s
      timeout: 5s
      retries: 3
      start_period: 20s
    restart: unless-stopped
```

With a reverse proxy (nginx) in front, you can deploy a new version alongside the old one, wait for healthchecks to pass, then switch traffic.

### Scenario 2: Seeding a Development Database

```yaml
services:
  db-seed:
    image: myapi:latest
    command: node dist/seed.js
    profiles: [seed]
    depends_on:
      db:
        condition: service_healthy
    environment:
      DATABASE_URL: postgres://app:dev@db/app
    restart: "no"
```

```bash
# Seed after first start
docker compose --profile seed run --rm db-seed
```

### Scenario 3: CI Environment

`compose.ci.yml`:
```yaml
services:
  test:
    build:
      context: .
      target: builder
    command: npm test
    environment:
      DATABASE_URL: postgres://app:ci@db/app_test
      NODE_ENV: test
    depends_on:
      db:
        condition: service_healthy

  db:
    image: postgres:16-alpine
    environment:
      POSTGRES_USER: app
      POSTGRES_PASSWORD: ci
      POSTGRES_DB: app_test
    tmpfs:
      - /var/lib/postgresql/data   # in-memory DB for speed in CI
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U app"]
      interval: 3s
      timeout: 3s
      retries: 10
```

```bash
# In CI pipeline
docker compose -f compose.ci.yml run --rm test
docker compose -f compose.ci.yml down -v
```

---

## Further Reading

- [Compose Specification](https://compose-spec.io/)
- [Docker Compose CLI Reference](https://docs.docker.com/compose/reference/)
- [Compose file reference — version 3](https://docs.docker.com/compose/compose-file/)
- [Multi-environment setups](https://docs.docker.com/compose/extends/)

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| `docker compose up -d` | Start all services detached |
| `depends_on` + healthcheck | Wait for DB to be truly ready, not just started |
| Override files | Share base config; override per environment |
| Named volumes | Persist data across `docker compose down` |
| Profiles | Conditional services (tools, seed, etc.) |
| `env_file` | Separate secrets from the compose file |
| One-off commands | `docker compose run --rm service command` |
| Migrations | Separate `migrate` service, not inside CMD |
| CI | Use `tmpfs` for DB volumes to speed up tests |

Compose is the foundation of local development and a valid production solution for single-host deployments. Learn it deeply before reaching for Kubernetes — most applications don't need an orchestrator.
