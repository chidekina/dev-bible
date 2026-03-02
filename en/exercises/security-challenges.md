# Security Challenges

> Practical security exercises covering authentication, authorization, secure coding, and hardening. Each exercise reflects a real production scenario. Stack assumed: Node.js 20+, TypeScript, Fastify, Zod, PostgreSQL. Target skill level: intermediate to advanced.

---

## Exercise 1 — Password Hashing with bcrypt (Easy)

**Scenario:** You are building a user registration and login system. Passwords must be stored securely so that a DB breach does not expose user credentials.

**Requirements:**
- Implement `hashPassword(plain: string): Promise<string>` using bcrypt with a cost factor of 12.
- Implement `verifyPassword(plain: string, hash: string): Promise<boolean>`.
- Both functions must live in a `src/lib/password.ts` module.
- Never log or return the plain-text password anywhere in the call stack.
- The cost factor must be configurable via a `BCRYPT_ROUNDS` env variable (default: 12, minimum: 10).

**Acceptance Criteria:**
- [ ] Storing the same password twice produces two different hashes (salt uniqueness).
- [ ] `verifyPassword('correct', hash)` returns `true`; `verifyPassword('wrong', hash)` returns `false`.
- [ ] Reducing `BCRYPT_ROUNDS` below 10 throws a configuration error at startup — not at hash time.
- [ ] Unit tests cover: correct verification, wrong password, tampered hash string.
- [ ] No plain-text password appears in any log output (verified by a test that captures `console` output).

**Hints:**
1. Use the `bcryptjs` package (pure JS, no native bindings needed) or `bcrypt` (native, faster in production).
2. `bcrypt.hash(password, rounds)` returns a salted hash. The salt is embedded in the hash string — no need to store it separately.
3. For the config guard: read `BCRYPT_ROUNDS` at module load time, parse to int, and throw if `< 10`.
4. In tests, set `BCRYPT_ROUNDS=10` to keep tests fast without compromising algorithm correctness.

---

## Exercise 2 — JWT Auth Middleware (Medium)

**Scenario:** Build a complete JWT authentication system: issue tokens on login, verify them on protected routes, and implement refresh token rotation.

**Requirements:**
- `POST /auth/login`: issues `{ accessToken, refreshToken }`. Access token expires in 15 minutes; refresh token expires in 7 days.
- `POST /auth/refresh`: accepts a valid refresh token, issues a new access token **and** a new refresh token (rotation). Invalidates the previous refresh token.
- A Fastify `preHandler` hook verifies the `Authorization: Bearer <token>` header on protected routes.
- Refresh tokens are stored in Redis. On rotation, delete the old token and set the new one.
- On logout (`POST /auth/logout`), delete the refresh token from Redis.

**Acceptance Criteria:**
- [ ] Using an expired access token on a protected route returns `401` with `{ code: "TOKEN_EXPIRED" }`.
- [ ] Using a revoked refresh token (after rotation) returns `401` — replay attack prevention.
- [ ] The refresh token in Redis uses the key format `rt:<userId>:<tokenId>` with a 7-day TTL.
- [ ] The `preHandler` hook passes `req.user` (decoded payload) to the route handler.
- [ ] Integration test covers the full rotation cycle: login → refresh → attempt reuse of old refresh token → fail.

**Hints:**
1. Use `@fastify/jwt` to simplify token sign/verify. Configure it with `secret`, `sign: { expiresIn: '15m' }`.
2. Embed a `jti` (JWT ID, a `crypto.randomUUID()`) in both tokens. Use `jti` as the Redis key suffix for revocation lookups.
3. Refresh endpoint: verify the incoming refresh token, confirm `jti` exists in Redis, then atomically delete old key and set new one.
4. Decode without verification first to extract `jti` for the revocation check only if you need the error type — always verify signature too.

---

## Exercise 3 — CSRF Token Middleware for Fastify (Medium)

**Scenario:** Your Fastify app serves a server-rendered frontend. You need to protect state-changing endpoints against Cross-Site Request Forgery attacks.

**Requirements:**
- Implement a Fastify plugin that generates a CSRF token and sets it as a cookie (`csrf-token`, `HttpOnly: false` so JS can read it).
- On `POST`, `PUT`, `PATCH`, `DELETE` requests, validate the `X-CSRF-Token` header against the cookie value.
- Token must be a cryptographically random 32-byte hex string, regenerated per session.
- Exempt paths (e.g., `/health`, `/webhooks/*`) from CSRF validation.
- The plugin must be configurable: `{ exemptPaths: string[], cookieName: string, headerName: string }`.

