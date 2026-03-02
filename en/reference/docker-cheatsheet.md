# Docker Cheatsheet

> Quick reference — use Ctrl+F to find what you need.

---

## Image Commands

```bash
docker images                         # list local images
docker pull nginx:alpine              # pull from registry
docker build -t myapp:1.0 .           # build from Dockerfile
docker build -t myapp:1.0 -f Dockerfile.prod .
docker tag myapp:1.0 registry/myapp:1.0
docker push registry/myapp:1.0
docker rmi myapp:1.0                  # remove image
docker image prune                    # remove dangling images
docker image prune -a                 # remove all unused images
docker history myapp:1.0              # show layer history
docker inspect myapp:1.0              # full metadata JSON
```

---

## Container Commands

```bash
# Run
docker run nginx                                # foreground
docker run -d nginx                             # detached (background)
docker run -it ubuntu bash                      # interactive TTY
docker run --rm ubuntu echo hello               # auto-remove on exit
docker run -p 8080:80 nginx                     # host:container port
docker run -p 127.0.0.1:8080:80 nginx          # bind to localhost only
docker run -e ENV_VAR=value nginx               # env variable
docker run --env-file .env nginx                # env file
docker run -v /host/path:/container/path nginx  # bind mount
docker run -v myvolume:/data nginx              # named volume
docker run --name mycontainer nginx             # named container
docker run --network mynet nginx                # custom network
docker run --memory 512m --cpus 1.5 nginx      # resource limits
docker run --restart unless-stopped nginx       # restart policy

# Lifecycle
docker start mycontainer
docker stop mycontainer           # SIGTERM, then SIGKILL after 10s
docker stop -t 30 mycontainer     # custom timeout
docker restart mycontainer
docker kill mycontainer           # immediate SIGKILL
docker pause mycontainer
docker unpause mycontainer
docker rm mycontainer             # remove stopped container
docker rm -f mycontainer          # force remove running container

# Info
docker ps                         # running containers
docker ps -a                      # all containers
docker logs mycontainer           # stdout/stderr
docker logs -f mycontainer        # follow logs
docker logs --tail 100 mycontainer
docker logs --since 10m mycontainer
docker stats                      # live resource usage
docker top mycontainer            # processes inside container
docker inspect mycontainer        # full metadata JSON
docker diff mycontainer           # filesystem changes

# Execute
docker exec mycontainer ls /app
docker exec -it mycontainer bash      # interactive shell
docker exec -u root mycontainer whoami

# Copy
docker cp mycontainer:/app/logs ./logs    # container → host
docker cp ./config.json mycontainer:/app  # host → container
```

---

## Dockerfile Directives

```dockerfile
# Base image
FROM node:22-alpine AS base

# Build arguments (available at build time)
ARG NODE_ENV=production

# Environment variables (available at runtime)
ENV PORT=3000 \
    NODE_ENV=production

# Working directory
WORKDIR /app

# Copy files (COPY <src> <dst>)
COPY package*.json ./
COPY --chown=node:node . .

# Run commands (creates a new layer)
RUN npm ci --only=production

# Expose port (documentation only — does not publish)
EXPOSE 3000

# Health check
HEALTHCHECK --interval=30s --timeout=5s --retries=3 \
  CMD curl -f http://localhost:3000/health || exit 1

# Volume mount point
VOLUME ["/data"]

# User to run as (never run as root in production)
USER node

# Default command (can be overridden)
CMD ["node", "dist/index.js"]

# Entrypoint (not easily overridden — use for executables)
ENTRYPOINT ["docker-entrypoint.sh"]
```

### Multi-Stage Build

```dockerfile
# Stage 1: build
FROM node:22 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci
COPY . .
RUN npm run build

# Stage 2: runtime (lean image)
FROM node:22-alpine AS runtime
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules
USER node
CMD ["node", "dist/index.js"]
```

### .dockerignore

```
node_modules
.git
.env
*.log
dist
coverage
.DS_Store
```

---

## Docker Compose

