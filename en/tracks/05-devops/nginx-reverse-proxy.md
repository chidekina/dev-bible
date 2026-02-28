# Nginx & Reverse Proxy

## Overview

Nginx (pronounced "engine-x") is a high-performance web server and reverse proxy. In modern deployments, it sits in front of your application and handles the concerns that applications shouldn't manage directly: SSL termination, request routing, static file serving, load balancing, rate limiting, and connection management.

A **reverse proxy** is a server that receives client requests and forwards them to one or more backend services. Clients talk to Nginx; Nginx talks to your app. This indirection provides security (hide internal topology), flexibility (swap backends transparently), and performance (handle thousands of concurrent connections efficiently).

This chapter covers Nginx as a reverse proxy for Node.js/TypeScript applications in production, including HTTPS with Let's Encrypt, load balancing, and Docker integration.

---

## Prerequisites

- A Linux server (Ubuntu 22.04 / 24.04 recommended)
- Basic understanding of HTTP, DNS, and ports
- Docker and Docker Compose (for the Docker-integrated sections)

```bash
# Install Nginx (Ubuntu)
sudo apt update
sudo apt install -y nginx

# Verify
nginx -v
systemctl status nginx
```

---

## Core Concepts

### How a Reverse Proxy Works

```
Client (browser)
    |
    | HTTPS :443
    v
Nginx (reverse proxy)
    |
    | HTTP :3000 (internal)
    v
Node.js Application
    |
    | TCP :5432 (internal)
    v
PostgreSQL Database
```

The client never directly reaches Node.js. All traffic goes through Nginx, which:
1. Terminates TLS (decrypts HTTPS)
2. Validates the request
3. Forwards it to the upstream application
4. Returns the response

### Key Nginx Concepts

| Concept | Description |
|---------|-------------|
| **server block** | Virtual host — handles requests for a specific domain |
| **location block** | Routes specific URL paths to different handlers |
| **upstream** | Defines a pool of backend servers for proxying/load balancing |
| **proxy_pass** | Forwards a request to an upstream server |
| **listen** | Port and protocol the server block accepts |
| **server_name** | Domain name(s) this block handles |

### Configuration File Structure

```
/etc/nginx/
  nginx.conf               ← main config (includes others)
  conf.d/                  ← drop-in site configs
    myapp.conf
  sites-available/         ← available site configs
    myapp
  sites-enabled/           ← symlinks to sites-available
    myapp -> ../sites-available/myapp
  snippets/                ← reusable config fragments
    ssl-params.conf
```

Debian/Ubuntu convention: use `sites-available` + `sites-enabled`. Other distros: use `conf.d/` directly.

---

## Hands-On Examples

### Example 1: Basic Reverse Proxy

`/etc/nginx/sites-available/myapp`:
```nginx
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;

    location / {
        proxy_pass http://localhost:3000;

        # Forward real client info to the app
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Timeouts
        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;
    }
}
```

Enable and test:
```bash
sudo ln -s /etc/nginx/sites-available/myapp /etc/nginx/sites-enabled/
sudo nginx -t           # test configuration syntax
sudo systemctl reload nginx
```

### Example 2: HTTPS with Let's Encrypt (Certbot)

```bash
# Install Certbot
sudo apt install -y certbot python3-certbot-nginx

# Obtain and install certificate (modifies Nginx config automatically)
sudo certbot --nginx -d myapp.com -d www.myapp.com

# Verify auto-renewal
sudo certbot renew --dry-run
```

