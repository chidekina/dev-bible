# Authentication: JWT & OAuth2

## 1. What & Why

Authentication ("who are you?") and authorization ("what can you do?") are foundational to every production application. Getting them wrong does not produce a subtle bug — it produces a security incident.

This file covers the two dominant approaches to web authentication: session-based auth (stateful, server-side) and token-based auth (stateless, JWT), the OAuth2 authorization framework for delegated access, OpenID Connect for federated identity, and the practical security decisions that matter in production: where to store tokens, how to handle revocation, and why algorithm choice matters.

---

## 2. Core Concepts

### Session-Based Authentication

In session-based auth, the server maintains state about logged-in users.

**Flow:**
1. User submits credentials (email + password).
2. Server validates, creates a session record in a session store (memory, Redis, DB).
3. Server sets a `Set-Cookie: session=<session-id>` header.
4. Browser stores the cookie and sends it on every subsequent request.
5. Server looks up the session ID in the store to identify the user.

**Characteristics:**
- **Stateful**: server must maintain session store.
- **Revocation is trivial**: delete the session record to immediately invalidate.
- **Horizontal scaling**: multiple server instances need access to the same session store. In-process memory sessions only work with sticky sessions (one server per user), which breaks if that server goes down. Redis as a shared session store is the standard solution.
- **CSRF risk**: cookie-based auth is vulnerable to Cross-Site Request Forgery — mitigate with `SameSite=Strict/Lax` and CSRF tokens.

```typescript
// Session-based auth with Fastify + Redis
import fastifySession from '@fastify/session';
import fastifyCookie from '@fastify/cookie';
import RedisStore from 'connect-redis';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

app.register(fastifyCookie);
app.register(fastifySession, {
  secret: process.env.SESSION_SECRET!, // at least 32 chars
  store: new RedisStore({ client: redis }),
  cookie: {
    secure: process.env.NODE_ENV === 'production', // HTTPS only in prod
    httpOnly: true,    // not accessible via document.cookie
    sameSite: 'lax',   // CSRF protection
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 days
  },
  saveUninitialized: false,
});

app.post('/login', async (req, reply) => {
  const user = await validateCredentials(req.body.email, req.body.password);
  if (!user) return reply.status(401).send({ message: 'Invalid credentials' });

  req.session.userId = user.id;
  req.session.role = user.role;
  return reply.send({ message: 'Logged in' });
});

app.post('/logout', async (req, reply) => {
  await req.session.destroy(); // removes from Redis immediately
  return reply.send({ message: 'Logged out' });
});
```

### Token-Based Authentication (JWT)

JWT (JSON Web Token) is a compact, self-contained token encoding claims about the user, signed by the server.

**Flow:**
1. User submits credentials.
2. Server validates and issues a signed JWT.
3. Client stores the token and sends it as `Authorization: Bearer <token>` on each request.
4. Server verifies the signature — no database lookup needed.

**JWT Structure:**

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiJ1c2VyXzQyIiwibmFtZSI6IkFsaWNlIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzA5MDAwMDAwLCJleHAiOjE3MDkwMDA5MDB9
.SIGNATURE

Part 1: Base64url(header)   {"alg":"RS256","typ":"JWT"}
Part 2: Base64url(payload)  {"sub":"user_42","name":"Alice","role":"admin","iat":1709000000,"exp":1709000900}
Part 3: Signature           RS256(base64url(header) + "." + base64url(payload), privateKey)
```

> ⚠️ JWT payloads are base64url-encoded, NOT encrypted. Anyone can decode them. Never put sensitive information (passwords, PII, secrets) in JWT claims.

**Standard claims:**

| Claim | Meaning |
|-------|---------|
| `sub` | Subject — the user identifier |
| `iss` | Issuer — who issued the token |
| `aud` | Audience — who the token is intended for |
| `exp` | Expiration time (Unix timestamp) |
| `iat` | Issued at time |
| `jti` | JWT ID — unique identifier for this token (enables revocation) |

---

## 3. How It Works

### Signing Algorithms

**HS256 (HMAC-SHA256) — Symmetric:**
- Single secret key used for both signing and verification.
- Fast and simple.
- **Problem**: any service that needs to verify tokens must also know the secret — and therefore could forge tokens.
- Use when: single service signs and verifies its own tokens.

**RS256 (RSA-SHA256) — Asymmetric:**
- Private key signs the token (held only by the auth service).
- Public key verifies the token (can be distributed freely).
- Any service can verify without being able to forge.
- Slightly slower than HS256 due to asymmetric crypto.
- Use when: multiple microservices need to verify tokens, or tokens are verified by third parties.
- Public keys often exposed as JWKS (JSON Web Key Set) endpoint.

```typescript
import { SignJWT, jwtVerify, generateKeyPair, exportSPKI, exportPKCS8 } from 'jose';

