# CORS & CSRF

## Visão Geral

CORS (Cross-Origin Resource Sharing) e CSRF (Cross-Site Request Forgery) são dois mecanismos de segurança web distintos, porém relacionados — ambos têm raiz na same-origin policy do navegador. Compreender mal ou configurar incorretamente qualquer um deles gera vulnerabilidades reais. Este capítulo explica como funcionam, por que existem e como implementá-los corretamente em uma API Fastify/Node.js.

---

## Pré-requisitos

- Fundamentos de HTTP (headers, métodos, cookies)
- Compreensão básica do modelo de segurança do navegador
- Familiaridade com Fastify

---

## Conceitos Fundamentais

### Same-Origin Policy

O navegador impõe que uma página de `https://app.example.com` não pode ler respostas de `https://api.other.com` a menos que o outro servidor permita explicitamente. "Origin" é definida como a combinação de protocolo + hostname + porta. `http://example.com` e `https://example.com` são origins diferentes.

### CORS — permitindo que origins confiáveis cruzem a fronteira

CORS é um mecanismo para que servidores relaxem a same-origin policy para origins específicas. Ele usa headers HTTP para informar ao navegador quais requisições cross-origin são permitidas.

### CSRF — surfando na sessão autenticada do usuário

Ataques CSRF enganam o navegador do usuário para que faça requisições autenticadas a um servidor no qual o usuário está logado, incorporando a requisição em uma página controlada pelo atacante. CORS não protege contra CSRF — os navegadores ainda *enviam* cookies cross-origin mesmo em requisições restritas.

---

## Exemplos Práticos

### Configuração de CORS no Fastify

```typescript
import cors from '@fastify/cors';

await fastify.register(cors, {
  origin: (origin, callback) => {
    const allowedOrigins = [
      'https://app.example.com',
      'https://www.example.com',
    ];

    // Permite requisições sem origin (server-to-server, curl)
    if (!origin) return callback(null, true);

    if (allowedOrigins.includes(origin)) {
      return callback(null, true);
    }

    // Permite localhost em desenvolvimento
    if (process.env.NODE_ENV !== 'production' && origin.startsWith('http://localhost')) {
      return callback(null, true);
    }

    return callback(new Error('Not allowed by CORS'), false);
  },
  methods: ['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS'],
  allowedHeaders: ['Content-Type', 'Authorization'],
  credentials: true,       // permite cookies/headers Authorization
  maxAge: 86400,           // faz cache da resposta preflight por 24 horas
});
```

### Entendendo preflight requests

Quando um navegador faz uma requisição cross-origin que não é "simples" (usa headers customizados, métodos além de GET/POST, ou content type diferente de `application/x-www-form-urlencoded`/`multipart/form-data`/`text/plain`), ele primeiro envia um preflight `OPTIONS`:

```
OPTIONS /api/users HTTP/1.1
Origin: https://app.example.com
Access-Control-Request-Method: POST
Access-Control-Request-Headers: Content-Type, Authorization
```

O servidor responde:

```
HTTP/1.1 204 No Content
Access-Control-Allow-Origin: https://app.example.com
Access-Control-Allow-Methods: GET, POST, PUT, DELETE
Access-Control-Allow-Headers: Content-Type, Authorization
Access-Control-Allow-Credentials: true
Access-Control-Max-Age: 86400
```

Se o servidor não responder corretamente, o navegador bloqueia a requisição real.

### Demonstração de ataque CSRF

Imagine que um usuário está logado em `https://bank.example.com` com um cookie de sessão. Um atacante cria uma página maliciosa:

```html
<!-- attacker.com/steal.html -->
<form action="https://bank.example.com/transfer" method="POST">
  <input type="hidden" name="amount" value="1000" />
  <input type="hidden" name="to" value="attacker-account" />
</form>
<script>document.forms[0].submit();</script>
```

Quando a vítima visita essa página, o navegador envia o formulário — e inclui o cookie de sessão automaticamente. O banco recebe uma requisição autenticada.

### Defesa contra CSRF 1: cookies SameSite

```typescript
fastify.post('/auth/login', async (request, reply) => {
  // ... valida credenciais ...

  reply.setCookie('session', sessionToken, {
    httpOnly: true,     // não acessível ao JavaScript
    secure: true,       // somente HTTPS
    sameSite: 'lax',   // bloqueia POST cross-site, permite navegação same-site e top-level
    path: '/',
    maxAge: 3600,
  });

  return { ok: true };
});
```

Valores de `SameSite`:
- `strict` — cookie nunca enviado em requisições cross-site (quebra fluxos OAuth e links compartilhados)
- `lax` — cookie não enviado em POSTs cross-site ou AJAX, mas enviado em navegação GET top-level (seguro e compatível)
- `none` — cookie sempre enviado cross-site (requer `secure: true`; necessário para iframes incorporados)

Para a maioria das APIs, `SameSite: lax` fornece proteção CSRF sem comprometer a usabilidade. APIs que usam headers `Authorization: Bearer` (JWTs) são imunes ao CSRF porque os navegadores não adicionam esse header automaticamente em requisições cross-site.