```yaml
# docker-compose.yml
version: "3.9"

services:
  api:
    build:
      context: .
      dockerfile: Dockerfile
      target: runtime
    image: myapp:latest
    container_name: myapp-api
    restart: unless-stopped
    ports:
      - "3000:3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgresql://user:pass@db:5432/mydb
    env_file:
      - .env
    volumes:
      - ./uploads:/app/uploads
    depends_on:
      db:
        condition: service_healthy
    networks:
      - internal
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:3000/health"]
      interval: 30s
      timeout: 5s
      retries: 3
    deploy:
      resources:
        limits:
          memory: 512m
          cpus: "1.0"

  db:
    image: postgres:16-alpine
    restart: unless-stopped
    environment:
      POSTGRES_USER: user
      POSTGRES_PASSWORD: pass
      POSTGRES_DB: mydb
    volumes:
      - pgdata:/var/lib/postgresql/data
    networks:
      - internal
    healthcheck:
      test: ["CMD", "pg_isready", "-U", "user"]
      interval: 10s
      timeout: 5s
      retries: 5

volumes:
  pgdata:

networks:
  internal:
    driver: bridge
```

### Compose Commands

```bash
docker compose up                    # start all services
docker compose up -d                 # detached
docker compose up --build            # rebuild images first
docker compose up api                # start specific service
docker compose down                  # stop and remove containers
docker compose down -v               # also remove volumes
docker compose ps                    # list services
docker compose logs                  # all logs
docker compose logs -f api           # follow specific service
docker compose exec api bash         # shell into service
docker compose run --rm api npm test # one-off command
docker compose pull                  # pull latest images
docker compose build                 # build all images
docker compose restart api           # restart service
docker compose stop                  # stop without removing
docker compose config                # validate and print config
```

---

## Networking

```bash
# Networks
docker network ls
docker network create mynet
docker network create --driver bridge --subnet 172.20.0.0/16 mynet
docker network inspect mynet
docker network connect mynet mycontainer
docker network disconnect mynet mycontainer
docker network rm mynet
docker network prune                 # remove unused networks

# Connect two containers
docker run -d --name db --network mynet postgres
docker run -d --name api --network mynet myapp
# api can reach db via hostname "db"
```

### Network Drivers

| Driver | Use Case |
|--------|---------|
| `bridge` | Default — containers on same host talk via virtual switch |
| `host` | Container shares host network stack (Linux only) |
| `none` | No networking |
| `overlay` | Multi-host (Docker Swarm / Compose with Swarm mode) |
| `macvlan` | Container gets its own MAC/IP on the physical network |

---

## Volumes

```bash
docker volume ls
docker volume create myvolume
docker volume inspect myvolume
docker volume rm myvolume
docker volume prune                  # remove unused volumes

# Backup a volume
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar czf /backup/myvolume.tar.gz /data

# Restore a volume
docker run --rm \
  -v myvolume:/data \
  -v $(pwd):/backup \
  alpine tar xzf /backup/myvolume.tar.gz -C /
```

---

## Registry & Image Management

```bash
# Login
docker login
docker login registry.example.com

# Tag and push
docker tag myapp:latest registry.example.com/myapp:1.0
docker push registry.example.com/myapp:1.0

# Save / load (without registry)
docker save myapp:1.0 | gzip > myapp.tar.gz
docker load < myapp.tar.gz

# Export / import (container filesystem)
docker export mycontainer | gzip > container.tar.gz
docker import container.tar.gz myimage:restored
```

---

## Cleanup

```bash
# Remove everything unused (safe)
docker system prune

# Remove everything including volumes
docker system prune --volumes -a

# Targeted cleanup
docker container prune     # stopped containers
docker image prune -a      # unused images
docker volume prune        # unused volumes
docker network prune       # unused networks

# Show disk usage
docker system df
docker system df -v        # verbose
```

---

## Production Patterns

```bash
# Always pin image versions (never use :latest in prod)
FROM postgres:16.2-alpine

# Read-only filesystem + tmpfs for writes
docker run --read-only --tmpfs /tmp myapp

# Drop capabilities
docker run --cap-drop ALL --cap-add NET_BIND_SERVICE myapp

# No new privileges
docker run --security-opt no-new-privileges myapp

# Non-root user (set in Dockerfile)
USER 1001

# Resource limits
docker run --memory 256m --memory-swap 256m --cpus 0.5 myapp

# Restart policy for production
restart: unless-stopped   # or: always, on-failure:5
```

---

## Useful One-Liners

```bash
# Stop all running containers
docker stop $(docker ps -q)

# Remove all stopped containers
docker rm $(docker ps -aq -f status=exited)

# Get container IP
docker inspect -f '{{range .NetworkSettings.Networks}}{{.IPAddress}}{{end}}' mycontainer

# Follow logs with timestamps
docker logs -f --timestamps mycontainer

# Copy file from image (without running)
docker create --name tmp myapp && docker cp tmp:/app/config.json . && docker rm tmp

# Shell into a running container as root
docker exec -u root -it mycontainer sh

# Watch container stats once
docker stats --no-stream
```