// One-time key generation (store private key securely in secrets manager)
const { privateKey, publicKey } = await generateKeyPair('RS256');
const publicKeyPEM = await exportSPKI(publicKey);
const privateKeyPEM = await exportPKCS8(privateKey);

// Sign a token
async function signToken(userId: string, role: string): Promise<string> {
  return new SignJWT({ role })
    .setProtectedHeader({ alg: 'RS256' })
    .setSubject(userId)
    .setIssuedAt()
    .setIssuer('https://auth.example.com')
    .setAudience('https://api.example.com')
    .setExpirationTime('15m') // short-lived!
    .sign(privateKey);
}

// Verify a token — throws if invalid, expired, wrong audience, etc.
async function verifyToken(token: string) {
  const { payload } = await jwtVerify(token, publicKey, {
    issuer: 'https://auth.example.com',
    audience: 'https://api.example.com',
    algorithms: ['RS256'], // NEVER allow 'none'!
  });
  return payload;
}
```

### The alg:none Attack

```typescript
// Malicious JWT with alg:none in header:
// Header: {"alg":"none","typ":"JWT"}
// Payload: {"sub":"admin_user","role":"admin","exp":99999999999}
// Signature: (empty)

// A library that accepts the alg from the token header would accept this!
// This is why you MUST explicitly specify allowed algorithms:

await jwtVerify(token, publicKey, {
  algorithms: ['RS256'], // reject anything else, including 'none'
});
```

> ⚠️ The `alg:none` attack allows an attacker to forge arbitrary JWT payloads with no signature. Always hardcode the expected algorithm and never trust the `alg` header from an untrusted token.

### JWT Storage: httpOnly Cookie vs localStorage

```
localStorage:
  ✓ Easy to access from JavaScript
  ✗ Vulnerable to XSS — any injected script can read it
  ✗ Never use for sensitive tokens

httpOnly Cookie:
  ✓ Not accessible via JavaScript (document.cookie won't reveal it)
  ✓ Automatically sent on requests to the same domain
  ✗ Vulnerable to CSRF — mitigate with SameSite=Lax/Strict + CSRF token
  ✓ Use this for production auth tokens
```

```typescript
// Set JWT in httpOnly cookie (recommended)
reply
  .setCookie('access_token', jwt, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 15 * 60, // 15 minutes (seconds, not ms)
  });

// Auth middleware: read from cookie OR Authorization header
async function authenticate(req: FastifyRequest, reply: FastifyReply) {
  const token =
    req.cookies.access_token ??
    req.headers.authorization?.replace('Bearer ', '');

  if (!token) {
    return reply.status(401).send({ message: 'Authentication required' });
  }

  try {
    const payload = await verifyToken(token);
    req.user = payload;
  } catch {
    return reply.status(401).send({ message: 'Invalid or expired token' });
  }
}
```

### Refresh Token Rotation

Short-lived access tokens (15 minutes) + long-lived refresh tokens solve the revocation problem:

```typescript
// Full refresh token rotation implementation
interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

async function issueTokenPair(userId: string, role: string): Promise<TokenPair> {
  const accessToken = await signToken(userId, role); // 15 min expiry

  // Refresh token: opaque random string (NOT a JWT — stored in DB)
  const refreshToken = crypto.randomBytes(48).toString('base64url');
  const tokenHash = crypto.createHash('sha256').update(refreshToken).digest('hex');

  // Store hashed refresh token in DB with expiry
  await db.refreshToken.create({
    data: {
      tokenHash,
      userId,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 days
      familyId: crypto.randomUUID(), // track token family for reuse detection
    },
  });

  return { accessToken, refreshToken };
}