**Acceptance Criteria:**
- [ ] A `POST` request without the `X-CSRF-Token` header returns `403`.
- [ ] A `POST` request with a mismatched header value returns `403`.
- [ ] A `GET` request with no CSRF header passes through (CSRF not needed for safe methods).
- [ ] The token comparison uses `crypto.timingSafeEqual` — not `===`.
- [ ] `/webhooks/stripe` is exempted and accepts `POST` without a CSRF token.

**Hints:**
1. Register the plugin with `fastify.register(csrfPlugin, { exemptPaths: ['/health', '/webhooks/*'] })`.
2. In `onRequest` hook: skip safe methods (`GET`, `HEAD`, `OPTIONS`). Skip exempt paths (use a glob matcher or simple prefix check).
3. Token generation: `crypto.randomBytes(32).toString('hex')`.
4. Timing-safe comparison: both values must be the same byte length — pad or hash to normalize before comparing.
5. Store the expected token in the session (not just the cookie) to prevent cookie-to-header attacks from a same-site context.

---

## Exercise 4 — Dockerfile Security Audit (Medium)

**Scenario:** A colleague submitted the following Dockerfile. Identify all security issues and produce a hardened version.

**Given Dockerfile (intentionally flawed):**
```dockerfile
FROM node:latest

WORKDIR /app

COPY . .

RUN npm install

ENV NODE_ENV=production
ENV DATABASE_URL=postgres://admin:secret@db:5432/myapp
ENV JWT_SECRET=supersecretkey123

RUN npm run build

EXPOSE 3000

CMD ["node", "dist/index.js"]
```

**Requirements:**
- Identify at least 6 distinct security/best-practice issues in the Dockerfile above.
- Produce a hardened Dockerfile that fixes all identified issues.
- Use a multi-stage build to separate build-time and runtime dependencies.
- The final image must run as a non-root user.

**Acceptance Criteria:**
- [ ] No secrets are hardcoded in the image (secrets are injected at runtime via env or secrets manager).
- [ ] Uses a pinned base image tag (e.g., `node:20.11-alpine3.19`), not `latest`.
- [ ] Final image uses a non-root user (`USER node` or a custom `appuser`).
- [ ] `npm install` in the final stage only installs production dependencies (`--omit=dev`).
- [ ] The build stage and runtime stage are separate — `node_modules` for dev is not present in the final image.
- [ ] `.dockerignore` file is included, excluding at minimum: `.git`, `node_modules`, `.env*`, `*.log`.

**Hints:**
1. Issues to find: `latest` tag, hardcoded secrets, root user, no `.dockerignore`, dev dependencies in prod, no health check, unnecessary files copied.
2. Multi-stage: `FROM node:20-alpine AS builder` → install all deps + build → `FROM node:20-alpine AS runtime` → copy only `dist/` + prod `node_modules`.
3. Non-root: Alpine includes a `node` user. Add `USER node` before `CMD`.
4. Secrets: remove `ENV` secrets entirely. Document that they must be passed via `docker run -e` or a secrets manager at runtime.

---

## Exercise 5 — Helmet.js with Strict CSP (Easy)

**Scenario:** Your Express/Fastify API also serves an admin dashboard. Add security headers including a strict Content Security Policy that allows your frontend assets and blocks everything else.

**Requirements:**
- Integrate `helmet` (for Express) or `@fastify/helmet` (for Fastify).
- Configure a CSP that allows: scripts and styles only from `'self'` and your CDN (`https://cdn.yourdomain.com`), images from `'self'` and `data:`, fonts from `'self'`.
- Deny all `<frame>` and `<iframe>` embedding (`X-Frame-Options: DENY`).
- Enable `Strict-Transport-Security` with a 1-year max-age and `includeSubDomains`.
- Disable the `X-Powered-By` header.

**Acceptance Criteria:**
- [ ] `Content-Security-Policy` header is present on all responses.
- [ ] Inline scripts are blocked by the CSP (no `'unsafe-inline'`).
- [ ] `X-Frame-Options: DENY` is set.
- [ ] `Strict-Transport-Security: max-age=31536000; includeSubDomains` is set.
- [ ] `X-Powered-By` header is absent from all responses.
- [ ] Test that loading a page in a headless browser with the headers active does not trigger CSP violations for legitimate assets.