Manual HTTPS configuration (after certbot installs the cert):
```nginx
# HTTP → HTTPS redirect
server {
    listen 80;
    listen [::]:80;
    server_name myapp.com www.myapp.com;
    return 301 https://$host$request_uri;
}

# HTTPS server
server {
    listen 443 ssl http2;
    listen [::]:443 ssl http2;
    server_name myapp.com www.myapp.com;

    # SSL certificate (managed by certbot)
    ssl_certificate     /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    # Modern SSL configuration
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_prefer_server_ciphers off;
    ssl_session_timeout 1d;
    ssl_session_cache shared:SSL:10m;
    ssl_session_tickets off;

    # HSTS (6 months, include subdomains)
    add_header Strict-Transport-Security "max-age=15768000; includeSubDomains" always;

    # Security headers
    add_header X-Content-Type-Options   "nosniff" always;
    add_header X-Frame-Options          "SAMEORIGIN" always;
    add_header X-XSS-Protection         "1; mode=block" always;
    add_header Referrer-Policy          "strict-origin-when-cross-origin" always;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        proxy_connect_timeout 10s;
        proxy_send_timeout    60s;
        proxy_read_timeout    60s;

        # Buffer settings for performance
        proxy_buffering    on;
        proxy_buffer_size  4k;
        proxy_buffers      8 4k;
    }
}
```

### Example 3: Serving Static Files + API

```nginx
server {
    listen 443 ssl http2;
    server_name myapp.com;

    # ... ssl config ...

    root /var/www/myapp/dist;
    index index.html;

    # Static assets — serve directly with aggressive caching
    location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf)$ {
        expires 1y;
        add_header Cache-Control "public, immutable";
        try_files $uri =404;
    }

    # API — proxy to backend
    location /api/ {
        proxy_pass http://localhost:3000;
        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # SPA fallback — serve index.html for all non-file routes
    location / {
        try_files $uri $uri/ /index.html;
    }
}
```

### Example 4: Load Balancing Multiple Instances

```nginx
# Define upstream pool
upstream api_pool {
    least_conn;   # send to server with fewest active connections

    server localhost:3001;
    server localhost:3002;
    server localhost:3003;

    # Health checking (requires Nginx Plus, or use open-source alternative)
    keepalive 32;  # maintain persistent connections to upstreams
}

server {
    listen 443 ssl http2;
    server_name myapp.com;

    location /api/ {
        proxy_pass http://api_pool;

        proxy_set_header Host              $host;
        proxy_set_header X-Real-IP         $remote_addr;
        proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;

        # Enable keepalive to upstream
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
```

Load balancing methods:
```nginx
upstream api_pool {
    # round_robin (default) — distributes evenly
    server localhost:3001;
    server localhost:3002;

    # least_conn — fewest active connections
    least_conn;

    # ip_hash — same client always hits same server (session affinity)
    ip_hash;

    # weighted — server 1 gets 3x more traffic than server 2
    server localhost:3001 weight=3;
    server localhost:3002 weight=1;

    # backup — only used if all primary servers are down
    server localhost:3003 backup;
}
```

### Example 5: WebSocket Support

```nginx
server {
    listen 443 ssl http2;
    server_name myapp.com;

    location /ws/ {
        proxy_pass http://localhost:3000;
        proxy_http_version 1.1;
        proxy_set_header Upgrade    $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host       $host;

        # WebSocket connections are long-lived
        proxy_read_timeout  3600s;
        proxy_send_timeout  3600s;
    }
}
```

### Example 6: Rate Limiting

```nginx
# Define rate limit zone (10 MB storage, 10 requests/second per IP)
limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
limit_req_zone $binary_remote_addr zone=login:10m rate=5r/m;

server {
    listen 443 ssl http2;
    server_name myapp.com;

    # Apply general rate limit
    location /api/ {
        limit_req zone=api burst=20 nodelay;
        limit_req_status 429;
        proxy_pass http://localhost:3000;
    }

    # Stricter limit on auth endpoints
    location /api/auth/ {
        limit_req zone=login burst=3 nodelay;
        limit_req_status 429;
        proxy_pass http://localhost:3000;
    }
}
```

---

## Common Patterns & Best Practices

### Reusable Proxy Headers Snippet

`/etc/nginx/snippets/proxy-headers.conf`:
```nginx
proxy_set_header Host              $host;
proxy_set_header X-Real-IP         $remote_addr;
proxy_set_header X-Forwarded-For   $proxy_add_x_forwarded_for;
proxy_set_header X-Forwarded-Proto $scheme;
proxy_set_header X-Forwarded-Host  $host;
proxy_http_version 1.1;
proxy_set_header Connection "";
```