app.post('/auth/refresh', async (req, reply) => {
  const refreshToken = req.cookies.refresh_token;
  if (!refreshToken) return reply.status(401).send({ message: 'No refresh token' });

  const tokenHash = crypto.createHash('sha256').update(refreshToken).digest('hex');
  const stored = await db.refreshToken.findUnique({ where: { tokenHash } });

  if (!stored) {
    // Token not found — could be a stolen/replayed token
    // If the family exists but this hash is missing, someone reused an old token
    // Revoke the entire family as a precaution
    return reply.status(401).send({ message: 'Invalid refresh token' });
  }

  if (stored.expiresAt < new Date()) {
    await db.refreshToken.delete({ where: { tokenHash } });
    return reply.status(401).send({ message: 'Refresh token expired' });
  }

  // Rotate: delete old token, issue new pair
  await db.refreshToken.delete({ where: { tokenHash } });

  const user = await db.user.findUniqueOrThrow({ where: { id: stored.userId } });
  const tokens = await issueTokenPair(user.id, user.role);

  reply
    .setCookie('access_token', tokens.accessToken, {
      httpOnly: true, secure: true, sameSite: 'lax', maxAge: 900,
    })
    .setCookie('refresh_token', tokens.refreshToken, {
      httpOnly: true, secure: true, sameSite: 'lax', path: '/auth/refresh',
      maxAge: 30 * 24 * 60 * 60,
    });

  return reply.send({ message: 'Token refreshed' });
});
```

> 💡 Set the refresh token cookie path to `/auth/refresh` — it is only sent on requests to that endpoint, not on every API request. This limits its exposure.

---

## 4. OAuth2

OAuth2 is an authorization framework that allows a user to grant a third-party application limited access to their account without sharing their credentials.

### Authorization Code Flow + PKCE

The recommended flow for web and mobile apps. PKCE (Proof Key for Code Exchange) prevents authorization code interception attacks.

```
1. Client generates:
   code_verifier = random high-entropy string (43–128 chars)
   code_challenge = BASE64URL(SHA256(code_verifier))

2. Client redirects user to authorization server:
   GET https://auth.example.com/authorize?
     response_type=code
     &client_id=MY_CLIENT_ID
     &redirect_uri=https://myapp.com/callback
     &scope=read:user write:posts
     &state=RANDOM_CSRF_TOKEN     ← prevent CSRF
     &code_challenge=ABC123...     ← PKCE challenge
     &code_challenge_method=S256

3. User authenticates and grants permission.

4. Auth server redirects to redirect_uri:
   GET https://myapp.com/callback?code=AUTH_CODE&state=SAME_CSRF_TOKEN

5. Client verifies state parameter matches.

6. Client exchanges code for tokens:
   POST https://auth.example.com/token
   {
     grant_type: 'authorization_code',
     code: AUTH_CODE,
     redirect_uri: 'https://myapp.com/callback',
     client_id: MY_CLIENT_ID,
     code_verifier: ORIGINAL_VERIFIER  ← PKCE proof
   }

7. Auth server verifies: SHA256(code_verifier) == code_challenge
   Returns: { access_token, refresh_token, id_token, expires_in }
```

```typescript
// PKCE implementation in TypeScript
import crypto from 'crypto';

function generatePKCE(): { verifier: string; challenge: string } {
  const verifier = crypto.randomBytes(32).toString('base64url');
  const challenge = crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url');
  return { verifier, challenge };
}

// Store verifier in session (httpOnly cookie or sessionStorage for SPAs)
app.get('/auth/login', async (req, reply) => {
  const state = crypto.randomBytes(16).toString('hex');
  const { verifier, challenge } = generatePKCE();

  // Store in session for verification after redirect
  req.session.oauthState = state;
  req.session.codeVerifier = verifier;

  const params = new URLSearchParams({
    response_type: 'code',
    client_id: process.env.OAUTH_CLIENT_ID!,
    redirect_uri: `${process.env.APP_URL}/auth/callback`,
    scope: 'openid email profile',
    state,
    code_challenge: challenge,
    code_challenge_method: 'S256',
  });

  return reply.redirect(`https://accounts.google.com/o/oauth2/v2/auth?${params}`);
});

app.get('/auth/callback', async (req, reply) => {
  const { code, state } = req.query as { code: string; state: string };

  // Verify CSRF state
  if (state !== req.session.oauthState) {
    return reply.status(400).send({ message: 'State mismatch — possible CSRF' });
  }

  // Exchange code for tokens
  const tokenResponse = await fetch('https://oauth2.googleapis.com/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'authorization_code',
      code,
      redirect_uri: `${process.env.APP_URL}/auth/callback`,
      client_id: process.env.OAUTH_CLIENT_ID!,
      client_secret: process.env.OAUTH_CLIENT_SECRET!,
      code_verifier: req.session.codeVerifier!,
    }),
  });

  const tokens = await tokenResponse.json();
  // tokens.id_token contains the user's identity (OpenID Connect)
  // tokens.access_token for API calls
});
```

### Client Credentials Flow (Machine-to-Machine)

```typescript
// M2M: service A calls service B
async function getServiceToken(): Promise<string> {
  const response = await fetch('https://auth.example.com/oauth/token', {
    method: 'POST',
    headers: { 'Content-Type': 'application/x-www-form-urlencoded' },
    body: new URLSearchParams({
      grant_type: 'client_credentials',
      client_id: process.env.SERVICE_CLIENT_ID!,
      client_secret: process.env.SERVICE_CLIENT_SECRET!,
      scope: 'read:inventory write:orders',
    }),
  });
  const { access_token, expires_in } = await response.json();
  return access_token;
}

