# HTTPS & TLS

## Overview

HTTPS is HTTP over TLS (Transport Layer Security). It provides three guarantees: confidentiality (data is encrypted in transit), integrity (data cannot be tampered with undetected), and authentication (you are talking to the server you think you are). In 2025, serving anything over plain HTTP is unacceptable — browsers mark it as insecure, search engines penalize it, and users rightly distrust it. This chapter explains how TLS works, how to configure it correctly, and how to enforce it.

---

## Prerequisites

- Basic understanding of HTTP (requests, responses, headers)
- Familiarity with DNS and server configuration
- Node.js and Fastify basics

---

## Core Concepts

### The TLS handshake

When a browser connects to `https://example.com`:

1. **ClientHello** — client sends supported TLS version and cipher suites
2. **ServerHello** — server picks the best cipher suite and sends its certificate
3. **Certificate verification** — client checks the certificate is signed by a trusted CA and the domain matches
4. **Key exchange** — client and server establish a shared secret (ECDHE is common)
5. **Session established** — all subsequent data is encrypted with the shared secret

The entire handshake takes ~1 RTT with TLS 1.3 (the current standard) vs. ~2 RTT with TLS 1.2.

### Certificate types

| Type | Validation | Use case |
|------|------------|----------|
| DV (Domain Validation) | Proves you control the domain | Most websites — Let's Encrypt provides these free |
| OV (Organization Validation) | Proves your organization is real | Business websites |
| EV (Extended Validation) | Full identity verification | Banks, financial services |

### TLS versions

| Version | Status |
|---------|--------|
| SSL 3.0, TLS 1.0, TLS 1.1 | Deprecated — do not use |
| TLS 1.2 | Acceptable but not preferred |
| TLS 1.3 | Current standard — use this |

---

## Hands-On Examples

### Let's Encrypt with Certbot (Linux server)

```bash
# Install certbot
sudo apt install certbot python3-certbot-nginx

# Obtain a certificate for your domain
sudo certbot --nginx -d example.com -d www.example.com

# Certbot auto-renews via a cron job — verify it
sudo certbot renew --dry-run

# Certificate files are at:
# /etc/letsencrypt/live/example.com/fullchain.pem
# /etc/letsencrypt/live/example.com/privkey.pem
```

### Nginx TLS configuration (reverse proxy)

In production, terminate TLS at a reverse proxy (Nginx, Traefik, or a load balancer) rather than in Node.js. The proxy handles TLS; Node.js handles HTTP on an internal port.

```nginx
# /etc/nginx/sites-available/example.com
server {
    listen 80;
    server_name example.com www.example.com;
    # Redirect all HTTP to HTTPS
    return 301 https://$host$request_uri;
}

server {
    listen 443 ssl;
    server_name example.com www.example.com;

    ssl_certificate /etc/letsencrypt/live/example.com/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/example.com/privkey.pem;

    # TLS hardening
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256:ECDHE-ECDSA-AES256-GCM-SHA384:ECDHE-RSA-AES256-GCM-SHA384;
    ssl_prefer_server_ciphers off;

    # HSTS — tell browsers to always use HTTPS for this domain
    add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;

    # OCSP stapling — speeds up certificate validation
    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /etc/letsencrypt/live/example.com/chain.pem;

    location / {
        proxy_pass http://localhost:3000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

### Traefik with automatic Let's Encrypt (Docker)

```yaml
# docker-compose.yml
services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=false"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.myresolver.acme.httpchallenge=true"
      - "--certificatesresolvers.myresolver.acme.httpchallenge.entrypoint=web"
      - "--certificatesresolvers.myresolver.acme.email=you@example.com"
      - "--certificatesresolvers.myresolver.acme.storage=/letsencrypt/acme.json"
      # Global HTTP to HTTPS redirect
      - "--entrypoints.web.http.redirections.entryPoint.to=websecure"
      - "--entrypoints.web.http.redirections.entryPoint.scheme=https"
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - "/var/run/docker.sock:/var/run/docker.sock:ro"
      - "letsencrypt:/letsencrypt"

  api:
    image: myapp:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.api.rule=Host(`api.example.com`)"
      - "traefik.http.routers.api.entrypoints=websecure"
      - "traefik.http.routers.api.tls.certresolver=myresolver"
      - "traefik.http.services.api.loadbalancer.server.port=3000"

volumes:
  letsencrypt:
```

### HSTS in Node.js with Helmet

When you control the HTTP layer (e.g., running Fastify directly without a proxy in development):

```typescript
import helmet from '@fastify/helmet';

