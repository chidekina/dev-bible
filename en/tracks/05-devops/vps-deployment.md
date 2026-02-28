# VPS Deployment

## Overview

A VPS (Virtual Private Server) is a virtualized Linux server in a data center that you rent and fully control. Unlike managed platforms (Heroku, Railway, Render), a VPS gives you root access, full control over the software stack, and significantly lower costs at scale — at the expense of operational responsibility.

Deploying to a VPS means you own the entire stack: OS, web server, runtime, database, backups, monitoring, and security. This is more work than a PaaS, but the skills transfer everywhere, the costs are predictable, and there are no platform-specific constraints.

This chapter covers deploying a production Node.js/TypeScript application to a VPS using Docker, Nginx, and a GitHub Actions CI/CD pipeline.

---

## Prerequisites

- A VPS with Ubuntu 22.04/24.04 (DigitalOcean, Hetzner, Linode, Vultr, etc.)
- A domain name with DNS pointed to your VPS IP
- Basic Linux and Docker knowledge
- A GitHub repository with your application

Minimum recommended specs for a small production app: 2 vCPU, 2 GB RAM, 40 GB SSD.

---

## Core Concepts

### Deployment Architecture

```
GitHub (source code + CI/CD)
    |
    | push to main → GitHub Actions runs
    |
    ↓
Docker Hub / GHCR (container registry)
    |
    | docker pull (from VPS)
    |
    ↓
VPS (Ubuntu 22.04)
    ├── Nginx (reverse proxy, SSL termination)
    ├── Docker containers:
    │   ├── myapi:latest    (port 3000, internal)
    │   ├── postgres:16     (port 5432, internal)
    │   └── redis:7         (port 6379, internal)
    └── Certbot (Let's Encrypt, auto-renewal)
```

### Server Directory Structure

```
/opt/myapp/
  compose.yml              ← production docker compose file
  .env.production          ← secrets (600 permissions, not in git)
  backups/                 ← database backups
  logs/                    ← application logs (if bind-mounted)

/var/log/nginx/            ← Nginx access and error logs
/etc/nginx/sites-available/myapp  ← Nginx config
/etc/letsencrypt/          ← SSL certificates
```

### Deployment Strategies

| Strategy | Description | Downtime |
|----------|-------------|----------|
| Stop → Pull → Start | Simplest. Stop old, start new. | ~5-30 seconds |
| Blue-Green | Run two environments, switch routing | Zero |
| Rolling update | Replace instances one by one | Zero (with load balancer) |
| Canary | Route small % to new version first | Zero |

For a single VPS, Stop → Pull → Start is usually acceptable. For zero-downtime, you need two containers and a health-check-aware restart.

---

## Hands-On Examples

### Example 1: First-Time Server Setup

After getting VPS root access:

```bash
# 1. Update system
apt update && apt upgrade -y

# 2. Install essentials
apt install -y \
  curl wget git ufw fail2ban \
  htop ncdu tmux jq

# 3. Install Docker
curl -fsSL https://get.docker.com | sh

# 4. Create deploy user (never run apps as root)
useradd -m -s /bin/bash deploy
usermod -aG docker deploy
usermod -aG sudo deploy

# 5. Set up SSH key for deploy user
mkdir -p /home/deploy/.ssh
# Add your public key:
echo "ssh-ed25519 AAAA..." >> /home/deploy/.ssh/authorized_keys
chown -R deploy:deploy /home/deploy/.ssh
chmod 700 /home/deploy/.ssh
chmod 600 /home/deploy/.ssh/authorized_keys

# 6. Install Nginx
apt install -y nginx certbot python3-certbot-nginx

# 7. Create app directory
mkdir -p /opt/myapp
chown deploy:deploy /opt/myapp

# 8. Configure firewall (see vps-hardening.md for full hardening)
ufw allow ssh
ufw allow 80/tcp
ufw allow 443/tcp
ufw --force enable
```

### Example 2: Production Docker Compose

`/opt/myapp/compose.yml`:
```yaml
name: myapp

services:
  api:
    image: ghcr.io/yourusername/myapp:${VERSION:-latest}
    restart: unless-stopped
    expose:
      - "3000"
    environment:
      NODE_ENV: production
      DATABASE_URL: postgres://app:${POSTGRES_PASSWORD}@db:5432/app
      REDIS_URL: redis://cache:6379
      PORT: "3000"
    depends_on:
      db:
        condition: service_healthy
      cache:
        condition: service_healthy
    networks:
      - internal
    logging:
      driver: "json-file"
      options:
        max-size: "10m"
        max-file: "3"

  db:
    image: postgres:16-alpine
    restart: unless-stopped
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
    networks:
      - internal
    logging:
      driver: "json-file"
      options:
        max-size: "5m"
        max-file: "3"

  cache:
    image: redis:7-alpine
    restart: unless-stopped
    command: redis-server --maxmemory 128mb --maxmemory-policy allkeys-lru --save ""
    volumes:
      - redisdata:/data
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 5s
      timeout: 3s
      retries: 5
    networks:
      - internal

volumes:
  pgdata:
  redisdata:

networks:
  internal:
    driver: bridge
```