// Cache the token until it expires (minus a buffer)
let cachedToken: { token: string; expiresAt: number } | null = null;

async function getToken(): Promise<string> {
  const now = Date.now();
  if (cachedToken && cachedToken.expiresAt > now + 60_000) {
    return cachedToken.token;
  }
  const token = await getServiceToken();
  // Parse expiry from response and cache
  cachedToken = { token, expiresAt: now + 3600_000 };
  return token;
}
```

### OpenID Connect

OpenID Connect (OIDC) is a thin identity layer built on top of OAuth2. It adds:
- **ID Token**: a JWT containing the user's identity claims (sub, email, name, picture).
- **UserInfo endpoint**: returns additional user claims.
- **Standard claims**: `sub`, `email`, `email_verified`, `name`, `given_name`, `family_name`, `picture`, `locale`.

```typescript
import { Issuer, generators } from 'openid-client';

// Discover the provider's configuration automatically
const googleIssuer = await Issuer.discover('https://accounts.google.com');
const client = new googleIssuer.Client({
  client_id: process.env.GOOGLE_CLIENT_ID!,
  client_secret: process.env.GOOGLE_CLIENT_SECRET!,
  redirect_uris: [`${process.env.APP_URL}/auth/callback`],
  response_types: ['code'],
});

// Get user identity from ID token
app.get('/auth/callback', async (req, reply) => {
  const params = client.callbackParams(req.url);
  const tokenSet = await client.callback(
    `${process.env.APP_URL}/auth/callback`,
    params,
    { code_verifier: req.session.codeVerifier, state: req.session.oauthState }
  );

  const claims = tokenSet.claims(); // ID token claims
  console.log(claims.sub);         // unique user identifier
  console.log(claims.email);       // user@gmail.com
  console.log(claims.name);        // "Alice Smith"

  // Create or find user in your DB
  const user = await upsertUser({
    providerId: claims.sub,
    provider: 'google',
    email: claims.email!,
    name: claims.name!,
  });

  // Issue your own session/JWT
  const tokens = await issueTokenPair(user.id, user.role);
  // ... set cookies, redirect to app
});
```

---

## 5. Common Mistakes & Pitfalls

**Storing JWTs in localStorage:**

```typescript
// WRONG — XSS can steal tokens
localStorage.setItem('token', jwt);
const token = localStorage.getItem('token');

// CORRECT — httpOnly cookie, not accessible to JavaScript
reply.setCookie('access_token', jwt, { httpOnly: true, secure: true });
```

**Long-lived JWTs without revocation:**

```typescript
// WRONG — 30-day access token with no revocation mechanism
const token = jwt.sign({ userId }, secret, { expiresIn: '30d' });
// A stolen token is valid for 30 days with no way to invalidate

// CORRECT — short-lived access + refresh token rotation
const accessToken = await signToken(userId, role); // 15 min
const refreshToken = await issueRefreshToken(userId); // stored in DB, can be revoked
```

**Not validating JWT claims:**

```typescript
// WRONG — only verifies signature, not audience/issuer
jwt.verify(token, publicKey);

// CORRECT — verify all claims
await jwtVerify(token, publicKey, {
  issuer: 'https://auth.example.com',
  audience: 'https://api.example.com',
  algorithms: ['RS256'],
  clockTolerance: 30, // 30 second tolerance for clock skew between servers
});
```

**Using the same token for different purposes:**

```typescript
// WRONG — same JWT used for both auth and email verification
// An attacker who steals an auth token could use it to verify emails

// CORRECT — issue purpose-specific tokens with different audiences
const emailVerifyToken = new SignJWT({ purpose: 'email_verification' })
  .setAudience('https://auth.example.com/verify-email')
  .setExpirationTime('24h')
  .sign(privateKey);