**Hints:**
1. `@fastify/helmet` wraps Helmet — pass CSP config via `contentSecurityPolicy: { directives: { ... } }`.
2. CSP `default-src 'self'` is the base — then add specific overrides for `script-src`, `style-src`, `img-src`, `font-src`.
3. To allow your CDN: `script-src: ["'self'", "https://cdn.yourdomain.com"]`.
4. For SPAs that use `eval` (some bundlers do): prefer using a nonce-based approach over `'unsafe-eval'`. Generate a nonce per request and inject it into the HTML template.

---

## Exercise 6 — Sliding Window Rate Limiter (Medium)

**Scenario:** Implement a rate limiter middleware using the sliding window algorithm backed by Redis. It must be accurate (no boundary exploits like fixed-window double-spend) and performant.

**Requirements:**
- Limit: 100 requests per 60-second sliding window per client (identified by IP or API key).
- Return `429` with `Retry-After` (seconds until the oldest request exits the window) and `X-RateLimit-Remaining` headers.
- The algorithm must be atomic (no race conditions under concurrent requests).
- Redis failure must fail open — requests are allowed with a `X-RateLimit-Bypassed: true` header.

**Acceptance Criteria:**
- [ ] A client sending 101 requests within 60 seconds gets a `429` on the 101st.
- [ ] After 61 seconds, the client can send 100 more requests (the window has slid past the oldest entries).
- [ ] Two concurrent requests at the 100-request boundary result in exactly one being accepted (atomicity via Lua script).
- [ ] `Retry-After` value is accurate — not just a fixed `60` but the actual time until a slot opens.
- [ ] Unit test does not require a real Redis instance (use `ioredis-mock` or inject a fake store).

**Hints:**
1. Sliding window with sorted set: `ZADD key <now_ms> <uuid>` + `ZREMRANGEBYSCORE key 0 <now_ms - windowMs>` + `ZCOUNT key -inf +inf` — wrap all three in a Lua script for atomicity.
2. `Retry-After`: get the score of the oldest entry (`ZRANGE key 0 0 WITHSCORES`) and compute `oldest + windowMs - now`.
3. Key TTL: after the Lua script, run `EXPIRE key windowSeconds` to auto-clean inactive clients.
4. The Lua script returns both the count and the oldest score so you can calculate `Retry-After` in one round trip.

---

## Exercise 7 — Input Validation with Zod on All Routes (Easy)

**Scenario:** An existing Fastify API has no input validation. Raw `req.body` is passed directly to the database layer, causing 500 errors on malformed input and potential injection vectors.

**Requirements:**
- Add Zod schemas for request body, query parameters, and URL params on every route.
- Return structured `422 Unprocessable Entity` errors with field-level details on validation failure.
- Wire Zod into Fastify using `fastify-type-provider-zod` so schemas generate TypeScript types automatically.
- All string inputs must be `.trim()`'d and length-capped at appropriate limits.
- IDs in URL params must validate as UUIDs (`z.string().uuid()`).

**Acceptance Criteria:**
- [ ] `POST /users` with a missing required field returns `422` with `{ errors: [{ field: "email", message: "Required" }] }`.
- [ ] `GET /users/:id` with `id = "not-a-uuid"` returns `422`, not a DB error.
- [ ] Route handler TypeScript types are inferred from the Zod schema — no manual type assertions.
- [ ] A string field passed with 10,000 characters is rejected with a descriptive error.
- [ ] `GET /items?page=-1` is rejected (page must be a positive integer).

**Hints:**
1. Install `fastify-type-provider-zod` and call `fastify.withTypeProvider<ZodTypeProvider>()` before registering routes.
2. Register the schema: `fastify.route({ schema: { body: createUserSchema, params: z.object({ id: z.string().uuid() }) }, handler })`.
3. Custom error handler: in `fastify.setErrorHandler`, detect `ZodError` (from `err instanceof ZodError`) and map issues to the structured format.
4. Length caps: `z.string().max(255)` for names, `z.string().max(10000)` for content. Trim with `.trim()` chained.

---

## Exercise 8 — SQL Injection Vulnerability Fix (Medium)

**Scenario:** A junior developer wrote the following query builder. Find the SQL injection vulnerabilities and rewrite the affected functions using parameterized queries.

**Given code (intentionally vulnerable):**
```typescript
async function getUserByEmail(email: string) {
  const query = `SELECT * FROM users WHERE email = '${email}'`;
  return db.query(query);
}

async function searchProducts(name: string, category: string) {
  const query = `
    SELECT * FROM products
    WHERE name LIKE '%${name}%'
    AND category = '${category}'
    ORDER BY created_at DESC
  `;
  return db.query(query);
}

async function updateUserRole(userId: string, role: string) {
  const query = `UPDATE users SET role = '${role}' WHERE id = ${userId}`;
  return db.query(query);
}
```