Use in server blocks:
```nginx
location /api/ {
    include snippets/proxy-headers.conf;
    proxy_pass http://localhost:3000;
}
```

### Nginx in Docker with Traefik Alternative

For Docker deployments, Nginx can run as a container:

```yaml
# compose.yml
services:
  nginx:
    image: nginx:1.25-alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx/conf.d:/etc/nginx/conf.d:ro
      - ./nginx/ssl:/etc/nginx/ssl:ro
      - letsencrypt:/etc/letsencrypt:ro
    depends_on:
      - api
    networks:
      - frontend

  api:
    image: myapi:latest
    expose:
      - "3000"   # only exposed to Docker network, not host
    networks:
      - frontend
      - backend

networks:
  frontend:
  backend:
    internal: true
```

`nginx/conf.d/myapp.conf`:
```nginx
server {
    listen 80;
    server_name myapp.com;
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl http2;
    server_name myapp.com;

    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;

    location / {
        proxy_pass http://api:3000;   # Docker service name resolution
        include /etc/nginx/snippets/proxy-headers.conf;
    }
}
```

### Gzip Compression

```nginx
# In nginx.conf http block
gzip on;
gzip_vary on;
gzip_min_length 1024;
gzip_proxied expired no-cache no-store private auth;
gzip_types
    text/plain
    text/css
    text/javascript
    application/javascript
    application/json
    application/x-javascript
    image/svg+xml;
gzip_comp_level 6;
```

### Client Body Size Limit

```nginx
# Default is 1m — increase for file uploads
client_max_body_size 50m;
```

---

## Anti-Patterns to Avoid

### Proxying to `0.0.0.0`

```nginx
# BAD — connects to all interfaces, ambiguous
proxy_pass http://0.0.0.0:3000;

# GOOD — explicit
proxy_pass http://127.0.0.1:3000;
```

### Not Forwarding Real IP

Without `X-Real-IP` and `X-Forwarded-For`, your application logs will show Nginx's IP (`127.0.0.1`) instead of the real client IP, breaking analytics, rate limiting, and security logging.

Your Node.js/Fastify app must also be configured to trust the proxy:
```typescript
// Fastify
const app = Fastify({
  trustProxy: true  // reads X-Forwarded-For
});
```

### Leaving Default Nginx Config Active

```bash
# Remove default site that reveals Nginx version/config
sudo rm /etc/nginx/sites-enabled/default
```

### No Timeouts Set

Without timeouts, a slow upstream can hold Nginx connections open indefinitely, eventually exhausting worker connections:

```nginx
proxy_connect_timeout  10s;
proxy_send_timeout     60s;
proxy_read_timeout     60s;
```

### Exposing Backend Port on Host

```yaml
services:
  api:
    # BAD — exposes port 3000 on host, bypassing nginx
    ports:
      - "3000:3000"

    # GOOD — only nginx can reach it
    expose:
      - "3000"
```

---

## Debugging & Troubleshooting

### Test Configuration

```bash
sudo nginx -t           # test syntax
sudo nginx -T           # test + dump final config
sudo nginx -T | grep -A5 "server_name myapp"
```

### Reload vs Restart

```bash
sudo systemctl reload nginx    # graceful reload — no downtime (use this)
sudo systemctl restart nginx   # kills all connections — avoid in production
```

### Check Logs

```bash
# Access log (every request)
sudo tail -f /var/log/nginx/access.log

# Error log (problems)
sudo tail -f /var/log/nginx/error.log

# Filter for errors
sudo grep -i "error\|warn" /var/log/nginx/error.log | tail -50

# Check if upstream is reachable
curl -v http://localhost:3000/health
```

### Common Errors

**502 Bad Gateway** — Nginx can't reach the upstream:
```bash
# Is the app running?
systemctl status myapp
# Or
docker compose ps

# Can Nginx reach it?
curl http://localhost:3000/health
```

