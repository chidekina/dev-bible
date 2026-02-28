# CORS & CSRF

## Overview

CORS (Cross-Origin Resource Sharing) and CSRF (Cross-Site Request Forgery) are two distinct but related web security mechanisms, both rooted in the browser's same-origin policy. Misunderstanding or misconfiguring either one leads to real vulnerabilities. This chapter explains how they work, why they exist, and how to implement them correctly in a Fastify/Node.js API.

---

## Prerequisites

- HTTP fundamentals (headers, methods, cookies)
- Basic understanding of browser security model
- Familiarity with Fastify

---

## Core Concepts

### Same-Origin Policy

The browser enforces that a page from `https://app.example.com` cannot read responses from `https://api.other.com` unless the other server explicitly permits it. "Origin" is defined as the combination of protocol + hostname + port. `http://example.com` and `https://example.com` are different origins.

### CORS — letting trusted origins cross the boundary

CORS is a mechanism for servers to relax the same-origin policy for specific origins. It uses HTTP headers to tell the browser which cross-origin requests are allowed.

### CSRF — riding the user's authenticated session

CSRF attacks trick a user's browser into making authenticated requests to a server the user is logged into, by embedding the request in a page the attacker controls. CORS does not protect against CSRF — browsers will still *send* cookies cross-origin even for restricted requests.

---

## Hands-On Examples

### CORS configuration in Fastify

```typescript
import cors from '@fastify/cors';

await fastify.register(cors, {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.example.com',
      'https://www.example.com',
    ];

    // Allow requests with no origin (server-to-server, curl)
    if (!origin) return callback(null, true);

    if (allowedOrigins.includes(origin)) {
      return callback(null, true);
    }

    // Also allow localhost in development
    if (process.env.NODE_ENV !== 'production' && origin.startsWith('http://localhost')) {
      return callback(null, true);
    }

    return callback(new Error('Not allowed by CORS'), false);
  },
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,       // allow cookies/Authorization headers
  maxAge: 86400,           // cache preflight response for 24 hours
});
```

### Understanding preflight requests

When a browser makes a cross-origin request that is not "simple" (uses custom headers, non-GET/POST methods, or a content type other than `application/x-www-form-urlencoded`/`multipart/form-data`/`text/plain`), it first sends an `OPTIONS` preflight:

```
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

The server responds:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

If the server does not respond correctly, the browser blocks the actual request.

### CSRF attack demonstration

Imagine a user is logged in to `https://bank.example.com` with a session cookie. An attacker creates a malicious page:

```html
<!-- attacker.com/steal.html -->
<form action="https://bank.example.com/transfer" method="POST">
  <input type="hidden" name="amount" value="1000" />
  <input type="hidden" name="to" value="attacker-account" />
</form>
<script>document.forms[0].submit();</script>
```

When the victim visits this page, the browser submits the form — and includes the session cookie automatically. The bank receives an authenticated request.

### CSRF defense 1: SameSite cookies

```typescript
fastify.post('/auth/login', async (request, reply) => {
  // ... validate credentials ...

  reply.setCookie('session', sessionToken, {
    httpOnly: true,     // not accessible to JavaScript
    secure: true,       // HTTPS only
    sameSite: 'lax',   // blocks cross-site POST, allows same-site and top-level navigation
    path: '/',
    maxAge: 3600,
  });

  return { ok: true };
});
```

`SameSite` values:
- `strict` — cookie never sent on cross-site requests (breaks OAuth flows and shared links)
- `lax` — cookie not sent on cross-site form POSTs or AJAX, but sent on top-level navigation GET (safe and compatible)
- `none` — cookie always sent cross-site (requires `secure: true`; needed for embedded iframes)

For most APIs, `SameSite: lax` provides CSRF protection without breaking usability. APIs that use `Authorization: Bearer` headers (JWTs) are immune to CSRF because browsers do not automatically add that header to cross-site requests.

### CSRF defense 2: CSRF tokens (for session-cookie-based apps)

If your app uses session cookies and cannot use `SameSite: strict/lax` (e.g., embedded in an iframe from a different origin), implement CSRF tokens:

