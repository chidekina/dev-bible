# Autenticação: JWT e OAuth2

## 1. O Que é e Por Que Usar

Autenticação ("quem é você?") e autorização ("o que você pode fazer?") são a base de toda aplicação em produção. Errar nelas não produz um bug sutil — produz um incidente de segurança.

Este arquivo cobre as duas abordagens dominantes de autenticação web: autenticação baseada em sessão (stateful, server-side) e autenticação baseada em token (stateless, JWT), o framework de autorização OAuth2 para acesso delegado, OpenID Connect para identidade federada, e as decisões práticas de segurança que importam em produção: onde armazenar tokens, como lidar com revogação e por que a escolha do algoritmo importa.

---

## 2. Conceitos Fundamentais

### Autenticação Baseada em Sessão

Na autenticação baseada em sessão, o servidor mantém estado sobre os usuários logados.

**Fluxo:**
1. Usuário envia credenciais (email + senha).
2. Servidor valida, cria um registro de sessão em um session store (memória, Redis, banco de dados).
3. Servidor define um cookie `Set-Cookie: session=<session-id>`.
4. Browser armazena o cookie e o envia em cada requisição subsequente.
5. Servidor consulta o session ID no store para identificar o usuário.

**Características:**
- **Stateful**: o servidor deve manter o session store.
- **Revogação trivial**: delete o registro de sessão para invalidar imediatamente.
- **Escalabilidade horizontal**: múltiplas instâncias de servidor precisam acessar o mesmo session store. Sessões em memória só funcionam com sticky sessions (um servidor por usuário), o que quebra se esse servidor cair. Redis como session store compartilhado é a solução padrão.
- **Risco de CSRF**: autenticação baseada em cookie é vulnerável a Cross-Site Request Forgery — mitigue com `SameSite=Strict/Lax` e tokens CSRF.

```typescript
// Autenticação baseada em sessão com Fastify + Redis
import fastifySession from '@fastify/session';
import fastifyCookie from '@fastify/cookie';
import RedisStore from 'connect-redis';
import { Redis } from 'ioredis';

const redis = new Redis(process.env.REDIS_URL);

app.register(fastifyCookie);
app.register(fastifySession, {
  secret: process.env.SESSION_SECRET!, // pelo menos 32 chars
  store: new RedisStore({ client: redis }),
  cookie: {
    secure: process.env.NODE_ENV === 'production', // apenas HTTPS em prod
    httpOnly: true,    // não acessível via document.cookie
    sameSite: 'lax',   // proteção contra CSRF
    maxAge: 7 * 24 * 60 * 60 * 1000, // 7 dias
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
  await req.session.destroy(); // remove do Redis imediatamente
  return reply.send({ message: 'Logged out' });
});
```

### Autenticação Baseada em Token (JWT)

JWT (JSON Web Token) é um token compacto e auto-contido que codifica claims sobre o usuário, assinado pelo servidor.

**Fluxo:**
1. Usuário envia credenciais.
2. Servidor valida e emite um JWT assinado.
3. Cliente armazena o token e o envia como `Authorization: Bearer <token>` em cada requisição.
4. Servidor verifica a assinatura — sem consulta ao banco de dados.

**Estrutura do JWT:**

```
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
.eyJzdWIiOiJ1c2VyXzQyIiwibmFtZSI6IkFsaWNlIiwicm9sZSI6ImFkbWluIiwiaWF0IjoxNzA5MDAwMDAwLCJleHAiOjE3MDkwMDA5MDB9
.SIGNATURE

Parte 1: Base64url(header)   {"alg":"RS256","typ":"JWT"}
Parte 2: Base64url(payload)  {"sub":"user_42","name":"Alice","role":"admin","iat":1709000000,"exp":1709000900}
Parte 3: Signature           RS256(base64url(header) + "." + base64url(payload), privateKey)
```

> Payloads de JWT são codificados em base64url, NÃO criptografados. Qualquer um pode decodificá-los. Nunca coloque informações sensíveis (senhas, PII, segredos) em claims JWT.

**Claims padrão:**

| Claim | Significado |
|-------|-------------|
| `sub` | Subject — o identificador do usuário |
| `iss` | Issuer — quem emitiu o token |
| `aud` | Audience — para quem o token é destinado |
| `exp` | Expiration time (timestamp Unix) |
| `iat` | Issued at time |
| `jti` | JWT ID — identificador único para este token (permite revogação) |

---

## 3. Como Funciona

### Algoritmos de Assinatura

