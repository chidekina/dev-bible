# Authentication & Authorization

## Overview

Authentication answers "who are you?" Authorization answers "what are you allowed to do?" They are different problems that are often conflated, and getting either wrong produces security vulnerabilities. This chapter covers the full spectrum: password hashing, JWT lifecycle, session management, OAuth 2.0, and access control models (RBAC and ABAC) — all implemented in a Node.js/TypeScript/Fastify context.

---

## Prerequisites

- HTTP cookies, headers, and the request/response cycle
- Basic TypeScript and async/await
- Familiarity with Fastify middleware (preHandler hooks)

---

## Core Concepts

### Authentication mechanisms

| Mechanism | Use case | Risk |
|-----------|----------|------|
| Username + password | Most web apps | Credential stuffing, weak passwords |
| JWT (stateless) | Microservices, SPAs | Token theft, no instant revocation |
| Session cookies (stateful) | Traditional web apps | Session hijacking |
| OAuth 2.0 (delegated) | "Login with Google/GitHub" | Misconfigured redirect URIs |
| API keys | Machine-to-machine | Key leakage, no expiry |

### Authorization models

**RBAC (Role-Based Access Control):** Users have roles, roles have permissions. Simple to reason about.

**ABAC (Attribute-Based Access Control):** Access decisions use attributes of the subject, resource, action, and environment. More expressive, more complex.

---

## Hands-On Examples

### Password hashing with bcrypt

```typescript
// src/lib/password.ts
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12;

export async function hashPassword(plaintext: string): Promise<string> {
  return bcrypt.hash(plaintext, SALT_ROUNDS);
}

export async function verifyPassword(
  plaintext: string,
  hash: string
): Promise<boolean> {
  return bcrypt.compare(plaintext, hash);
}
```

```typescript
// Registration
const passwordHash = await hashPassword(body.password);
await db.user.create({
  data: { email: body.email, passwordHash },
});

// Login
const user = await db.user.findUnique({ where: { email: body.email } });
if (!user || !(await verifyPassword(body.password, user.passwordHash))) {
  return reply.status(401).send({ error: 'Invalid credentials' });
}
```

Never store plaintext passwords. Never use MD5 or SHA-1. bcrypt (or argon2id) with a work factor of 12+ is the current standard.

---

### JWT-based stateless auth

```typescript
// src/lib/token.ts
import jwt from 'jsonwebtoken';
import { z } from 'zod';

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET!;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;

const AccessPayloadSchema = z.object({
  sub: z.string(),   // user ID
  role: z.string(),
  iat: z.number(),
  exp: z.number(),
});

export type AccessPayload = z.infer<typeof AccessPayloadSchema>;

export function signAccessToken(userId: string, role: string): string {
  return jwt.sign({ sub: userId, role }, ACCESS_SECRET, { expiresIn: '15m' });
}

export function signRefreshToken(userId: string): string {
  return jwt.sign({ sub: userId }, REFRESH_SECRET, { expiresIn: '7d' });
}

export function verifyAccessToken(token: string): AccessPayload {
  const raw = jwt.verify(token, ACCESS_SECRET);
  return AccessPayloadSchema.parse(raw);
}

export function verifyRefreshToken(token: string): { sub: string } {
  return jwt.verify(token, REFRESH_SECRET) as { sub: string };
}
```

```typescript
// src/plugins/auth.ts — Fastify plugin
import { FastifyPluginAsync } from 'fastify';
import fp from 'fastify-plugin';
import { verifyAccessToken, AccessPayload } from '../lib/token.js';

declare module 'fastify' {
  interface FastifyRequest {
    user: AccessPayload;
  }
}

const authPlugin: FastifyPluginAsync = async (fastify) => {
  fastify.decorateRequest('user', null);

  fastify.addHook('preHandler', async (request, reply) => {
    const authHeader = request.headers.authorization;
    if (!authHeader?.startsWith('Bearer ')) return; // allow unauthenticated routes to proceed

    const token = authHeader.slice(7);
    try {
      request.user = verifyAccessToken(token);
    } catch {
      return reply.status(401).send({ error: 'Invalid or expired token' });
    }
  });
};

export default fp(authPlugin);
```

```typescript
// Require authentication on a route
export async function requireAuth(request: FastifyRequest, reply: FastifyReply) {
  if (!request.user) {
    return reply.status(401).send({ error: 'Authentication required' });
  }
}

fastify.get('/profile', { preHandler: [requireAuth] }, async (request) => {
  return db.user.findUnique({ where: { id: request.user.sub } });
});
```

---

### Refresh token rotation

Refresh tokens allow issuing short-lived access tokens without forcing users to log in frequently. Store refresh tokens in the database so they can be revoked.