**413 Request Entity Too Large** — increase body size:
```nginx
client_max_body_size 50m;
```

**504 Gateway Timeout** — upstream took too long:
```nginx
proxy_read_timeout 120s;  # increase for slow operations
```

**SSL/TLS handshake errors** — check certificate:
```bash
sudo certbot certificates
openssl s_client -connect myapp.com:443 -servername myapp.com
```

### Performance Analysis

```bash
# Check active connections
nginx -s status   # if status module enabled
# Or via stub_status:
curl http://localhost/nginx-status

# Check worker processes
ps aux | grep nginx

# Watch access log for slow responses
awk '{print $NF}' /var/log/nginx/access.log | sort -n | tail -20
```

---

## Real-World Scenarios

### Scenario 1: Multi-App Server (Multiple Domains)

```nginx
# App 1: myapp.com → port 3000
server {
    listen 443 ssl http2;
    server_name myapp.com;
    ssl_certificate /etc/letsencrypt/live/myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/myapp.com/privkey.pem;
    location / { proxy_pass http://localhost:3000; include snippets/proxy-headers.conf; }
}

# App 2: api.myapp.com → port 4000
server {
    listen 443 ssl http2;
    server_name api.myapp.com;
    ssl_certificate /etc/letsencrypt/live/api.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/api.myapp.com/privkey.pem;
    location / { proxy_pass http://localhost:4000; include snippets/proxy-headers.conf; }
}

# App 3: blog.myapp.com → port 8080
server {
    listen 443 ssl http2;
    server_name blog.myapp.com;
    ssl_certificate /etc/letsencrypt/live/blog.myapp.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/blog.myapp.com/privkey.pem;
    location / { proxy_pass http://localhost:8080; include snippets/proxy-headers.conf; }
}

# Redirect all HTTP to HTTPS
server {
    listen 80;
    server_name _;
    return 301 https://$host$request_uri;
}
```

### Scenario 2: Maintenance Page

```nginx
# Check if maintenance file exists, serve it instead of proxying
location / {
    if (-f /var/www/maintenance.html) {
        return 503;
    }
    proxy_pass http://localhost:3000;
}

error_page 503 /maintenance.html;
location = /maintenance.html {
    root /var/www;
    internal;
}
```

```bash
# Enable maintenance mode
touch /var/www/maintenance.html
# ... do your deployment ...
rm /var/www/maintenance.html
```

### Scenario 3: Basic Auth for Staging

```bash
# Create password file
sudo apt install -y apache2-utils
sudo htpasswd -c /etc/nginx/.htpasswd staging_user
```

```nginx
server {
    listen 443 ssl http2;
    server_name staging.myapp.com;

    auth_basic "Staging Environment";
    auth_basic_user_file /etc/nginx/.htpasswd;

    location / {
        proxy_pass http://localhost:3001;
        include snippets/proxy-headers.conf;
    }
}
```

---

## Further Reading

- [Nginx Documentation](https://nginx.org/en/docs/)
- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/) — generates hardened Nginx TLS config
- [Certbot Documentation](https://certbot.eff.org/docs/)
- [Nginx Performance Tuning](https://www.nginx.com/blog/tuning-nginx/)

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| Reverse proxy | Nginx sits in front of your app — SSL, routing, headers |
| `proxy_pass` | Forward requests to your upstream application |
| `proxy_set_header` | Always forward real client IP and protocol |
| HTTPS | Use Certbot + Let's Encrypt for free, auto-renewing certificates |
| Security headers | HSTS, X-Frame-Options, X-Content-Type-Options |
| Rate limiting | `limit_req_zone` protects your app from abuse |
| Load balancing | `upstream` block with `least_conn` for even distribution |
| WebSockets | Requires `Upgrade` header forwarding and long timeouts |
| Gzip | Enable for JSON, HTML, JS, CSS — significant bandwidth savings |
| `nginx -t` | Always test config before reload |

Nginx is the load-bearing wall of web infrastructure. Learn its config syntax, understand the proxy headers contract with your application, and automate certificate renewal from day one.