**HS256 (HMAC-SHA256) — Simétrico:**
- Chave secreta única usada tanto para assinar quanto para verificar.
- Rápido e simples.
- **Problema**: qualquer serviço que precise verificar tokens também precisa conhecer o segredo — e portanto poderia forjar tokens.
- Use quando: um único serviço assina e verifica seus próprios tokens.

**RS256 (RSA-SHA256) — Assimétrico:**
- Chave privada assina o token (mantida apenas pelo serviço de autenticação).
- Chave pública verifica o token (pode ser distribuída livremente).
- Qualquer serviço pode verificar sem poder forjar.
- Ligeiramente mais lento que HS256 devido à criptografia assimétrica.
- Use quando: múltiplos microsserviços precisam verificar tokens, ou tokens são verificados por terceiros.
- Chaves públicas frequentemente expostas como endpoint JWKS (JSON Web Key Set).

```typescript
import { SignJWT, jwtVerify, generateKeyPair, exportSPKI, exportPKCS8 } from 'jose';

// Geração de chave única (armazene a chave privada com segurança no secrets manager)
const { privateKey, publicKey } = await generateKeyPair('RS256');
const publicKeyPEM = await exportSPKI(publicKey);
const privateKeyPEM = await exportPKCS8(privateKey);

// Assina um token
async function signToken(userId: string, role: string): Promise<string> {
  return new SignJWT({ role })
    .setProtectedHeader({ alg: 'RS256' })
    .setSubject(userId)
    .setIssuedAt()
    .setIssuer('https://auth.example.com')
    .setAudience('https://api.example.com')
    .setExpirationTime('15m') // curta duração!
    .sign(privateKey);
}

// Verifica um token — lança exceção se inválido, expirado, audience errado, etc.
async function verifyToken(token: string) {
  const { payload } = await jwtVerify(token, publicKey, {
    issuer: 'https://auth.example.com',
    audience: 'https://api.example.com',
    algorithms: ['RS256'], // NUNCA permita 'none'!
  });
  return payload;
}
```

### O Ataque alg:none

```typescript
// JWT malicioso com alg:none no header:
// Header: {"alg":"none","typ":"JWT"}
// Payload: {"sub":"admin_user","role":"admin","exp":99999999999}
// Signature: (vazia)

// Uma biblioteca que aceita o alg do header do token aceitaria isso!
// Por isso você DEVE especificar explicitamente os algoritmos permitidos:

await jwtVerify(token, publicKey, {
  algorithms: ['RS256'], // rejeita qualquer outro, incluindo 'none'
});
```

> O ataque `alg:none` permite que um invasor forje payloads JWT arbitrários sem assinatura. Sempre fixe o algoritmo esperado no lado da verificação e nunca confie no campo `alg` de um token não confiável.

### Armazenamento de JWT: httpOnly Cookie vs localStorage

```
localStorage:
  ✓ Fácil de acessar via JavaScript
  ✗ Vulnerável a XSS — qualquer script injetado pode lê-lo
  ✗ Nunca use para tokens sensíveis

httpOnly Cookie:
  ✓ Não acessível via JavaScript (document.cookie não o revela)
  ✓ Enviado automaticamente em requisições ao mesmo domínio
  ✗ Vulnerável a CSRF — mitigue com SameSite=Lax/Strict + token CSRF
  ✓ Use este para tokens de autenticação em produção
```

```typescript
// Define JWT em httpOnly cookie (recomendado)
reply
  .setCookie('access_token', jwt, {
    httpOnly: true,
    secure: process.env.NODE_ENV === 'production',
    sameSite: 'lax',
    path: '/',
    maxAge: 15 * 60, // 15 minutos (segundos, não ms)
  });

// Middleware de autenticação: lê do cookie OU do header Authorization
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

### Rotação de Refresh Token

Access tokens de curta duração (15 minutos) + refresh tokens de longa duração resolvem o problema de revogação:

```typescript
// Implementação completa de rotação de refresh token
interface TokenPair {
  accessToken: string;
  refreshToken: string;
}