`/opt/myapp/.env.production` (permissions: 600, never committed to git):
```bash
POSTGRES_PASSWORD=use_a_long_random_string_here
```

### Example 3: Nginx Configuration

`/etc/nginx/sites-available/myapp`:
```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;
    return 301 https://$host$request_uri;
}

# HTTPS
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.com www.myapp.com;

    ssl_certificate     /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_cache shared:SSL:10m;
    ssl_session_timeout 1d;

    add_header Strict-Transport-Security "max-age=63072000" always;
    add_header X-Frame-Options SAMEORIGIN always;
    add_header X-Content-Type-Options nosniff always;

    # Logging
    access_log /var/log/nginx/myapp.access.log;
    error_log  /var/log/nginx/myapp.error.log;

    # API proxy
    location / {
        proxy_pass         http://127.0.0.1:3000;
        proxy_http_version 1.1;
        proxy_set_header   Host              $host;
        proxy_set_header   X-Real-IP         $remote_addr;
        proxy_set_header   X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header   X-Forwarded-Proto $scheme;
        proxy_set_header   Connection        "";

        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        client_max_body_size 10m;
    }
}
```

**Note:** The container's port 3000 is not published directly. We need a host port mapping for Nginx to reach it. Update compose.yml:

```yaml
api:
  ports:
    - "127.0.0.1:3000:3000"   # bind to loopback only — Nginx reaches it, internet cannot
```

Enable and get certificate:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t
sudo systemctl reload nginx
sudo certbot --nginx -d myapp.com -d www.myapp.com
```

### Example 4: GitHub Actions Deploy Workflow

`.github/workflows/deploy.yml`:
```yaml
name: Deploy

on:
  push:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    name: Test
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
      - run: npm test

  build:
    name: Build & Push Image
    runs-on: ubuntu-latest
    needs: test
    permissions:
      contents: read
      packages: write
    outputs:
      image-tag: ${{ steps.meta.outputs.version }}
    steps:
      - uses: actions/checkout@v4

      - uses: docker/setup-buildx-action@v3

      - uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}

      - id: meta
        uses: docker/metadata-action@v5
        with:
          images: ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}
          tags: |
            type=sha,prefix=sha-
            type=raw,value=latest

      - uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          tags: ${{ steps.meta.outputs.tags }}
          cache-from: type=gha
          cache-to: type=gha,mode=max
          target: production

  deploy:
    name: Deploy to Production
    runs-on: ubuntu-latest
    needs: build
    environment:
      name: production
      url: https://myapp.com
    steps:
      - name: Deploy via SSH
        uses: appleboy/ssh-action@v1
        with:
          host: ${{ secrets.VPS_HOST }}
          username: deploy
          key: ${{ secrets.VPS_SSH_KEY }}
          script: |
            set -e
            cd /opt/myapp

            # Authenticate with registry
            echo "${{ secrets.GITHUB_TOKEN }}" | \
              docker login ghcr.io -u "${{ github.actor }}" --password-stdin

            # Pull new image
            docker compose pull api

            # Run migrations (before switching traffic)
            docker compose run --rm \
              -e DATABASE_URL="$(grep DATABASE_URL .env.production | cut -d= -f2)" \
              api node dist/migrate.js

            # Restart with new image
            docker compose up -d --remove-orphans

            # Wait for health check
            sleep 10
            curl -f http://localhost:3000/health || (docker compose logs api && exit 1)

            # Clean up old images
            docker image prune -f
```

Required GitHub secrets:
- `VPS_HOST`: IP or hostname of your VPS
- `VPS_SSH_KEY`: Private SSH key (corresponding public key is in `deploy`'s `authorized_keys`)

### Example 5: Database Backup Script

`/opt/myapp/scripts/backup.sh`:
```bash
#!/usr/bin/env bash
set -euo pipefail

BACKUP_DIR="/opt/myapp/backups"
DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/pg_${DATE}.sql.gz"
KEEP_DAYS=7

mkdir -p "$BACKUP_DIR"

# Load env
source /opt/myapp/.env.production

# Dump database from running postgres container
docker exec myapp-db-1 \
  pg_dump -U app app \
  | gzip > "$BACKUP_FILE"

echo "Backup created: $BACKUP_FILE ($(du -sh $BACKUP_FILE | cut -f1))"

# Remove backups older than KEEP_DAYS
find "$BACKUP_DIR" -name "pg_*.sql.gz" -mtime +"$KEEP_DAYS" -delete
echo "Cleaned up backups older than $KEEP_DAYS days"
```

```bash
chmod +x /opt/myapp/scripts/backup.sh

# Add to crontab (as deploy user)
crontab -e
# Add:
0 2 * * * /opt/myapp/scripts/backup.sh >> /var/log/myapp-backup.log 2>&1
```

---

## Common Patterns & Best Practices

### Environment Variables Management

```bash
# Generate a random secret
openssl rand -base64 32

