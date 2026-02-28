# Security Headers

## Overview

HTTP security headers are the fastest way to harden a web application. They instruct the browser on how to behave when rendering your content — preventing clickjacking, disabling dangerous features, enforcing HTTPS, and restricting what resources can load. A single `npm install @fastify/helmet` and five minutes of configuration addresses an entire class of vulnerabilities. This chapter covers the headers that matter, what they do, and how to configure them correctly.

---

## Prerequisites

- Basic HTTP headers knowledge
- Fastify and Node.js
- Familiarity with Content Security Policy concepts

---

## Core Concepts

Security headers fall into three categories:

1. **Transport security** — enforce HTTPS (`Strict-Transport-Security`)
2. **Content security** — control what resources can load and execute (`Content-Security-Policy`, `X-Content-Type-Options`)
3. **Frame and access control** — prevent clickjacking and control embedded content (`X-Frame-Options`, `Permissions-Policy`)

---

## Hands-On Examples

### Helmet.js — the one-liner baseline

```typescript
import Fastify from 'fastify';
import helmet from '@fastify/helmet';

const fastify = Fastify();

await fastify.register(helmet);
// Adds 11 security headers with sensible defaults
```

Helmet sets these headers by default:
- `Content-Security-Policy`
- `Cross-Origin-Embedder-Policy`
- `Cross-Origin-Opener-Policy`
- `Cross-Origin-Resource-Policy`
- `Origin-Agent-Cluster`
- `Referrer-Policy`
- `Strict-Transport-Security`
- `X-Content-Type-Options`
- `X-DNS-Prefetch-Control`
- `X-Download-Options`
- `X-Frame-Options`
- `X-Permitted-Cross-Domain-Policies`
- `X-XSS-Protection` (disabled by default — modern browsers do not need it)

---

### Strict-Transport-Security (HSTS)

Tells browsers to only access this site over HTTPS, even if the user types `http://`.

```typescript
await fastify.register(helmet, {
  hsts: {
    maxAge: 63072000,        // 2 years — required for HSTS preload list
    includeSubDomains: true, // applies to all subdomains
    preload: true,           // submit to browser preload lists
  },
});
```

After setting HSTS with a long `maxAge`, it is difficult to remove. Do not set `preload: true` until you are confident the entire domain (and all subdomains) serve HTTPS.

---

### Content-Security-Policy (CSP)

The most powerful and most complex header. Restricts which resources the browser loads and executes.

```typescript
await fastify.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      // Default for all resource types not explicitly listed
      defaultSrc: ["'self'"],

      // Scripts: only from own origin and a trusted CDN
      scriptSrc: ["'self'", 'https://cdn.jsdelivr.net'],

      // Styles: own origin and Google Fonts
      styleSrc: ["'self'", 'https://fonts.googleapis.com', "'unsafe-inline'"],
      // 'unsafe-inline' for styles is less risky than for scripts; ideally replace with nonces

      // Fonts
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],

      // Images: own origin, data URIs, and CDN
      imgSrc: ["'self'", 'data:', 'https://cdn.example.com'],

      // XHR and fetch
      connectSrc: ["'self'", 'https://api.example.com'],

      // No plugins, no object/embed tags
      objectSrc: ["'none'"],

      // No inline frames from other origins
      frameSrc: ["'none'"],

      // Upgrade plain HTTP resource references to HTTPS
      upgradeInsecureRequests: [],
    },
  },
});
```

**Using nonces for inline scripts (required for Next.js, many SPAs):**

```typescript
import crypto from 'crypto';

// Generate nonce per request
fastify.addHook('onRequest', async (request) => {
  request.cspNonce = crypto.randomBytes(16).toString('base64');
});

await fastify.register(helmet, {
  contentSecurityPolicy: {
    useDefaults: true,
    directives: {
      scriptSrc: [
        "'self'",
        (req) => `'nonce-${(req as any).cspNonce}'`,
      ],
    },
  },
});

// In your template engine:
// <script nonce="{{cspNonce}}">...</script>
```

**Report-only mode (for gradual rollout):**

```typescript
// Does not block — only sends violation reports
reply.header(
  'Content-Security-Policy-Report-Only',
  "default-src 'self'; report-uri /api/csp-report"
);

// Endpoint to receive violation reports
fastify.post('/api/csp-report', async (request) => {
  fastify.log.warn({ cspReport: request.body }, 'CSP violation');
  return { ok: true };
});
```

---

### X-Frame-Options

Prevents your site from being embedded in an iframe — the root of clickjacking attacks.

```typescript
await fastify.register(helmet, {
  frameguard: {
    action: 'deny', // never allow framing — use 'sameorigin' if you embed your own pages
  },
});
```

CSP's `frame-ancestors` directive supersedes `X-Frame-Options` in modern browsers. Use both for compatibility:

```typescript
contentSecurityPolicy: {
  directives: {
    frameAncestors: ["'none'"], // modern browsers
    // X-Frame-Options: DENY set by frameguard for older browsers
  },
},
```

---

### X-Content-Type-Options

Prevents MIME type sniffing — the browser must use the declared `Content-Type`, not guess from content:

```typescript
// Helmet sets this by default: X-Content-Type-Options: nosniff
await fastify.register(helmet, {
  noSniff: true,
});
```

Without this, a `.jpg` file containing JavaScript could be executed as a script.

