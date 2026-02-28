# SQL Injection & XSS

## Overview

SQL Injection and Cross-Site Scripting (XSS) are two of the oldest and most well-understood vulnerabilities in web applications — yet they continue to appear in production systems every year. SQL Injection allows attackers to manipulate database queries. XSS allows attackers to inject and execute JavaScript in victims' browsers. Both are preventable with the same fundamental technique: never trust user input, and always encode or parameterize it before use.

---

## Prerequisites

- Basic SQL knowledge (SELECT, INSERT, WHERE clauses)
- HTML and JavaScript fundamentals
- Node.js, TypeScript, and Fastify
- Basic understanding of the DOM

---

## Core Concepts

### SQL Injection

SQL injection occurs when user-supplied data is concatenated directly into a SQL query. The database interprets the user's input as SQL code rather than data.

**Classic example:** If `userId` is `1 OR 1=1`, the query `SELECT * FROM users WHERE id = ${userId}` becomes `SELECT * FROM users WHERE id = 1 OR 1=1`, which returns all users.

### XSS (Cross-Site Scripting)

XSS occurs when user-supplied data is rendered in an HTML page without encoding. The browser interprets the user's input as HTML or JavaScript code.

Three types:
- **Stored XSS**: malicious script saved to the database, served to all users who view the content
- **Reflected XSS**: malicious script in the URL, echoed back in the response
- **DOM-based XSS**: malicious script injected into the DOM directly by client-side JavaScript

---

## Hands-On Examples

### SQL Injection — parameterized queries

**Vulnerable:**

```typescript
// NEVER do this
const userId = request.params.id;
const result = await db.query(
  `SELECT * FROM users WHERE id = '${userId}'`
);
// userId = "1' OR '1'='1" → dumps all users
// userId = "1'; DROP TABLE users; --" → destroys data
```

**Fixed with Prisma (automatic parameterization):**

```typescript
// Prisma always uses parameterized queries under the hood
const user = await db.user.findUnique({
  where: { id: request.params.id },
});
// id is passed as a parameter, never interpolated into SQL
```

**Fixed with raw SQL using Prisma's tagged template:**

```typescript
// $queryRaw uses tagged template literals — values are parameterized
const users = await db.$queryRaw`
  SELECT * FROM users
  WHERE org_id = ${orgId}
  AND created_at > ${since}
`;
```

**Fixed with `pg` (node-postgres):**

```typescript
import { Pool } from 'pg';
const pool = new Pool();

// Always use parameterized queries with $1, $2, etc.
const result = await pool.query(
  'SELECT * FROM users WHERE id = $1 AND org_id = $2',
  [userId, orgId]
);
```

### Input validation with Zod

Parameterized queries prevent injection; Zod prevents nonsensical input from reaching the database layer at all:

```typescript
import { z } from 'zod';

const GetUserSchema = z.object({
  id: z.string().uuid(),
});

const SearchSchema = z.object({
  q: z.string().max(200).trim(),
  page: z.coerce.number().int().positive().max(1000).default(1),
  limit: z.coerce.number().int().positive().max(100).default(20),
});

fastify.get('/users/:id', async (request, reply) => {
  const { id } = GetUserSchema.parse(request.params);
  // id is guaranteed to be a UUID — no injection possible
  return db.user.findUnique({ where: { id } });
});

fastify.get('/users/search', async (request, reply) => {
  const { q, page, limit } = SearchSchema.parse(request.query);
  const offset = (page - 1) * limit;

  return db.$queryRaw`
    SELECT * FROM users
    WHERE name ILIKE ${'%' + q + '%'}
    LIMIT ${limit} OFFSET ${offset}
  `;
});
```

### NoSQL Injection (MongoDB-style)

ORMs that use object-based queries can also be injected if you pass raw user objects:

```typescript
// VULNERABLE — if body contains { username: { $gt: "" } }, MongoDB operator injection
const user = await UserModel.findOne({ username: request.body.username });

// FIXED — validate and extract scalar values
const { username } = z.object({ username: z.string().min(1).max(100) }).parse(request.body);
const user = await UserModel.findOne({ username });
```

---

### XSS — stored XSS prevention

When accepting user content that will be displayed as HTML, sanitize it with DOMPurify (browser) or `sanitize-html` (server):

```typescript
import sanitizeHtml from 'sanitize-html';

// User submits a blog post with HTML content
fastify.post('/posts', async (request, reply) => {
  const { title, content } = z.object({
    title: z.string().max(200),
    content: z.string().max(50000),
  }).parse(request.body);

  // Sanitize HTML to only allow safe tags and attributes
  const sanitizedContent = sanitizeHtml(content, {
    allowedTags: [
      'h1', 'h2', 'h3', 'p', 'br', 'ul', 'ol', 'li',
      'strong', 'em', 'a', 'code', 'pre', 'blockquote',
    ],
    allowedAttributes: {
      a: ['href', 'title'],
      code: ['class'],
    },
    allowedSchemes: ['http', 'https'], // no javascript: URLs
  });

  return db.post.create({
    data: { title, content: sanitizedContent, authorId: request.user.sub },
  });
});
```

### XSS — React (client-side)

React escapes HTML in JSX by default — `{userContent}` is safe. The danger is bypassing that:

```tsx
// SAFE — React escapes this
function Comment({ text }: { text: string }) {
  return <p>{text}</p>;
}

// DANGEROUS — bypasses React's escaping
function Comment({ html }: { html: string }) {
  return <p dangerouslySetInnerHTML={{ __html: html }} />;
}

// Safe use of dangerouslySetInnerHTML — only with sanitized content
import DOMPurify from 'dompurify';

function RichContent({ html }: { html: string }) {
  const clean = DOMPurify.sanitize(html);
  return <div dangerouslySetInnerHTML={{ __html: clean }} />;
}
```