```typescript
// POST /auth/refresh
fastify.post('/auth/refresh', async (request, reply) => {
  const { refreshToken } = request.body as { refreshToken: string };

  let payload: { sub: string };
  try {
    payload = verifyRefreshToken(refreshToken);
  } catch {
    return reply.status(401).send({ error: 'Invalid refresh token' });
  }

  // Check token exists in DB and has not been used (rotation)
  const stored = await db.refreshToken.findUnique({
    where: { token: refreshToken },
  });

  if (!stored || stored.used || stored.expiresAt < new Date()) {
    // Possible token reuse — revoke all tokens for this user
    await db.refreshToken.updateMany({
      where: { userId: payload.sub },
      data: { used: true },
    });
    return reply.status(401).send({ error: 'Refresh token invalid or reused' });
  }

  // Mark old token as used (rotation)
  await db.refreshToken.update({
    where: { token: refreshToken },
    data: { used: true },
  });

  const user = await db.user.findUnique({ where: { id: payload.sub } });
  if (!user) return reply.status(401).send({ error: 'User not found' });

  // Issue new pair
  const newAccess = signAccessToken(user.id, user.role);
  const newRefresh = signRefreshToken(user.id);

  await db.refreshToken.create({
    data: {
      token: newRefresh,
      userId: user.id,
      expiresAt: new Date(Date.now() + 7 * 24 * 60 * 60 * 1000),
    },
  });

  return { accessToken: newAccess, refreshToken: newRefresh };
});
```

---

### Role-Based Access Control (RBAC)

```typescript
// src/middleware/require-role.ts
import { FastifyRequest, FastifyReply } from 'fastify';

type Role = 'user' | 'admin' | 'superadmin';

const ROLE_HIERARCHY: Record<Role, number> = {
  user: 1,
  admin: 2,
  superadmin: 3,
};

export function requireRole(minimumRole: Role) {
  return async function (request: FastifyRequest, reply: FastifyReply) {
    if (!request.user) {
      return reply.status(401).send({ error: 'Authentication required' });
    }

    const userLevel = ROLE_HIERARCHY[request.user.role as Role] ?? 0;
    const requiredLevel = ROLE_HIERARCHY[minimumRole];

    if (userLevel < requiredLevel) {
      return reply.status(403).send({ error: 'Insufficient permissions' });
    }
  };
}
```

```typescript
// Usage
fastify.delete('/admin/users/:id', {
  preHandler: [requireAuth, requireRole('admin')],
}, async (request) => {
  const { id } = request.params as { id: string };
  await db.user.delete({ where: { id } });
  return { success: true };
});
```

---

### Attribute-Based Access Control (ABAC)

ABAC is useful when access depends on the relationship between the user and the resource, not just the user's role.

```typescript
// src/lib/permissions.ts
type Action = 'read' | 'update' | 'delete';

interface Permission {
  userId: string;
  userRole: string;
  action: Action;
  resource: { ownerId: string; orgId?: string; visibility?: 'public' | 'private' };
  context?: { userOrgIds?: string[] };
}

export function canPerform({ userId, userRole, action, resource, context }: Permission): boolean {
  // Admins can do anything
  if (userRole === 'admin') return true;

  // Owner can do anything to their own resource
  if (resource.ownerId === userId) return true;

  // Public resources can be read by anyone
  if (action === 'read' && resource.visibility === 'public') return true;

  // Org members can read org resources
  if (
    action === 'read' &&
    resource.orgId &&
    context?.userOrgIds?.includes(resource.orgId)
  ) {
    return true;
  }

  return false;
}
```

```typescript
// Usage in a route
fastify.get('/documents/:id', { preHandler: [requireAuth] }, async (request, reply) => {
  const { id } = request.params as { id: string };
  const doc = await db.document.findUnique({ where: { id } });

  if (!doc) return reply.status(404).send({ error: 'Not found' });

  const userOrgIds = await db.orgMember
    .findMany({ where: { userId: request.user.sub } })
    .then((m) => m.map((x) => x.orgId));

  const allowed = canPerform({
    userId: request.user.sub,
    userRole: request.user.role,
    action: 'read',
    resource: { ownerId: doc.ownerId, orgId: doc.orgId, visibility: doc.visibility },
    context: { userOrgIds },
  });

  if (!allowed) return reply.status(403).send({ error: 'Access denied' });

  return doc;
});
```

---

### OAuth 2.0 — Login with GitHub