---

### Permissions-Policy

Controls access to browser features (camera, microphone, geolocation, etc.):

```typescript
// Helmet includes a default Permissions-Policy
// To customize:
fastify.addHook('onSend', async (request, reply) => {
  reply.header(
    'Permissions-Policy',
    [
      'camera=()',           // disable camera access
      'microphone=()',       // disable microphone access
      'geolocation=(self)',  // allow geolocation only from own origin
      'payment=()',          // disable payment API
      'usb=()',              // disable USB access
    ].join(', ')
  );
});
```

---

### Referrer-Policy

Controls how much referrer information is included when the user navigates away:

```typescript
await fastify.register(helmet, {
  referrerPolicy: {
    policy: 'strict-origin-when-cross-origin',
    // Full URL for same-origin, only origin for cross-origin HTTPS, nothing for HTTP
  },
});
```

| Policy | What is sent |
|--------|-------------|
| `no-referrer` | Nothing — most private |
| `strict-origin-when-cross-origin` | Full URL same-origin, origin only cross-origin — good default |
| `unsafe-url` | Full URL always — leaks sensitive query params to third parties |

---

### Cross-Origin-Resource-Policy (CORP)

Controls which origins can load your resources:

```typescript
await fastify.register(helmet, {
  crossOriginResourcePolicy: {
    policy: 'same-origin', // only your own origin can load these resources
    // 'cross-origin' for CDN files meant to be loaded anywhere
    // 'same-site' for resources shared between subdomains
  },
});
```

---

### Complete production configuration

```typescript
await fastify.register(helmet, {
  // HSTS
  hsts: {
    maxAge: 63072000,
    includeSubDomains: true,
    preload: true,
  },

  // No iframing
  frameguard: { action: 'deny' },

  // MIME sniffing protection
  noSniff: true,

  // Referrer policy
  referrerPolicy: { policy: 'strict-origin-when-cross-origin' },

  // Cross-origin resource policy
  crossOriginResourcePolicy: { policy: 'same-origin' },

  // CSP
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'"],
      styleSrc: ["'self'"],
      imgSrc: ["'self'", 'data:'],
      connectSrc: ["'self'"],
      objectSrc: ["'none'"],
      frameAncestors: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
});
```

---

## Common Patterns & Best Practices

- **Register Helmet before routes** — hooks run in registration order
- **Start with CSP in report-only mode** — audit violations before enforcing
- **Test headers with securityheaders.com** — gives your site a grade and flags missing headers
- **Different CSP for different route groups** — stricter for the admin panel, looser for the public site
- **Use nonces, not `unsafe-inline`** — `unsafe-inline` defeats CSP for scripts
- **Set `Cache-Control` on sensitive endpoints** — `no-store` for auth responses and API data

---

## Anti-Patterns to Avoid

- Setting `X-XSS-Protection: 1; mode=block` — deprecated, can introduce XSS in some browsers
- Using `'unsafe-eval'` in CSP — allows `eval()` and `Function()` which XSS relies on
- Setting HSTS with `preload: true` on a subdomain that might not always serve HTTPS
- Skipping security headers in development — bugs often emerge from differences between envs
- Wide-open CSP (`default-src *`) — negates the purpose of the header

---

## Debugging & Troubleshooting

**"My React app is broken after adding Helmet"**
CSP is blocking inline scripts. Next.js and create-react-app inject inline scripts. Use nonces (server-rendered apps) or hash-based CSP values, or temporarily allow `'unsafe-inline'` while you audit.

**"How do I check what headers I'm setting?"**
`curl -I https://your-site.com` or open DevTools Network tab, click the main request, and look at Response Headers.

**"CSP is blocking Google Analytics / Intercom / other third-party script"**
Add the third-party domain to `scriptSrc` and its API endpoints to `connectSrc`. Check the third-party's documentation for their required CSP configuration.

---

## Real-World Scenarios

**Scenario: API-only backend (no HTML rendering)**

For a pure REST API, a subset of headers applies:

```typescript
await fastify.register(helmet, {
  // API does not serve HTML — CSP is not needed for JSON responses
  contentSecurityPolicy: false,
  // These still apply:
  hsts: { maxAge: 63072000, includeSubDomains: true },
  noSniff: true,
  referrerPolicy: { policy: 'no-referrer' },
  frameguard: { action: 'deny' },
});
```

---

## Further Reading

- [MDN: HTTP Security Headers](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers#security)
- [securityheaders.com](https://securityheaders.com/) — scan your site
- [CSP Evaluator](https://csp-evaluator.withgoogle.com/) — Google tool to check CSP strength
- [Helmet.js documentation](https://helmetjs.github.io/)
- Track 07: [CORS & CSRF](cors-csrf.md)
- Track 07: [HTTPS & TLS](https-tls.md)

---

## Summary

Security headers are the highest-leverage security improvement you can make in a single afternoon. Helmet.js sets a strong default baseline with one line. The headers that deserve the most attention are: HSTS (enforce HTTPS), Content-Security-Policy (restrict script execution), X-Frame-Options/frame-ancestors (prevent clickjacking), X-Content-Type-Options (prevent MIME sniffing), and Referrer-Policy (protect URL privacy). Roll out CSP incrementally using report-only mode to catch violations before enforcing. Verify your final configuration with securityheaders.com.
