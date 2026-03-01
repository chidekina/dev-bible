# Track 07 — Segurança

Entenda como atacantes pensam e como construir defesas que resistam. Segurança não é uma funcionalidade que você adiciona no final — é uma disciplina incorporada em cada camada do seu stack.

**Tempo estimado:** 2–3 semanas

---

## Tópicos

1. [OWASP Top 10](owasp-top-10.md) — os dez riscos de segurança mais críticos em aplicações web, com mitigações
2. [Autenticação & Autorização](authentication-authorization.md) — JWTs, sessões, OAuth 2.0, RBAC, ABAC
3. [Gerenciamento de Secrets](secrets-management.md) — variáveis de ambiente, Vault, rotação de secrets, nunca commit de credenciais
4. [HTTPS & TLS](https-tls.md) — certificados, handshake, HSTS, certificate pinning
5. [CORS & CSRF](cors-csrf.md) — same-origin policy, preflight requests, CSRF tokens e cookies SameSite
6. [SQL Injection & XSS](sql-injection-xss.md) — queries parametrizadas, codificação de saída, Content Security Policy
7. [Security Headers](security-headers.md) — Helmet.js, CSP, HSTS, X-Frame-Options, Permissions-Policy
8. [Auditoria de Dependências](dependency-auditing.md) — npm audit, Snyk, ataques à cadeia de suprimentos, higiene do lockfile

---

## Pré-requisitos

- Track 01 — Fundamentos (HTTP, conceitos básicos de Node.js)
- Track 03 — Backend (REST APIs, bancos de dados, Fastify/Express)
- Compreensão básica do fluxo de requisições web de ponta a ponta

---

## O Que Você Vai Construir

- Uma API Fastify com middleware de autenticação, rate limiting e um conjunto completo de security headers via Helmet
- Um setup de gerenciamento de secrets usando `.env` + padrão compatível com Vault, com lógica de rotação de secrets
- Um fluxo de envio de formulário protegido contra CSRF com cookies SameSite
- Uma camada de validação de entrada com Zod que previne ataques de injeção na fronteira
- Um step de auditoria de dependências em CI usando `npm audit --audit-level=high`