**Requirements:**
- Identify all injection points in the three functions above.
- Rewrite each function using parameterized queries (no string interpolation of user input).
- Add Zod validation for each function's inputs before the query is executed.
- Write a test that demonstrates the original code was vulnerable (use a payload like `'; DROP TABLE users; --`).

**Acceptance Criteria:**
- [ ] `getUserByEmail("' OR '1'='1")` returns zero results (not all users).
- [ ] `searchProducts("'; DROP TABLE products; --", "shoes")` does not drop the table.
- [ ] `updateUserRole("1 OR 1=1", "admin")` does not update all users.
- [ ] All three functions use `$1`, `$2` placeholders (or ORM parameter binding) — no template literals with user data.
- [ ] The `role` field in `updateUserRole` is validated against an allowlist (`['user', 'admin', 'moderator']`).

**Hints:**
1. PostgreSQL parameterized query: `db.query('SELECT * FROM users WHERE email = $1', [email])`.
2. `LIKE` with parameters: `db.query("SELECT * FROM products WHERE name LIKE $1", [`%${name}%`])` — the `%` wildcards go in the bound value, not the SQL string.
3. Never use string interpolation for `ORDER BY` column names either — use an allowlist map: `const allowed = { created_at: 'created_at', name: 'name' }`.
4. Zod for `role`: `z.enum(['user', 'admin', 'moderator'])`.

---

## Exercise 9 — Dependabot + npm audit in GitHub Actions (Easy)

**Scenario:** Your project has no automated dependency vulnerability scanning. Set up Dependabot for automated PRs and add `npm audit` as a CI gate.

**Requirements:**
- Create a `.github/dependabot.yml` that checks npm dependencies weekly and targets the `main` branch.
- Add a `security-audit` job to `.github/workflows/ci.yml` that runs `npm audit --audit-level=high` and fails the build on high or critical vulnerabilities.
- The audit job must run on every `push` to `main` and on every `pull_request`.
- Pin Dependabot to update only `patch` and `minor` versions automatically; `major` versions require manual review.
- Add a `license-checker` step that fails if any dependency uses a non-permissive license (GPL, AGPL).

**Acceptance Criteria:**
- [ ] `.github/dependabot.yml` is valid YAML and references the correct package ecosystem (`npm`).
- [ ] The `security-audit` CI job runs `npm audit --audit-level=high` and exits non-zero on failure.
- [ ] `npm audit --json` output is uploaded as a workflow artifact for debugging.
- [ ] A PR introducing a package with a known high CVE is blocked by the CI gate.
- [ ] Dependabot `ignore` rules are set for packages you pin intentionally (document why in a comment).

**Hints:**
1. `dependabot.yml` minimal config:
   ```yaml
   version: 2
   updates:
     - package-ecosystem: "npm"
       directory: "/"
       schedule:
         interval: "weekly"
   ```
2. For the audit step: `run: npm audit --audit-level=high`. The exit code is non-zero when vulnerabilities at or above the level are found.
3. Save audit results: `npm audit --json > audit-report.json` then `uses: actions/upload-artifact` with `path: audit-report.json`.
4. License check: use `license-checker` package — `npx license-checker --failOn "GPL;AGPL"`.

---

## Exercise 10 — Secrets Manager Wrapper (Medium)

**Scenario:** Your app reads secrets from environment variables. For production, secrets are stored in HashiCorp Vault (or AWS Secrets Manager). Build an abstraction that falls back gracefully.

**Requirements:**
- Implement `getSecret(key: string): Promise<string>` that:
  1. In development (`NODE_ENV !== 'production'`): reads from `process.env`.
  2. In production: fetches from Vault using the `node-vault` client.
- Cache resolved secrets in memory (Map) for the process lifetime — do not re-fetch on every call.
- If the secret is not found in either source, throw a typed `SecretNotFoundError`.
- Expose a `preloadSecrets(keys: string[]): Promise<void>` function for eager loading at startup.

**Acceptance Criteria:**
- [ ] In development, `getSecret('DATABASE_URL')` reads `process.env.DATABASE_URL`.
- [ ] In production, the Vault client is initialized once (singleton) using `VAULT_ADDR` and `VAULT_TOKEN` env vars.
- [ ] Calling `getSecret` twice for the same key only hits Vault once (verified by a spy/mock on the Vault client).
- [ ] `preloadSecrets(['DB_URL', 'JWT_SECRET'])` resolves all secrets before the HTTP server starts.
- [ ] `SecretNotFoundError` includes the key name in its message for fast debugging.

