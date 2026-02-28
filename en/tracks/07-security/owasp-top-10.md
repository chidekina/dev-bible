# OWASP Top 10

## Overview

The OWASP Top 10 is a standard awareness document published by the Open Web Application Security Project. It represents the ten most critical security risks to web applications, compiled from real-world vulnerability data across thousands of organizations. Understanding these risks — and knowing how to mitigate them in a Node.js/TypeScript stack — is the baseline for building secure software.

This chapter covers each category with a concrete example of how it manifests in a typical Fastify/Express application and how to eliminate it.

---

## Prerequisites

- HTTP fundamentals (requests, responses, headers, cookies)
- Basic Node.js and TypeScript
- Familiarity with SQL databases and REST APIs

---

## Core Concepts

The 2021 OWASP Top 10 categories are:

| # | Category | Common Root Cause |
|---|----------|-------------------|
| A01 | Broken Access Control | Missing authorization checks |
| A02 | Cryptographic Failures | Weak encryption, plaintext storage |
| A03 | Injection | Unsanitized user input in queries/commands |
| A04 | Insecure Design | No threat modeling, missing security requirements |
| A05 | Security Misconfiguration | Default configs, verbose errors, open ports |
| A06 | Vulnerable and Outdated Components | Unpatched dependencies |
| A07 | Identification and Authentication Failures | Weak passwords, no MFA, broken session management |
| A08 | Software and Data Integrity Failures | Unverified updates, deserialization |
| A09 | Security Logging and Monitoring Failures | No audit logs, undetected breaches |
| A10 | Server-Side Request Forgery (SSRF) | Accepting user-controlled URLs for server requests |

---

## Hands-On Examples

### A01 — Broken Access Control

The most common category. Happens when users can access resources or perform actions they should not be allowed to.

**Vulnerable pattern:**

```typescript
// GET /api/invoices/:id — no ownership check
fastify.get('/api/invoices/:id', async (request, reply) => {
  const { id } = request.params as { id: string };
  // Any authenticated user can read any invoice by guessing the ID
  const invoice = await db.invoice.findUnique({ where: { id } });
  return invoice;
});
```

**Fixed pattern — always enforce ownership:**

```typescript
fastify.get('/api/invoices/:id', { preHandler: [requireAuth] }, async (request, reply) => {
  const { id } = request.params as { id: string };
  const userId = request.user.id;

  const invoice = await db.invoice.findUnique({
    where: { id, userId }, // scoped to the authenticated user
  });

  if (!invoice) {
    return reply.status(404).send({ error: 'Invoice not found' });
  }

  return invoice;
});
```

Rule: never trust the client to tell you what they own. Always scope queries by the authenticated identity.

---

### A02 — Cryptographic Failures

Storing sensitive data in plaintext or using weak hashing.

**Vulnerable — MD5 for passwords:**

```typescript
import crypto from 'crypto';

// NEVER do this — MD5 is broken and has no salt
const hash = crypto.createHash('md5').update(password).digest('hex');
```

**Fixed — bcrypt with a work factor:**

```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // higher = slower = more resistant to brute force

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

Also encrypt data at rest when it is sensitive (PII, financial data) and enforce HTTPS to protect data in transit.

---

### A03 — Injection

The classic SQL injection, but also NoSQL injection, command injection, and LDAP injection.

**Vulnerable — string interpolation in a SQL query:**

```typescript
// If userId = "1 OR 1=1", this dumps all users
const result = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
```

**Fixed — parameterized queries:**

```typescript
// Prisma parameterizes automatically
const user = await db.user.findUnique({ where: { id: userId } });

// Raw SQL when needed — use tagged template or $queryRaw with Prisma
const users = await db.$queryRaw`SELECT * FROM users WHERE id = ${userId}`;
```

**Validate input at the boundary with Zod:**

```typescript
import { z } from 'zod';

const ParamsSchema = z.object({
  id: z.string().uuid(), // rejects anything that is not a UUID
});