# On VPS, .env.production contains secrets
# Never commit .env.production to git
# Use a secrets manager (Doppler, Vault) for team environments
```

### Log Rotation

Docker containers write logs to `/var/lib/docker/containers/<id>/<id>-json.log`. Configure limits in compose.yml:

```yaml
logging:
  driver: "json-file"
  options:
    max-size: "10m"   # max 10 MB per log file
    max-file: "5"     # keep 5 files → max 50 MB per service
```

### Keeping the OS Updated

```bash
# Enable automatic security updates
apt install -y unattended-upgrades
dpkg-reconfigure -pmedium unattended-upgrades
```

### Zero-Downtime Deploy with Health Checks

```bash
# In deploy script — wait for new container to be healthy before finishing
docker compose up -d --remove-orphans

for i in {1..30}; do
  if curl -sf http://localhost:3000/health > /dev/null; then
    echo "Service healthy after ${i}s"
    exit 0
  fi
  sleep 1
done

echo "Service failed to become healthy in 30s"
docker compose logs api --tail 50
exit 1
```

---

## Anti-Patterns to Avoid

### Running Apps Directly (No Docker)

Running Node.js directly on the host mixes the app's dependencies with the OS, creates port conflicts across apps, and makes updates painful. Use Docker.

### Publishing Database Ports to the Internet

```yaml
# BAD — Postgres accessible from internet
db:
  ports:
    - "5432:5432"

# GOOD — only accessible within Docker network
db:
  expose:
    - "5432"
```

### Storing Secrets in Git

`.env.production` must be in `.gitignore`. Use GitHub Secrets for CI/CD values.

### Not Testing Before Deploy

Always run tests in CI before the deploy job. A failing test should prevent deployment.

### No Backups

Your database is your business. Automated daily backups with off-site storage (S3, B2) are non-negotiable for any production database.

### Deploying as Root

Never SSH as root for deployments. Create a `deploy` user with minimal permissions. If it's compromised, blast radius is contained.

---

## Debugging & Troubleshooting

### Check Container Status

```bash
cd /opt/myapp
docker compose ps
docker compose logs api --tail 100
docker compose logs db --tail 50
```

### Container Won't Start

```bash
docker compose up api   # run in foreground to see errors
docker compose run --rm api node --version   # test image basics
```

### Database Connection Issues

```bash
# Test from inside the api container
docker compose exec api sh
# Then:
nc -zv db 5432    # can we reach postgres?
env | grep DATABASE_URL
```

### Nginx Errors

```bash
sudo nginx -t                              # syntax check
sudo tail -f /var/log/nginx/myapp.error.log
sudo tail -f /var/log/nginx/myapp.access.log | grep " 5[0-9][0-9] "
```

### Disk Space

```bash
df -h              # overall disk usage
docker system df   # docker-specific usage
docker system prune -f    # remove unused containers, networks, images
du -sh /var/lib/docker/   # total docker disk usage
```

---

## Real-World Scenarios

### Scenario 1: Rolling Back a Bad Deploy

```bash
# On VPS — roll back to previous image version
cd /opt/myapp

# List recent images
docker images ghcr.io/yourusername/myapp --format "{{.Tag}}\t{{.CreatedAt}}"

# Roll back to specific SHA
VERSION=sha-abc1234 docker compose up -d api
```

### Scenario 2: Running Database Migrations Safely

```bash
# Test migration on a database snapshot first
# 1. Take a backup
./scripts/backup.sh

# 2. Run migration
docker compose run --rm api node dist/migrate.js

# 3. If it fails, restore
gunzip -c backups/pg_YYYYMMDD_HHMMSS.sql.gz | \
  docker exec -i myapp-db-1 psql -U app app
```

### Scenario 3: Monitoring Container Health

```bash
# Quick health overview
watch -n 5 'docker compose ps && echo "---" && docker stats --no-stream'

# Check if containers are using too much memory
docker stats --format "table {{.Name}}\t{{.MemUsage}}\t{{.CPUPerc}}" --no-stream
```

---

## Further Reading

- [DigitalOcean Tutorials](https://www.digitalocean.com/community/tutorials)
- [Hetzner Cloud Docs](https://docs.hetzner.com/cloud/)
- [Docker in Production](https://docs.docker.com/config/containers/start-containers-automatically/)
- [SSH Key Management](https://www.ssh.com/academy/ssh/keygen)

---

## Summary

| Step | Command / Action |
|------|-----------------|
| Initial setup | Create `deploy` user, install Docker + Nginx, configure UFW |
| App directory | `/opt/myapp/compose.yml` + `.env.production` (chmod 600) |
| Nginx | Reverse proxy on 443, HTTP redirect, certbot SSL |
| CI/CD | GitHub Actions: test → build image → push → SSH deploy |
| Secrets | VPS_HOST + VPS_SSH_KEY in GitHub Secrets |
| Backups | Cron job, `pg_dump` via docker exec, keep 7 days |
| Logging | Docker json-file driver with size/rotation limits |
| Rollback | `docker compose up -d` with previous image tag |

A VPS is a blank canvas. The patterns in this chapter give you a production-grade deployment that is repeatable, secure, and automatable. Start simple, automate everything, and add complexity only when needed.
