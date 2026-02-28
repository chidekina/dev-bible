# Track 07 — Security

Understand how attackers think and how to build defenses that hold. Security is not a feature you add at the end — it is a discipline woven into every layer of your stack.

**Estimated time:** 2–3 weeks

---

## Topics

1. [OWASP Top 10](owasp-top-10.md) — the ten most critical web application security risks with mitigations
2. [Authentication & Authorization](authentication-authorization.md) — JWTs, sessions, OAuth 2.0, RBAC, ABAC
3. [Secrets Management](secrets-management.md) — environment variables, Vault, secret rotation, never committing credentials
4. [HTTPS & TLS](https-tls.md) — certificates, handshake, HSTS, certificate pinning
5. [CORS & CSRF](cors-csrf.md) — same-origin policy, preflight requests, CSRF tokens and SameSite cookies
6. [SQL Injection & XSS](sql-injection-xss.md) — parameterized queries, output encoding, Content Security Policy
7. [Security Headers](security-headers.md) — Helmet.js, CSP, HSTS, X-Frame-Options, Permissions-Policy
8. [Dependency Auditing](dependency-auditing.md) — npm audit, Snyk, supply chain attacks, lockfile hygiene

---

## Prerequisites

- Track 01 — Foundations (HTTP, Node.js basics)
- Track 03 — Backend (REST APIs, databases, Fastify/Express)
- Basic understanding of how web requests flow end-to-end

---

## What You'll Build

- A Fastify API with authentication middleware, rate limiting, and a full security header suite via Helmet
- A secrets management setup using `.env` + Vault-compatible pattern with secret rotation logic
- A CSRF-protected form submission flow with SameSite cookies
- An input validation layer using Zod that prevents injection attacks at the boundary
- A dependency audit CI step using `npm audit --audit-level=high`