fastify.get('/users/:id', async (request, reply) => {
  const { id } = ParamsSchema.parse(request.params);
  // id is now guaranteed to be a UUID — no room for injection
  return db.user.findUnique({ where: { id } });
});
```

---

### A04 — Insecure Design

Security requirements were never defined. The architecture has no rate limiting, no fraud controls, no separation of sensitive operations.

**Mitigation checklist during design:**

- Threat model each API endpoint: who calls it, what can they do, what is the worst case if abused?
- Apply rate limiting to authentication endpoints
- Require step-up authentication for sensitive actions (password change, payment, export)
- Log all sensitive operations to an audit trail

```typescript
import rateLimit from '@fastify/rate-limit';

// Strict rate limiting on auth endpoints
await fastify.register(rateLimit, {
  max: 5,
  timeWindow: '1 minute',
  keyGenerator: (request) => request.ip,
  errorResponseBuilder: () => ({
    error: 'Too many login attempts. Try again in 1 minute.',
  }),
});

fastify.post('/auth/login', async (request, reply) => {
  // ...
});
```

---

### A05 — Security Misconfiguration

Default credentials, directory listings enabled, verbose error messages exposing stack traces.

**Vulnerable — stack trace in production response:**

```typescript
fastify.setErrorHandler((error, request, reply) => {
  // Sends full stack trace to the client — exposes internals
  reply.status(500).send({ error: error.message, stack: error.stack });
});
```

**Fixed — sanitize error responses in production:**

```typescript
fastify.setErrorHandler((error, request, reply) => {
  const isProd = process.env.NODE_ENV === 'production';

  fastify.log.error({ err: error, requestId: request.id }, 'Unhandled error');

  reply.status(error.statusCode ?? 500).send({
    error: isProd ? 'Internal Server Error' : error.message,
    requestId: request.id, // useful for correlating logs without exposing internals
  });
});
```

---

### A06 — Vulnerable and Outdated Components

Using dependencies with known CVEs.

```bash
# Run in CI on every pull request
npm audit --audit-level=high

# Or use a dedicated tool
npx snyk test
```

See the [Dependency Auditing](dependency-auditing.md) chapter for a full CI integration guide.

---

### A07 — Identification and Authentication Failures

Weak session tokens, no account lockout, passwords stored without hashing.

Key mitigations:
- Use short-lived JWTs (15–60 minutes) with refresh tokens
- Implement account lockout after N failed attempts
- Enforce strong password policies via Zod
- Hash passwords with bcrypt (see A02)

```typescript
const PasswordSchema = z.string()
  .min(12)
  .regex(/[A-Z]/, 'Must contain uppercase')
  .regex(/[0-9]/, 'Must contain a number')
  .regex(/[^A-Za-z0-9]/, 'Must contain a special character');
```

---

### A08 — Software and Data Integrity Failures

Untrusted deserialization, unsigned updates, CI/CD pipelines with no artifact integrity checks.

In Node.js this surfaces as:
- Deserializing untrusted JSON with `eval()` or `new Function()`
- Accepting serialized objects and calling methods on them without validation
- Using `npm install` without a lockfile (allows substitution attacks)

Always commit `package-lock.json` or `bun.lockb` to source control. Pin exact dependency versions in critical systems.

---

### A09 — Security Logging and Monitoring Failures

No logs means breaches go undetected for months.

```typescript
import pino from 'pino';

const logger = pino({
  level: 'info',
  redact: ['password', 'token', 'authorization', 'cookie'], // never log secrets
});

// Log authentication events
async function loginUser(email: string, password: string) {
  const user = await db.user.findUnique({ where: { email } });

  if (!user || !(await verifyPassword(password, user.passwordHash))) {
    logger.warn({ email }, 'Failed login attempt');
    throw new Error('Invalid credentials');
  }

  logger.info({ userId: user.id }, 'User logged in');
  return createSession(user);
}
```

---

### A10 — Server-Side Request Forgery (SSRF)

The server fetches a URL provided by the user, allowing attackers to reach internal services.

**Vulnerable:**

```typescript
fastify.post('/fetch-url', async (request, reply) => {
  const { url } = request.body as { url: string };
  // Attacker sends url = "http://169.254.169.254/latest/meta-data/" (AWS metadata)
  const response = await fetch(url);
  return response.text();
});
```

**Fixed — allowlist approach:**

```typescript
import { URL } from 'url';