```

---

## 6. When to Use / Not Use

| Approach | Use When | Avoid When |
|----------|----------|------------|
| Sessions + Redis | Traditional web app, revocation required immediately, simple setup | Massive scale, microservices that all need auth |
| JWT (access + refresh) | Stateless APIs, microservices, mobile clients | When immediate revocation is required with no compromise |
| OAuth2 + OIDC | "Login with Google/GitHub", delegated access to third-party APIs, federated identity | Internal apps with no external identity provider |
| API Keys | M2M, developer-facing APIs, CLI tools | User-facing auth (use OAuth2 instead) |

---

## 7. Real-World Scenario

**Problem:** A SaaS application stores JWTs in localStorage. Security audit finds that a third-party analytics script has an XSS vulnerability — all user tokens are exposed.

**Resolution and new architecture:**

```typescript
// New auth flow: httpOnly cookie + CSRF protection

// 1. Login: set JWT in httpOnly cookie
app.post('/auth/login', async (req, reply) => {
  const user = await validateCredentials(req.body.email, req.body.password);
  if (!user) return reply.status(401).send({ message: 'Invalid credentials' });

  const { accessToken, refreshToken } = await issueTokenPair(user.id, user.role);

  // CSRF token: random value stored in cookie AND returned in response body
  // JavaScript can read the body but not the httpOnly cookies
  const csrfToken = crypto.randomBytes(16).toString('hex');

  return reply
    .setCookie('access_token', accessToken, {
      httpOnly: true, secure: true, sameSite: 'strict', maxAge: 900
    })
    .setCookie('refresh_token', refreshToken, {
      httpOnly: true, secure: true, sameSite: 'strict',
      path: '/auth/refresh', maxAge: 30 * 24 * 3600,
    })
    .setCookie('csrf_token', csrfToken, {
      httpOnly: false, // JS-readable for CSRF protection
      secure: true, sameSite: 'strict',
    })
    .send({ csrfToken, user: { id: user.id, name: user.name } });
});

// 2. CSRF middleware: verify X-CSRF-Token header matches cookie
app.addHook('preHandler', async (req, reply) => {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) return;
  const headerToken = req.headers['x-csrf-token'];
  const cookieToken = req.cookies.csrf_token;
  if (!headerToken || headerToken !== cookieToken) {
    return reply.status(403).send({ message: 'CSRF token mismatch' });
  }
});