```typescript
// src/routes/auth.ts
import { Octokit } from 'octokit';

fastify.get('/auth/github', async (request, reply) => {
  const state = crypto.randomBytes(16).toString('hex');
  // Store state in session or short-lived cookie to prevent CSRF
  reply.setCookie('oauth_state', state, { httpOnly: true, maxAge: 300 });

  const url = new URL('https://github.com/login/oauth/authorize');
  url.searchParams.set('client_id', process.env.GITHUB_CLIENT_ID!);
  url.searchParams.set('redirect_uri', `${process.env.BASE_URL}/auth/github/callback`);
  url.searchParams.set('scope', 'user:email');
  url.searchParams.set('state', state);

  return reply.redirect(url.toString());
});

fastify.get('/auth/github/callback', async (request, reply) => {
  const { code, state } = request.query as { code: string; state: string };
  const storedState = request.cookies.oauth_state;

  if (!storedState || storedState !== state) {
    return reply.status(400).send({ error: 'Invalid OAuth state' });
  }

  // Exchange code for token
  const tokenResponse = await fetch('https://github.com/login/oauth/access_token', {
    method: 'POST',
    headers: { Accept: 'application/json', 'Content-Type': 'application/json' },
    body: JSON.stringify({
      client_id: process.env.GITHUB_CLIENT_ID,
      client_secret: process.env.GITHUB_CLIENT_SECRET,
      code,
    }),
  });

  const { access_token } = await tokenResponse.json() as { access_token: string };

  // Fetch user info
  const octokit = new Octokit({ auth: access_token });
  const { data: githubUser } = await octokit.rest.users.getAuthenticated();

  // Upsert user in our DB
  const user = await db.user.upsert({
    where: { githubId: String(githubUser.id) },
    create: {
      githubId: String(githubUser.id),
      email: githubUser.email ?? '',
      name: githubUser.name ?? githubUser.login,
      role: 'user',
    },
    update: { name: githubUser.name ?? githubUser.login },
  });

  const accessToken = signAccessToken(user.id, user.role);
  return reply.redirect(`${process.env.FRONTEND_URL}/auth?token=${accessToken}`);
});
```

---

## Common Patterns & Best Practices

- **15-minute access tokens, 7-day refresh tokens** — minimize the blast radius of token theft
- **Rotate refresh tokens on every use** — detect reuse attacks
- **Store refresh tokens in the database** — enables instant revocation
- **Use `httpOnly` cookies for refresh tokens in browser apps** — prevents XSS theft
- **Never put sensitive data in JWT payloads** — JWTs are base64-encoded, not encrypted
- **Validate OAuth `state` parameter** — prevents CSRF attacks on the OAuth flow
- **Use constant-time comparison for token validation** — prevents timing attacks

---

## Anti-Patterns to Avoid

- Rolling your own token generation with `Math.random()` — use `crypto.randomBytes()`
- Storing user passwords in plaintext or with MD5/SHA-1
- Long-lived access tokens (hours or days) without refresh rotation
- Checking roles client-side only — always enforce on the server
- Trusting the `userId` in the request body instead of extracting it from the token
- Using the same secret for access and refresh tokens
- Not logging authentication events (login, logout, failed attempts, password change)

---

## Debugging & Troubleshooting

**"jwt expired" errors immediately after issuing a token**
Check that the server clock is accurate. Clock skew between services causes JWT validation failures. Use `leeway` option in the JWT library to allow a small drift.

**Refresh token "invalid or reused" after a single use**
Check that you are not making multiple parallel requests with the same refresh token. Client-side, queue token refresh calls to prevent races.

**OAuth redirect URI mismatch**
The redirect URI in your authorization URL must exactly match what is registered in the OAuth provider's application settings — including the protocol, domain, port, and path.

---

## Real-World Scenarios

**Scenario: Admin-only route protection**

```typescript
const adminRoutes: FastifyPluginAsync = async (fastify) => {
  fastify.addHook('preHandler', async (request, reply) => {
    await requireAuth(request, reply);
    await requireRole('admin')(request, reply);
  });

  fastify.get('/admin/users', async () => db.user.findMany());
  fastify.delete('/admin/users/:id', async (request) => {
    const { id } = request.params as { id: string };
    return db.user.delete({ where: { id } });
  });
};
```

Registering the hook at the plugin level means every route inside the plugin requires auth + admin role, without repeating the `preHandler` on each route.

---

## Further Reading

- [JWT.io Introduction](https://jwt.io/introduction)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OAuth 2.0 Security Best Practices (RFC 9700)](https://datatracker.ietf.org/doc/html/rfc9700)
- Track 07: [CORS & CSRF](cors-csrf.md)
- Track 07: [Secrets Management](secrets-management.md)

---

## Summary

Authentication and authorization are distinct concerns that must both be implemented correctly. Authentication (proving identity) relies on password hashing with bcrypt, short-lived JWTs, and refresh token rotation. Authorization (enforcing permissions) relies on scoping every database query to the authenticated user and implementing RBAC or ABAC checks as server-side middleware. The most dangerous mistake in both areas is trusting the client: always extract identity from a verified token, always check permissions on the server, and always scope data queries by that verified identity.