const ALLOWED_HOSTS = new Set(['api.example.com', 'cdn.example.com']);

fastify.post('/fetch-url', async (request, reply) => {
  const { url } = request.body as { url: string };

  let parsed: URL;
  try {
    parsed = new URL(url);
  } catch {
    return reply.status(400).send({ error: 'Invalid URL' });
  }

  if (!ALLOWED_HOSTS.has(parsed.hostname)) {
    return reply.status(403).send({ error: 'URL not allowed' });
  }

  const response = await fetch(parsed.toString());
  return response.text();
});
```

---

## Common Patterns & Best Practices

- **Validate all input at the boundary** — use Zod schemas on every Fastify route
- **Scope all database queries by authenticated identity** — never trust IDs from the request body
- **Apply defense in depth** — multiple layers (validation + ORM parameterization + authorization check)
- **Use Helmet.js** — sets security headers with a single import
- **Run `npm audit` in CI** — fail the build on high severity vulnerabilities
- **Log authentication and authorization events** — not the credentials themselves
- **Use environment variables for secrets** — never hardcode API keys or passwords

---

## Anti-Patterns to Avoid

- Building your own cryptography — use bcrypt, argon2, or libsodium
- Trusting `Content-Type` without validation
- Using `eval()` or `new Function()` on user input
- Returning stack traces to API clients in production
- Logging passwords, tokens, or PII
- Committing secrets to version control (even in private repos)
- Ignoring `npm audit` warnings because "we'll fix it later"

---

## Debugging & Troubleshooting

**"My Zod validation is not catching injection payloads"**
Check that you are calling `.parse()` (throws on invalid input) not `.safeParse()` without handling the error case. Also ensure the schema is strict enough — `z.string()` alone accepts SQL fragments, you need `z.string().uuid()` or similar constraints.

**"npm audit reports vulnerabilities in transitive dependencies"**
Use `npm audit fix` for auto-fixable issues. For complex cases, use `overrides` in `package.json` to pin a safe version of the transitive dep, or file an issue with the direct dependency to update it.

**"Rate limiting blocks legitimate users"**
Use a smarter key than IP for authenticated routes — key by `userId` instead of IP. Apply strict limits only to unauthenticated endpoints.

---

## Real-World Scenarios

**Scenario 1: Multi-tenant SaaS API**

Every resource (projects, documents, invoices) belongs to an organization. The pattern:

```typescript
// middleware/require-org-access.ts
export async function requireOrgAccess(request: FastifyRequest, reply: FastifyReply) {
  const { orgId } = request.params as { orgId: string };
  const userId = request.user.id;

  const membership = await db.orgMember.findUnique({
    where: { orgId_userId: { orgId, userId } },
  });

  if (!membership) {
    return reply.status(403).send({ error: 'Access denied' });
  }

  request.org = { id: orgId, role: membership.role };
}
```

Apply this middleware to all org-scoped routes. Then scope every query: `db.project.findMany({ where: { orgId } })`.

**Scenario 2: Public API with rate limiting per tier**

```typescript
fastify.addHook('preHandler', async (request, reply) => {
  const limit = request.user?.tier === 'pro' ? 1000 : 100;
  // Check rate limit against Redis using user.id as key
});
```

---

## Further Reading

- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [NodeJS Security Handbook — Snyk](https://snyk.io/learn/nodejs-security-best-practice/)
- Track 07: [Authentication & Authorization](authentication-authorization.md)
- Track 07: [Security Headers](security-headers.md)

---

## Summary

The OWASP Top 10 is not a checklist to complete once — it is a mindset shift. The most common vulnerability (Broken Access Control) is fixed by one habit: always scope database queries to the authenticated identity. Injection (A03) is fixed by never interpolating user input into queries or shell commands. Cryptographic failures (A02) are fixed by using bcrypt for passwords and HTTPS for transport. The rest of the top 10 falls into two buckets: configuration hygiene and operational discipline (logging, monitoring, dependency management). Addressing all ten categories is achievable in a typical Node.js application without any specialized security tooling — just consistent application of these patterns.