await fastify.register(helmet, {
  hsts: {
    maxAge: 63072000,        // 2 years in seconds
    includeSubDomains: true,
    preload: true,           // submit to browser preload lists
  },
});
```

### Detecting insecure requests

When behind a TLS-terminating proxy, check `X-Forwarded-Proto` to redirect HTTP to HTTPS:

```typescript
fastify.addHook('onRequest', async (request, reply) => {
  if (
    process.env.NODE_ENV === 'production' &&
    request.headers['x-forwarded-proto'] === 'http'
  ) {
    const httpsUrl = `https://${request.hostname}${request.url}`;
    return reply.redirect(301, httpsUrl);
  }
});
```

### Certificate pinning (mobile/API clients)

Certificate pinning ties your client to a specific certificate or public key, preventing MITM attacks even with a valid certificate signed by a trusted CA.

```typescript
// Node.js HTTPS client with pinning
import https from 'https';
import crypto from 'crypto';

const EXPECTED_PIN = 'sha256/BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB='; // base64-encoded SHA-256 of SubjectPublicKeyInfo

function createPinnedAgent(): https.Agent {
  return new https.Agent({
    checkServerIdentity: (host, cert) => {
      const pubKeyDer = cert.pubkey;
      const hash = crypto
        .createHash('sha256')
        .update(pubKeyDer)
        .digest('base64');

      const pin = `sha256/${hash}`;
      if (pin !== EXPECTED_PIN) {
        throw new Error(`Certificate pin mismatch: got ${pin}`);
      }
      return undefined; // no error
    },
  });
}

const agent = createPinnedAgent();
const response = await fetch('https://api.example.com/data', { agent } as RequestInit);
```

---

## Common Patterns & Best Practices

- **Redirect HTTP to HTTPS** with a 301 permanent redirect — never serve content over plain HTTP
- **HSTS with preload** — once your site is stable on HTTPS, submit to browser preload lists
- **TLS 1.3 only** — disable TLS 1.0 and 1.1 at the reverse proxy level
- **Use ECDHE cipher suites** — provide perfect forward secrecy so past sessions cannot be decrypted if the key is later compromised
- **Enable OCSP stapling** — speeds up certificate validation and improves privacy
- **Automate certificate renewal** — Let's Encrypt certificates expire every 90 days; Certbot and Traefik handle renewal automatically
- **Monitor certificate expiry** — set up alerts at 30 days before expiry as a safety net

---

## Anti-Patterns to Avoid

- Serving content on port 80 without redirecting to HTTPS
- Using self-signed certificates in production (use Let's Encrypt — it's free)
- Ignoring certificate expiry until users report connection errors
- Setting HSTS with a very short `maxAge` and `preload: true` — once preloaded, removing HSTS locks users out
- Disabling certificate verification in development with `NODE_TLS_REJECT_UNAUTHORIZED=0` and forgetting to remove it

---

## Debugging & Troubleshooting

**"Certificate expired" errors**
Check expiry: `openssl s_client -connect example.com:443 | openssl x509 -noout -dates`
Ensure the renewal cron job is running: `sudo certbot renew --dry-run`

**"SSL handshake error" in Node.js**
Usually caused by mismatched TLS versions. Check that the server supports TLS 1.2+. If connecting to a legacy server: `NODE_OPTIONS=--tls-min-v1.0` (temporary workaround only).

**"HSTS preload check failed"**
Requirements for preload: `max-age` must be at least 31536000 (1 year), `includeSubDomains` must be set, `preload` must be set, and the domain must redirect HTTP to HTTPS. Check at [hstspreload.org](https://hstspreload.org).

---

## Real-World Scenarios

**Scenario: Zero-downtime certificate rotation**

Certbot and Traefik handle this automatically. For manual setups:
1. Obtain the new certificate before the old one expires
2. Update the Nginx/proxy configuration to point to the new cert files
3. Reload Nginx (`nginx -s reload`) — no downtime, existing connections continue

**Scenario: Wildcard certificate for subdomains**

```bash
# Use DNS challenge for wildcard certs (requires DNS provider API access)
sudo certbot certonly \
  --dns-cloudflare \
  --dns-cloudflare-credentials ~/.cloudflare.ini \
  -d "*.example.com" \
  -d "example.com"
```

---

## Further Reading

- [Mozilla SSL Configuration Generator](https://ssl-config.mozilla.org/)
- [Let's Encrypt documentation](https://letsencrypt.org/docs/)
- [TLS 1.3 explained — Cloudflare blog](https://blog.cloudflare.com/rfc-8446-aka-tls-1-3/)
- [HSTS Preload list](https://hstspreload.org/)
- Track 07: [Security Headers](security-headers.md)

---

## Summary

HTTPS is non-negotiable in 2025. The practical setup is simple: terminate TLS at a reverse proxy (Nginx or Traefik), use Let's Encrypt for free certificates with automatic renewal, redirect all HTTP to HTTPS with a 301, and set HSTS headers to lock in HTTPS. TLS 1.3 is the current standard — disable 1.0 and 1.1 at the proxy level. The hardest part of HTTPS is not the initial setup but the operational discipline: monitoring certificate expiry, automating renewal, and keeping proxy configurations up to date as cipher suite recommendations evolve.