### XSS — Content Security Policy

CSP is a browser mechanism that restricts what scripts, styles, and resources can be loaded. It provides defense-in-depth against XSS — even if a script is injected, CSP can prevent it from executing.

```typescript
import helmet from '@fastify/helmet';

await fastify.register(helmet, {
  contentSecurityPolicy: {
    directives: {
      defaultSrc: ["'self'"],
      scriptSrc: ["'self'", "'nonce-{nonce}'"], // nonces for inline scripts
      styleSrc: ["'self'", 'https://fonts.googleapis.com'],
      fontSrc: ["'self'", 'https://fonts.gstatic.com'],
      imgSrc: ["'self'", 'data:', 'https://cdn.example.com'],
      connectSrc: ["'self'", 'https://api.example.com'],
      objectSrc: ["'none'"],
      upgradeInsecureRequests: [],
    },
  },
});
```

For inline scripts (e.g., Next.js), use nonces:

```typescript
// Generate a unique nonce per request
fastify.addHook('onRequest', async (request, reply) => {
  const nonce = crypto.randomBytes(16).toString('base64');
  request.cspNonce = nonce;
});
```

### XSS — reflected XSS via URL parameters

```typescript
// VULNERABLE — echoing URL params back in HTML without encoding
fastify.get('/search', async (request, reply) => {
  const { q } = request.query as { q: string };
  return reply.type('text/html').send(`
    <h1>Results for: ${q}</h1>
  `);
  // q = "<script>document.cookie</script>" → script executes
});

// FIXED — never interpolate user input into HTML; use a template engine with auto-escaping
import Handlebars from 'handlebars';

const template = Handlebars.compile(`<h1>Results for: {{q}}</h1>`);
// Handlebars HTML-encodes {{q}} automatically
fastify.get('/search', async (request, reply) => {
  const { q } = request.query as { q: string };
  return reply.type('text/html').send(template({ q }));
});
```

---

## Common Patterns & Best Practices

- **Always use parameterized queries** — never string-interpolate user input into SQL
- **Validate input shape and type with Zod** at every API boundary
- **Sanitize HTML on input** (server-side) before storing rich text content
- **Let React/framework escape output** — never use `dangerouslySetInnerHTML` without sanitizing
- **Implement Content Security Policy** as a second line of defense against XSS
- **Encode output** when generating HTML outside a framework
- **Use ORM query builders** rather than raw SQL where possible

---

## Anti-Patterns to Avoid

- String concatenation to build SQL queries
- Trusting HTML sanitization on the client only — also sanitize server-side before storing
- Using `eval()` on user input
- Using `innerHTML` in JavaScript without sanitizing
- Relying solely on frontend validation — always validate on the backend
- Thinking "users can't see this input path" — automated tools can

---

## Debugging & Troubleshooting

**"Prisma is still vulnerable to injection"**
Prisma is safe for standard queries. Risk only appears if you use `db.$queryRawUnsafe()` with string concatenation — use `db.$queryRaw` tagged template instead.

**"My CSP is blocking legitimate scripts"**
Start with a report-only CSP: `Content-Security-Policy-Report-Only`. This logs violations without blocking. Tune the policy until there are no violations, then switch to enforcement mode.

**"sanitize-html is removing tags I need"**
Configure the `allowedTags` and `allowedAttributes` options explicitly. The defaults are conservative. Check the [sanitize-html docs](https://github.com/apostrophecms/sanitize-html) for the full configuration reference.

---

## Real-World Scenarios

**Scenario: Search endpoint safe against injection and XSS**

```typescript
const SearchSchema = z.object({
  q: z.string().trim().max(200).default(''),
  category: z.enum(['articles', 'products', 'users']).default('articles'),
});

fastify.get('/search', async (request, reply) => {
  const { q, category } = SearchSchema.parse(request.query);

  const results = await db.$queryRaw`
    SELECT id, title, excerpt
    FROM ${Prisma.raw(category)}  -- enum validated — safe to use as identifier
    WHERE to_tsvector('english', title || ' ' || content) @@ plainto_tsquery(${q})
    LIMIT 20
  `;

  return { q, category, results };
});
```

Note: `Prisma.raw()` should only be used for identifiers (table names, column names) that have been validated against an enum — never for user-supplied strings.

---

## Further Reading

- [OWASP SQL Injection Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/SQL_Injection_Prevention_Cheat_Sheet.html)
- [OWASP XSS Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross_Site_Scripting_Prevention_Cheat_Sheet.html)
- [Content Security Policy reference — MDN](https://developer.mozilla.org/en-US/docs/Web/HTTP/CSP)
- [sanitize-html](https://github.com/apostrophecms/sanitize-html)
- Track 07: [OWASP Top 10](owasp-top-10.md)

---

## Summary

SQL injection and XSS share the same root cause: unsanitized user input is interpreted as code. SQL injection is prevented by parameterized queries — every modern ORM and database driver supports them; there is never a reason to concatenate user input into SQL. XSS is prevented by encoding output — React and other modern frameworks do this by default, but `dangerouslySetInnerHTML`, string interpolation into HTML, and template engines without auto-escaping are all escape hatches that must be used carefully. Input validation with Zod narrows the attack surface further by rejecting malformed input before it reaches either the database or the template layer.