// 3. Client JavaScript: read csrf_token cookie and set header
// (Cannot steal access_token because it is httpOnly)
const response = await fetch('/api/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': getCookie('csrf_token'), // read from non-httpOnly cookie
  },
  body: JSON.stringify({ title: 'Hello' }),
  credentials: 'include', // send httpOnly cookies
});
```

**Result:** Even if an XSS vulnerability exists, the attacker can run scripts but cannot read the `access_token` or `refresh_token` cookies. The CSRF token in the httpOnly access token cookie prevents cross-site forgery.

---

## 8. Interview Questions

**Q1: What are the trade-offs between session-based and JWT-based authentication?**

Sessions are stateful: the server stores session data (in Redis for horizontal scale), revocation is instant (delete the session). JWTs are stateless: the server only needs a public key to verify, scales easily across services, but revocation requires workarounds (short expiry + refresh token blocklist, or maintaining a token blocklist which re-introduces statefulness). Sessions suit traditional web apps; JWTs suit APIs and microservices where stateless verification across services is valuable.

**Q2: Why should you store JWTs in httpOnly cookies instead of localStorage?**

localStorage is accessible to any JavaScript on the page — including injected scripts from XSS vulnerabilities, compromised third-party SDKs, or browser extensions. An httpOnly cookie cannot be read by JavaScript at all (`document.cookie` does not reveal it), making it immune to token theft via XSS. The trade-off is CSRF risk, which is mitigated with `SameSite=Lax/Strict` and a CSRF token.

**Q3: What is PKCE and why is it needed?**

PKCE (Proof Key for Code Exchange) is a security extension to the OAuth2 Authorization Code flow for public clients (mobile apps, SPAs) that cannot securely store a client secret. The client generates a random `code_verifier` and sends its SHA-256 hash (`code_challenge`) to the auth server with the initial request. When exchanging the code for tokens, it sends the original `code_verifier`. This proves the token exchange request comes from the same client that initiated the flow — preventing authorization code interception attacks where a malicious app intercepts the redirect URI.

**Q4: Explain refresh token rotation and how it detects token theft.**

Short-lived access tokens (15 minutes) + long-lived refresh tokens (30 days). When the access token expires, the client sends the refresh token to get a new pair. On use, the old refresh token is invalidated and a new one issued. If an attacker steals a refresh token and uses it after the legitimate client already rotated it, the server detects a reuse: the old token (which was already invalidated) is presented again. The server should then revoke the entire token family, forcing the user to re-authenticate. This limits the damage window from any single token theft.

**Q5: What is the difference between HS256 and RS256?**

HS256 uses a single shared secret for both signing and verification — symmetric. RS256 uses an RSA key pair: the private key signs tokens (only the issuer has it), and the public key verifies them (freely distributable). RS256 is preferred in microservice architectures where multiple services need to verify tokens — they only need the public key and cannot forge tokens. HS256 is simpler and faster for single-service setups.

**Q6: What is the alg:none JWT attack and how do you prevent it?**

The JWT header includes an `alg` field. If a library trusts this field blindly and the attacker crafts a token with `"alg":"none"`, the library may skip signature verification entirely, accepting any payload. Prevention: always hardcode the expected algorithm on the verification side and never accept `none`. Libraries like `jose` require you to specify `algorithms: ['RS256']` in the verify call, rejecting any other algorithm.

**Q7: What is the difference between OAuth2 and OpenID Connect?**

OAuth2 is an authorization framework for granting delegated access to resources. It answers "can application X access resource Y on behalf of user Z?" — it issues access tokens but defines no standard for user identity. OpenID Connect (OIDC) is an identity layer built on OAuth2. It adds an ID Token (a signed JWT with standard user identity claims: sub, email, name), a UserInfo endpoint, and a discovery mechanism. OAuth2 = authorization; OIDC = OAuth2 + authentication + identity.

**Q8: How do you revoke a JWT before it expires?**

JWTs are self-contained and stateless — the server cannot "reach into" a token and invalidate it. Strategies: (1) **Short expiry** — accept a 15-minute revocation delay by using very short-lived tokens with refresh rotation. (2) **Token blocklist** — maintain a set of revoked `jti` (JWT ID) claims in Redis with TTL equal to the token's remaining lifetime. Check the blocklist on every request. (3) **Version field** — include a `tokenVersion` in the JWT; when user logs out or changes password, increment their version in DB; tokens with an old version are rejected (requires one DB lookup per request — equivalent to sessions). Approach 1 is most common; approach 2 for security-critical logout; approach 3 for "log out all devices."

---

## 9. Exercises

**Exercise 1: Implement JWT auth middleware**

Build a Fastify plugin that:
- Reads the JWT from the `Authorization: Bearer <token>` header OR an httpOnly cookie
- Verifies the signature using RS256 with a public key loaded from an environment variable
- Sets `req.user` with the decoded payload
- Returns 401 with RFC 7807 error for missing/invalid/expired tokens
- Allows specifying required roles: `app.get('/admin', { preHandler: authenticate(['admin']) }, handler)`

*Hint:* Use the `jose` library for verification. Load the public key with `importSPKI()`.

**Exercise 2: Refresh token rotation**

Implement the full refresh token flow:
- `POST /auth/login` → issue access token (15 min) + refresh token (30 days), set in httpOnly cookies
- `POST /auth/refresh` → rotate refresh token, issue new pair
- `POST /auth/logout` → invalidate refresh token, clear cookies
- Detect refresh token reuse: if a token that was already rotated is presented, revoke all tokens for that user and return 401

*Hint:* Store refresh tokens hashed (`sha256`) in a DB table with `userId`, `familyId`, `tokenHash`, `expiresAt`. On reuse detection, delete all records matching the `familyId`.

**Exercise 3: OAuth2 with GitHub**

Build a "Login with GitHub" flow:
- Generate the authorization URL with state parameter
- Handle the callback, verify state, exchange code for token
- Call GitHub's `/user` API to get user info
- Upsert the user in your DB (match by GitHub ID or email)
- Issue your own JWT session

*Hint:* GitHub uses standard OAuth2 without PKCE (it's a confidential client). Use `client_id` + `client_secret` in the token exchange.

---

## 10. Further Reading

- [JWT RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)
- [OAuth 2.0 RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
- [OAuth 2.0 Security Best Practices RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)
- [PKCE RFC 7636](https://www.rfc-editor.org/rfc/rfc7636)
- [OpenID Connect Core specification](https://openid.net/specs/openid-connect-core-1_0.html)
- [jose library (TypeScript JWT/JWK)](https://github.com/panva/jose)
- [OWASP Authentication Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP JWT Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [Auth0 blog — Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- Book: *OAuth 2 in Action* by Justin Richer & Antonio Sanso