```typescript
import crypto from 'crypto';

// Generate a CSRF token tied to the session
function generateCsrfToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

// Store in session on login
fastify.post('/auth/login', async (request, reply) => {
  // ... validate ...
  const csrfToken = generateCsrfToken();
  await db.session.create({ data: { token: sessionToken, csrfToken, userId } });

  reply.setCookie('session', sessionToken, { httpOnly: true, secure: true, sameSite: 'none' });
  // Return CSRF token in response body — JS can read it, but cross-site JS cannot
  return { csrfToken };
});

// Validate CSRF token on state-changing requests
fastify.addHook('preHandler', async (request, reply) => {
  const unsafeMethods = ['POST', 'PUT', 'PATCH', 'DELETE'];
  if (!unsafeMethods.includes(request.method)) return;

  const sessionCookie = request.cookies.session;
  if (!sessionCookie) return reply.status(401).send({ error: 'Not authenticated' });

  const session = await db.session.findUnique({ where: { token: sessionCookie } });
  if (!session) return reply.status(401).send({ error: 'Invalid session' });

  const csrfHeader = request.headers['x-csrf-token'];
  if (!csrfHeader || csrfHeader !== session.csrfToken) {
    return reply.status(403).send({ error: 'CSRF token mismatch' });
  }
});
```

Frontend usage:

```typescript
// Store CSRF token from login response
const { csrfToken } = await login(email, password);
localStorage.setItem('csrf_token', csrfToken);

// Include on every mutating request
async function apiPost(url: string, body: unknown) {
  return fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-Token': localStorage.getItem('csrf_token') ?? '',
    },
    body: JSON.stringify(body),
    credentials: 'include', // send cookies
  });
}
```

### The "wildcard origin + credentials" mistake

```typescript
// NEVER do this — it defeats CORS entirely
await fastify.register(cors, {
  origin: '*',
  credentials: true, // browsers block this combination — it's also a CORS spec violation
});
```

Browsers will refuse to send credentials when `Access-Control-Allow-Origin: *`. The spec prohibits it. The correct approach is to echo back the specific trusted origin.

---

## Common Patterns & Best Practices

- **For JWT + Authorization header APIs**: no CSRF defense needed — browsers do not automatically add `Authorization` headers to cross-site requests
- **For session cookie APIs**: use `SameSite: lax` or `SameSite: strict` cookies; this eliminates most CSRF risks
- **Allowlist origins explicitly** — never use regex patterns that can be bypassed (`example.com.evil.com` matches `.*example.com.*`)
- **Log CORS rejections** — they often indicate configuration mistakes or probing attacks
- **Include `OPTIONS` in allowed methods** — preflight requests must succeed

---

## Anti-Patterns to Avoid

- `origin: '*'` on an API that uses cookies
- Regex-based origin validation that is too broad
- Checking `Referer` header for CSRF protection — it can be absent or spoofed
- Storing CSRF tokens in cookies (then they are subject to the same CSRF vulnerability)
- Disabling CORS in development (`origin: true` for all) and forgetting to restrict it in production
- Trusting `X-Requested-With: XMLHttpRequest` as a CSRF defense — modern fetch doesn't require it and it can be set by attackers

---

## Debugging & Troubleshooting

**"CORS error: No 'Access-Control-Allow-Origin' header"**
The server is not responding to the preflight OPTIONS request correctly. Check that your CORS plugin is registered before route handlers and that the `origin` function returns `true` for the requesting origin.

**"CORS error with credentials"**
When `credentials: true` (client-side `credentials: 'include'`), the server must respond with a specific origin (not `*`) and `Access-Control-Allow-Credentials: true`.

**"Forms work but fetch doesn't"**
Likely a preflight failure. Check the `Access-Control-Allow-Headers` — the request may be sending a custom header (like `Authorization`) that is not listed.

---

## Further Reading

- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: SameSite cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- Track 07: [Security Headers](security-headers.md)
- Track 07: [Authentication & Authorization](authentication-authorization.md)

---

## Summary

CORS and CSRF are different solutions to different problems that share the same root cause: the browser's same-origin policy. CORS is a server-side opt-in to allow specific cross-origin reads. CSRF is a class of attack where an attacker causes the victim's browser to make authenticated cross-origin writes. For modern APIs using `Authorization: Bearer` (JWT), CSRF is not a concern — browsers do not add that header automatically. For session-cookie-based apps, `SameSite: lax` is the first line of CSRF defense, with CSRF tokens as a backup for cases where SameSite is not sufficient. CORS must never use `origin: '*'` with `credentials: true` — always allowlist specific trusted origins.