async function issueTokenPair(userId: string, role: string): Promise<TokenPair> {
  const accessToken = await signToken(userId, role); // validade de 15 min

  // Refresh token: string aleatória opaca (NÃO um JWT — armazenado no banco)
  const refreshToken = crypto.randomBytes(48).toString('base64url');
  const tokenHash = crypto.createHash('sha256').update(refreshToken).digest('hex');

  // Armazena refresh token com hash no banco com validade
  await db.refreshToken.create({
    data: {
      tokenHash,
      userId,
      expiresAt: new Date(Date.now() + 30 * 24 * 60 * 60 * 1000), // 30 dias
      familyId: crypto.randomUUID(), // rastreia família de tokens para detecção de reutilização
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
    // Token não encontrado — pode ser um token roubado/repetido
    return reply.status(401).send({ message: 'Invalid refresh token' });
  }

  if (stored.expiresAt < new Date()) {
    await db.refreshToken.delete({ where: { tokenHash } });
    return reply.status(401).send({ message: 'Refresh token expired' });
  }

  // Rotaciona: deleta o token antigo, emite novo par
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

> Defina o path do cookie de refresh token como `/auth/refresh` — ele só é enviado em requisições para esse endpoint, não em toda requisição de API. Isso limita sua exposição.

---

## 4. OAuth2

OAuth2 é um framework de autorização que permite a um usuário conceder a uma aplicação de terceiro acesso limitado à sua conta sem compartilhar suas credenciais.

### Authorization Code Flow + PKCE

O fluxo recomendado para apps web e mobile. PKCE (Proof Key for Code Exchange) previne ataques de interceptação de código de autorização.

```
1. Cliente gera:
   code_verifier = string aleatória de alta entropia (43–128 chars)
   code_challenge = BASE64URL(SHA256(code_verifier))

2. Cliente redireciona o usuário para o servidor de autorização:
   GET https://auth.example.com/authorize?
     response_type=code
     &client_id=MY_CLIENT_ID
     &redirect_uri=https://myapp.com/callback
     &scope=read:user write:posts
     &state=RANDOM_CSRF_TOKEN     ← previne CSRF
     &code_challenge=ABC123...     ← desafio PKCE
     &code_challenge_method=S256

3. Usuário autentica e concede permissão.

4. Auth server redireciona para redirect_uri:
   GET https://myapp.com/callback?code=AUTH_CODE&state=SAME_CSRF_TOKEN

5. Cliente verifica se o parâmetro state coincide.

6. Cliente troca o código por tokens:
   POST https://auth.example.com/token
   {
     grant_type: 'authorization_code',
     code: AUTH_CODE,
     redirect_uri: 'https://myapp.com/callback',
     client_id: MY_CLIENT_ID,
     code_verifier: ORIGINAL_VERIFIER  ← prova PKCE
   }

7. Auth server verifica: SHA256(code_verifier) == code_challenge
   Retorna: { access_token, refresh_token, id_token, expires_in }
```

```typescript
// Implementação de PKCE em TypeScript
import crypto from 'crypto';

function generatePKCE(): { verifier: string; challenge: string } {
  const verifier = crypto.randomBytes(32).toString('base64url');
  const challenge = crypto
    .createHash('sha256')
    .update(verifier)
    .digest('base64url');
  return { verifier, challenge };
}

app.get('/auth/login', async (req, reply) => {
  const state = crypto.randomBytes(16).toString('hex');
  const { verifier, challenge } = generatePKCE();

  // Armazena na sessão para verificação após redirect
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

  // Verifica state CSRF
  if (state !== req.session.oauthState) {
    return reply.status(400).send({ message: 'State mismatch — possible CSRF' });
  }

  // Troca o código por tokens
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
  // tokens.id_token contém a identidade do usuário (OpenID Connect)
  // tokens.access_token para chamadas de API
});
```

### Client Credentials Flow (Máquina a Máquina)

```typescript
// M2M: serviço A chama serviço B
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

// Armazena o token em cache até expirar (menos um buffer)
let cachedToken: { token: string; expiresAt: number } | null = null;

async function getToken(): Promise<string> {
  const now = Date.now();
  if (cachedToken && cachedToken.expiresAt > now + 60_000) {
    return cachedToken.token;
  }
  const token = await getServiceToken();
  cachedToken = { token, expiresAt: now + 3600_000 };
  return token;
}
```

### OpenID Connect

OpenID Connect (OIDC) é uma camada de identidade fina construída sobre OAuth2. Ela adiciona:
- **ID Token**: um JWT contendo os claims de identidade do usuário (sub, email, name, picture).
- **Endpoint UserInfo**: retorna claims adicionais do usuário.
- **Claims padrão**: `sub`, `email`, `email_verified`, `name`, `given_name`, `family_name`, `picture`, `locale`.

```typescript
import { Issuer, generators } from 'openid-client';

// Descobre automaticamente a configuração do provider
const googleIssuer = await Issuer.discover('https://accounts.google.com');
const client = new googleIssuer.Client({
  client_id: process.env.GOOGLE_CLIENT_ID!,
  client_secret: process.env.GOOGLE_CLIENT_SECRET!,
  redirect_uris: [`${process.env.APP_URL}/auth/callback`],
  response_types: ['code'],
});

app.get('/auth/callback', async (req, reply) => {
  const params = client.callbackParams(req.url);
  const tokenSet = await client.callback(
    `${process.env.APP_URL}/auth/callback`,
    params,
    { code_verifier: req.session.codeVerifier, state: req.session.oauthState }
  );

  const claims = tokenSet.claims(); // claims do ID token
  console.log(claims.sub);         // identificador único do usuário
  console.log(claims.email);       // user@gmail.com
  console.log(claims.name);        // "Alice Smith"

  // Cria ou encontra o usuário no banco
  const user = await upsertUser({
    providerId: claims.sub,
    provider: 'google',
    email: claims.email!,
    name: claims.name!,
  });

  // Emite sua própria sessão/JWT
  const tokens = await issueTokenPair(user.id, user.role);
  // ... define cookies, redireciona para o app
});
```

---

## 5. Erros Comuns e Armadilhas

**Armazenar JWTs em localStorage:**

```typescript
// ERRADO — XSS pode roubar tokens
localStorage.setItem('token', jwt);
const token = localStorage.getItem('token');

// CORRETO — httpOnly cookie, não acessível ao JavaScript
reply.setCookie('access_token', jwt, { httpOnly: true, secure: true });
```

**JWTs de longa duração sem mecanismo de revogação:**

```typescript
// ERRADO — access token de 30 dias sem mecanismo de revogação
const token = jwt.sign({ userId }, secret, { expiresIn: '30d' });
// Um token roubado é válido por 30 dias sem como invalidar

// CORRETO — access token de curta duração + rotação de refresh token
const accessToken = await signToken(userId, role); // 15 min
const refreshToken = await issueRefreshToken(userId); // armazenado no banco, pode ser revogado
```

**Não validar claims do JWT:**

```typescript
// ERRADO — verifica apenas a assinatura, não audience/issuer
jwt.verify(token, publicKey);

// CORRETO — verifica todos os claims
await jwtVerify(token, publicKey, {
  issuer: 'https://auth.example.com',
  audience: 'https://api.example.com',
  algorithms: ['RS256'],
  clockTolerance: 30, // tolerância de 30 segundos para diferença de clock entre servidores
});
```

**Usar o mesmo token para propósitos diferentes:**

```typescript
// ERRADO — mesmo JWT usado tanto para autenticação quanto para verificação de email

// CORRETO — emita tokens específicos por propósito com audiences diferentes
const emailVerifyToken = new SignJWT({ purpose: 'email_verification' })
  .setAudience('https://auth.example.com/verify-email')
  .setExpirationTime('24h')
  .sign(privateKey);
```

---

## 6. Quando Usar / Não Usar

| Abordagem | Use Quando | Evite Quando |
|-----------|------------|--------------|
| Sessões + Redis | App web tradicional, revogação imediata necessária, setup simples | Escala massiva, microsserviços que todos precisam autenticar |
| JWT (access + refresh) | APIs stateless, microsserviços, clientes mobile | Quando revogação imediata é necessária sem compromisso |
| OAuth2 + OIDC | "Login com Google/GitHub", acesso delegado a APIs de terceiros, identidade federada | Apps internos sem identity provider externo |
| API Keys | M2M, APIs para desenvolvedores, ferramentas CLI | Autenticação de usuário (use OAuth2 em vez disso) |

---

## 7. Cenário Real

**Problema:** Uma aplicação SaaS armazena JWTs em localStorage. Uma auditoria de segurança descobriu que um script de analytics de terceiros tem uma vulnerabilidade XSS — todos os tokens de usuário ficam expostos.

**Nova arquitetura:**

```typescript
// Novo fluxo de autenticação: httpOnly cookie + proteção CSRF

app.post('/auth/login', async (req, reply) => {
  const user = await validateCredentials(req.body.email, req.body.password);
  if (!user) return reply.status(401).send({ message: 'Invalid credentials' });

  const { accessToken, refreshToken } = await issueTokenPair(user.id, user.role);

  // Token CSRF: valor aleatório armazenado em cookie E retornado no body da resposta
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
      httpOnly: false, // legível pelo JS para proteção CSRF
      secure: true, sameSite: 'strict',
    })
    .send({ csrfToken, user: { id: user.id, name: user.name } });
});

// Middleware CSRF: verifica se o header X-CSRF-Token coincide com o cookie
app.addHook('preHandler', async (req, reply) => {
  if (['GET', 'HEAD', 'OPTIONS'].includes(req.method)) return;
  const headerToken = req.headers['x-csrf-token'];
  const cookieToken = req.cookies.csrf_token;
  if (!headerToken || headerToken !== cookieToken) {
    return reply.status(403).send({ message: 'CSRF token mismatch' });
  }
});

// JavaScript do cliente: lê o cookie csrf_token e define o header
const response = await fetch('/api/posts', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-CSRF-Token': getCookie('csrf_token'), // lê do cookie não-httpOnly
  },
  body: JSON.stringify({ title: 'Hello' }),
  credentials: 'include', // envia cookies httpOnly
});
```

**Resultado:** Mesmo que exista uma vulnerabilidade XSS, o invasor pode executar scripts mas não consegue ler os cookies `access_token` ou `refresh_token`. O token CSRF previne falsificação cross-site.

---

## 8. Perguntas de Entrevista

**Q1: Quais são os trade-offs entre autenticação baseada em sessão e baseada em JWT?**

Sessões são stateful: o servidor armazena dados de sessão (no Redis para escala horizontal), a revogação é instantânea (delete a sessão). JWTs são stateless: o servidor só precisa de uma chave pública para verificar, escala facilmente entre serviços, mas a revogação requer soluções alternativas (expiração curta + blocklist de refresh tokens, ou manter uma blocklist de tokens o que reintroduz statefulness). Sessões são adequadas para apps web tradicionais; JWTs para APIs e microsserviços onde a verificação stateless entre serviços é valiosa.

**Q2: Por que você deve armazenar JWTs em httpOnly cookies ao invés de localStorage?**

localStorage é acessível a qualquer JavaScript na página — incluindo scripts injetados por vulnerabilidades XSS, SDKs de terceiros comprometidos ou extensões de browser. Um httpOnly cookie não pode ser lido pelo JavaScript (`document.cookie` não o revela), tornando-o imune ao roubo de token via XSS. O trade-off é risco de CSRF, mitigado com `SameSite=Lax/Strict` e um token CSRF.

**Q3: O que é PKCE e por que ele é necessário?**

PKCE (Proof Key for Code Exchange) é uma extensão de segurança para o fluxo OAuth2 Authorization Code para clientes públicos (apps mobile, SPAs) que não conseguem armazenar com segurança um client secret. O cliente gera um `code_verifier` aleatório e envia seu hash SHA-256 (`code_challenge`) para o auth server com a requisição inicial. Ao trocar o código por tokens, envia o `code_verifier` original. Isso prova que a requisição de troca de token vem do mesmo cliente que iniciou o fluxo — prevenindo ataques de interceptação do authorization code.

**Q4: Explique a rotação de refresh token e como ela detecta roubo de token.**

Access tokens de curta duração (15 minutos) + refresh tokens de longa duração (30 dias). Quando o access token expira, o cliente envia o refresh token para obter um novo par. No uso, o refresh token antigo é invalidado e um novo é emitido. Se um invasor roubar um refresh token e usá-lo depois que o cliente legítimo já rotacionou, o servidor detecta uma reutilização. O servidor então deve revogar toda a família de tokens, forçando o usuário a se autenticar novamente. Isso limita a janela de dano de qualquer roubo de token.

**Q5: Qual é a diferença entre HS256 e RS256?**

HS256 usa um único segredo compartilhado tanto para assinar quanto para verificar — simétrico. RS256 usa um par de chaves RSA: a chave privada assina tokens (apenas o emissor a tem), e a chave pública verifica (distribuída livremente). RS256 é preferido em arquiteturas de microsserviços onde múltiplos serviços precisam verificar tokens — eles só precisam da chave pública e não podem forjar tokens. HS256 é mais simples e rápido para setups de serviço único.

**Q6: O que é o ataque alg:none em JWT e como você o previne?**

O header JWT inclui um campo `alg`. Se uma biblioteca confiar neste campo cegamente e o invasor criar um token com `"alg":"none"`, a biblioteca pode pular a verificação de assinatura completamente, aceitando qualquer payload. Prevenção: sempre fixe o algoritmo esperado no lado da verificação e nunca aceite `none`. Bibliotecas como `jose` exigem que você especifique `algorithms: ['RS256']` na chamada de verificação, rejeitando qualquer outro algoritmo.

**Q7: Qual é a diferença entre OAuth2 e OpenID Connect?**

OAuth2 é um framework de autorização para conceder acesso delegado a recursos. Ele responde "a aplicação X pode acessar o recurso Y em nome do usuário Z?" — emite access tokens mas não define padrão para identidade do usuário. OpenID Connect (OIDC) é uma camada de identidade construída sobre OAuth2. Ela adiciona um ID Token (um JWT assinado com claims padrão de identidade do usuário: sub, email, name), um endpoint UserInfo e um mecanismo de discovery. OAuth2 = autorização; OIDC = OAuth2 + autenticação + identidade.

**Q8: Como você revoga um JWT antes de ele expirar?**

JWTs são auto-contidos e stateless — o servidor não pode "alcançar" um token e invalidá-lo. Estratégias: (1) **Expiração curta** — aceite um delay de revogação de 15 minutos usando tokens de duração muito curta com rotação de refresh. (2) **Blocklist de tokens** — mantenha um conjunto de claims `jti` revogados no Redis com TTL igual ao tempo de vida restante do token. Verifique a blocklist em cada requisição. (3) **Campo version** — inclua um `tokenVersion` no JWT; quando o usuário faz logout ou muda a senha, incremente sua versão no banco; tokens com versão antiga são rejeitados (requer uma consulta ao banco por requisição — equivalente a sessões). Abordagem 1 é mais comum; abordagem 2 para logout crítico de segurança; abordagem 3 para "logout em todos os dispositivos."

---

## 9. Exercícios

**Exercício 1: Implemente middleware de autenticação JWT**

Construa um plugin Fastify que:
- Lê o JWT do header `Authorization: Bearer <token>` OU de um httpOnly cookie
- Verifica a assinatura usando RS256 com uma chave pública carregada de uma variável de ambiente
- Define `req.user` com o payload decodificado
- Retorna 401 com erro RFC 7807 para tokens ausentes/inválidos/expirados
- Permite especificar roles necessários: `app.get('/admin', { preHandler: authenticate(['admin']) }, handler)`

*Dica:* Use a biblioteca `jose` para verificação. Carregue a chave pública com `importSPKI()`.

**Exercício 2: Rotação de refresh token**

Implemente o fluxo completo de refresh token:
- `POST /auth/login` → emite access token (15 min) + refresh token (30 dias), define em httpOnly cookies
- `POST /auth/refresh` → rotaciona refresh token, emite novo par
- `POST /auth/logout` → invalida refresh token, limpa cookies
- Detecta reutilização de refresh token: se um token que já foi rotacionado é apresentado, revoga todos os tokens para aquele usuário e retorna 401

*Dica:* Armazene refresh tokens com hash (`sha256`) em uma tabela do banco com `userId`, `familyId`, `tokenHash`, `expiresAt`. Na detecção de reutilização, delete todos os registros que correspondem ao `familyId`.

**Exercício 3: OAuth2 com GitHub**

Construa um fluxo "Login com GitHub":
- Gere a URL de autorização com parâmetro state
- Trate o callback, verifique o state, troque o código por token
- Chame a API `/user` do GitHub para obter informações do usuário
- Faça upsert do usuário no banco (combine por GitHub ID ou email)
- Emita sua própria sessão JWT

*Dica:* GitHub usa OAuth2 padrão sem PKCE (é um cliente confidencial). Use `client_id` + `client_secret` na troca de token.

---

## 10. Leitura Complementar

- [JWT RFC 7519](https://www.rfc-editor.org/rfc/rfc7519)
- [OAuth 2.0 RFC 6749](https://www.rfc-editor.org/rfc/rfc6749)
- [OAuth 2.0 Security Best Practices RFC 9700](https://www.rfc-editor.org/rfc/rfc9700)
- [PKCE RFC 7636](https://www.rfc-editor.org/rfc/rfc7636)
- [Especificação OpenID Connect Core](https://openid.net/specs/openid-connect-core-1_0.html)
- [Biblioteca jose (TypeScript JWT/JWK)](https://github.com/panva/jose)
- [OWASP Authentication Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OWASP JWT Cheatsheet](https://cheatsheetseries.owasp.org/cheatsheets/JSON_Web_Token_for_Java_Cheat_Sheet.html)
- [Blog Auth0 — Refresh Token Rotation](https://auth0.com/docs/secure/tokens/refresh-tokens/refresh-token-rotation)
- Livro: *OAuth 2 in Action* por Justin Richer & Antonio Sanso