**Hints:**
1. Pattern: a module-level `const cache = new Map<string, string>()`. Check cache first, then fetch source.
2. Vault path convention: secrets live at `secret/data/<appName>/<key>`. The `node-vault` client returns `data.data[key]`.
3. Eager loading: `await Promise.all(keys.map(getSecret))` — if any throws, the startup sequence should abort.
4. In tests: set `NODE_ENV=test`, populate `process.env`, and assert the Vault client is never called.

---

## Exercise 11 — RBAC Middleware (Medium)

**Scenario:** Your API has multiple roles (`guest`, `user`, `moderator`, `admin`) with different permissions. Implement a Role-Based Access Control middleware that protects routes declaratively.

**Requirements:**
- Define a permissions map: each role has a set of permissions (e.g., `user:read`, `post:create`, `admin:delete`).
- Implement `requirePermission(permission: string)` as a Fastify `preHandler` factory.
- Roles are hierarchical: `admin` inherits all `moderator` permissions, `moderator` inherits all `user` permissions.
- The authenticated user's role comes from the decoded JWT payload (`req.user.role`).
- Return `403 Forbidden` (not `401`) when a valid user lacks the required permission.

**Acceptance Criteria:**
- [ ] A `user` accessing an `admin:delete` route gets `403`.
- [ ] An `admin` can access all routes regardless of required permission.
- [ ] Adding `requirePermission('post:create')` to a route correctly blocks `guest` users.
- [ ] Permissions for an unknown role default to empty (fail-secure, not fail-open).
- [ ] Unit test: verify the full hierarchy — `moderator` gets `user`-level permissions without explicit assignment.

**Hints:**
1. Define permissions with a plain object: `const ROLE_PERMISSIONS: Record<Role, Set<string>> = { admin: new Set([...all]), moderator: new Set([...]), ... }`.
2. Hierarchical inheritance: at definition time, spread parent permissions into child sets. Computed once at module load — not re-evaluated per request.
3. `requirePermission` factory: `(perm: string) => async (req, reply) => { if (!ROLE_PERMISSIONS[req.user.role]?.has(perm)) reply.code(403)... }`.
4. Always check that `req.user` exists before accessing `req.user.role` — if auth middleware failed to set it, return `401`.

---

## Exercise 12 — Security Headers Audit Checklist (Easy)

**Scenario:** You are conducting a security review of a production web application. Use the checklist below to audit the HTTP response headers and document findings.

**Requirements:**
- Make a `GET` request to the target URL (use `curl -I` or a headless browser).
- For each header in the checklist, mark: Present / Missing / Misconfigured and explain why it matters.
- Produce a remediation script (shell or Node.js) that adds all missing headers to a Fastify server.
- Re-run the audit after applying fixes to confirm all headers are present and correctly configured.

**Checklist:**
```
[ ] Content-Security-Policy        — prevents XSS
[ ] Strict-Transport-Security      — enforces HTTPS
[ ] X-Content-Type-Options: nosniff — prevents MIME sniffing
[ ] X-Frame-Options: DENY          — prevents clickjacking
[ ] Referrer-Policy                — controls referrer leakage
[ ] Permissions-Policy             — disables unused browser APIs
[ ] Cache-Control on auth routes   — prevents caching of sensitive responses
[ ] X-Powered-By absent            — hides technology stack
```

**Acceptance Criteria:**
- [ ] Audit report lists every header with status and a one-sentence explanation.
- [ ] At least 3 headers are intentionally misconfigured in a test server for you to find and fix.
- [ ] Remediation script is idempotent — running it twice does not duplicate headers.
- [ ] After remediation, `curl -I` output shows all 8 items checked off.
- [ ] `Cache-Control: no-store` is applied specifically to `/auth/*` routes, not globally.

**Hints:**
1. Quick audit: `curl -sI https://yourapp.com | grep -i 'content-security\|strict-transport\|x-content\|x-frame\|referrer\|permissions\|x-powered'`.
2. In Fastify, add headers globally via `fastify.addHook('onSend', (req, reply, payload, done) => { reply.header(...); done() })`.
3. `Referrer-Policy: strict-origin-when-cross-origin` is a safe default that prevents full URL leakage to third parties.
4. `Permissions-Policy: camera=(), microphone=(), geolocation=()` disables common APIs your app likely does not need.