### Defesa contra CSRF 2: CSRF tokens (para apps baseados em cookies de sessão)

Se sua aplicação usa cookies de sessão e não pode usar `SameSite: strict/lax` (por exemplo, incorporada em iframe de origem diferente), implemente CSRF tokens:

```typescript
import crypto from 'crypto';

// Gera um CSRF token vinculado à sessão
function generateCsrfToken(): string {
  return crypto.randomBytes(32).toString('hex');
}

// Armazena na sessão no login
fastify.post('/auth/login', async (request, reply) => {
  // ... valida ...
  const csrfToken = generateCsrfToken();
  await db.session.create({ data: { token: sessionToken, csrfToken, userId } });

  reply.setCookie('session', sessionToken, { httpOnly: true, secure: true, sameSite: 'none' });
  // Retorna o CSRF token no corpo — JS pode lê-lo, mas JS cross-site não pode
  return { csrfToken };
});

// Valida o CSRF token em requisições que alteram estado
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

Uso no frontend:

```typescript
// Armazena o CSRF token da resposta de login
const { csrfToken } = await login(email, password);
localStorage.setItem('csrf_token', csrfToken);

// Inclui em toda requisição de mutação
async function apiPost(url: string, body: unknown) {
  return fetch(url, {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'X-CSRF-Token': localStorage.getItem('csrf_token') ?? '',
    },
    body: JSON.stringify(body),
    credentials: 'include', // envia cookies
  });
}
```

### O erro de "wildcard origin + credentials"

```typescript
// NUNCA faça isso — anula completamente o CORS
await fastify.register(cors, {
  origin: '*',
  credentials: true, // navegadores bloqueiam esta combinação — também é uma violação da spec CORS
});
```

Os navegadores se recusarão a enviar credentials quando `Access-Control-Allow-Origin: *`. A spec proíbe isso. A abordagem correta é repassar a origin específica confiável.

---

## Padrões Comuns e Boas Práticas

- **Para APIs com JWT + header Authorization**: nenhuma defesa CSRF é necessária — os navegadores não adicionam automaticamente headers `Authorization` em requisições cross-site
- **Para APIs com cookies de sessão**: use cookies `SameSite: lax` ou `SameSite: strict`; isso elimina a maioria dos riscos CSRF
- **Coloque origins em allowlist explicitamente** — nunca use padrões regex que possam ser contornados (`example.com.evil.com` corresponde a `.*example.com.*`)
- **Registre rejeições CORS** — frequentemente indicam erros de configuração ou tentativas de sondagem
- **Inclua `OPTIONS` nos métodos permitidos** — preflight requests devem ter sucesso

---

## Anti-Padrões a Evitar

- `origin: '*'` em uma API que usa cookies
- Validação de origin baseada em regex muito ampla
- Verificar o header `Referer` para proteção CSRF — pode estar ausente ou ser falsificado
- Armazenar CSRF tokens em cookies (estariam sujeitos à mesma vulnerabilidade CSRF)
- Desabilitar CORS em desenvolvimento (`origin: true` para todos) e esquecer de restringir em produção
- Confiar em `X-Requested-With: XMLHttpRequest` como defesa CSRF — o fetch moderno não exige isso e atacantes podem configurá-lo

---

## Debugging e Resolução de Problemas

**"CORS error: No 'Access-Control-Allow-Origin' header"**
O servidor não está respondendo ao preflight OPTIONS corretamente. Verifique se o plugin CORS está registrado antes dos handlers de rota e se a função `origin` retorna `true` para a origin que fez a requisição.

**"CORS error with credentials"**
Quando `credentials: true` (cliente com `credentials: 'include'`), o servidor deve responder com uma origin específica (não `*`) e `Access-Control-Allow-Credentials: true`.

**"Formulários funcionam mas fetch não"**
Provavelmente uma falha no preflight. Verifique o `Access-Control-Allow-Headers` — a requisição pode estar enviando um header customizado (como `Authorization`) que não está listado.

---

## Leitura Complementar

- [MDN: CORS](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS)
- [MDN: SameSite cookies](https://developer.mozilla.org/en-US/docs/Web/HTTP/Headers/Set-Cookie/SameSite)
- [OWASP CSRF Prevention Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Cross-Site_Request_Forgery_Prevention_Cheat_Sheet.html)
- Track 07: [Security Headers](security-headers.md)
- Track 07: [Autenticação & Autorização](authentication-authorization.md)

---

## Resumo

CORS e CSRF são soluções diferentes para problemas diferentes que compartilham a mesma causa raiz: a same-origin policy do navegador. CORS é um opt-in do servidor para permitir leituras cross-origin específicas. CSRF é uma classe de ataque onde o atacante faz o navegador da vítima realizar escritas cross-origin autenticadas. Para APIs modernas que usam `Authorization: Bearer` (JWT), CSRF não é uma preocupação — os navegadores não adicionam esse header automaticamente. Para apps baseados em cookies de sessão, `SameSite: lax` é a primeira linha de defesa CSRF, com CSRF tokens como complemento nos casos em que SameSite não é suficiente. CORS nunca deve usar `origin: '*'` com `credentials: true` — sempre coloque em allowlist as origins confiáveis específicas.
