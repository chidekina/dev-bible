# Autenticação & Autorização

## Visão Geral

Autenticação responde "quem é você?" Autorização responde "o que você pode fazer?" São problemas distintos que frequentemente se confundem, e errar em qualquer um deles gera vulnerabilidades de segurança. Este capítulo cobre o espectro completo: hashing de senhas, ciclo de vida de JWTs, gerenciamento de sessão, OAuth 2.0 e modelos de controle de acesso (RBAC e ABAC) — tudo implementado em um contexto Node.js/TypeScript/Fastify.

---

## Pré-requisitos

- Cookies HTTP, headers e o ciclo de requisição/resposta
- TypeScript básico e async/await
- Familiaridade com middleware Fastify (hooks preHandler)

---

## Conceitos Fundamentais

### Mecanismos de autenticação

| Mecanismo | Caso de uso | Risco |
|-----------|-------------|-------|
| Usuário + senha | A maioria das aplicações web | Credential stuffing, senhas fracas |
| JWT (stateless) | Microsserviços, SPAs | Roubo de token, sem revogação instantânea |
| Cookies de sessão (stateful) | Aplicações web tradicionais | Sequestro de sessão |
| OAuth 2.0 (delegado) | "Login com Google/GitHub" | URIs de redirecionamento mal configuradas |
| API keys | Comunicação máquina a máquina | Vazamento de chave, sem expiração |

### Modelos de autorização

**RBAC (Role-Based Access Control):** Usuários têm roles, roles têm permissões. Fácil de compreender e raciocinar.

**ABAC (Attribute-Based Access Control):** Decisões de acesso usam atributos do sujeito, recurso, ação e ambiente. Mais expressivo, mais complexo.

---

## Exemplos Práticos

### Hashing de senhas com bcrypt

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
// Cadastro
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

Nunca armazene senhas em texto plano. Nunca use MD5 ou SHA-1. bcrypt (ou argon2id) com fator de trabalho 12+ é o padrão atual.

---

### Autenticação stateless baseada em JWT

```typescript
// src/lib/token.ts
import jwt from 'jsonwebtoken';
import { z } from 'zod';

const ACCESS_SECRET = process.env.JWT_ACCESS_SECRET!;
const REFRESH_SECRET = process.env.JWT_REFRESH_SECRET!;

const AccessPayloadSchema = z.object({
  sub: z.string(),   // ID do usuário
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
// src/plugins/auth.ts — plugin Fastify
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
    if (!authHeader?.startsWith('Bearer ')) return; // permite que rotas não autenticadas prossigam

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
// Exigir autenticação em uma rota
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

### Rotação de refresh tokens

Refresh tokens permitem emitir access tokens de curta duração sem forçar o usuário a fazer login frequentemente. Armazene refresh tokens no banco de dados para que possam ser revogados.

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

  // Verifica se o token existe no DB e não foi usado (rotação)
  const stored = await db.refreshToken.findUnique({
    where: { token: refreshToken },
  });

  if (!stored || stored.used || stored.expiresAt < new Date()) {
    // Possível reutilização de token — revoga todos os tokens deste usuário
    await db.refreshToken.updateMany({
      where: { userId: payload.sub },
      data: { used: true },
    });
    return reply.status(401).send({ error: 'Refresh token invalid or reused' });
  }

  // Marca o token antigo como usado (rotação)
  await db.refreshToken.update({
    where: { token: refreshToken },
    data: { used: true },
  });

  const user = await db.user.findUnique({ where: { id: payload.sub } });
  if (!user) return reply.status(401).send({ error: 'User not found' });

  // Emite novo par de tokens
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
// Uso
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

ABAC é útil quando o acesso depende do relacionamento entre o usuário e o recurso, e não apenas do role do usuário.

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
  // Admins podem fazer tudo
  if (userRole === 'admin') return true;

  // O dono pode fazer qualquer coisa com seu próprio recurso
  if (resource.ownerId === userId) return true;

  // Recursos públicos podem ser lidos por qualquer pessoa
  if (action === 'read' && resource.visibility === 'public') return true;

  // Membros da organização podem ler recursos da organização
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
// Uso em uma rota
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

### OAuth 2.0 — Login com GitHub

```typescript
// src/routes/auth.ts
import { Octokit } from 'octokit';

fastify.get('/auth/github', async (request, reply) => {
  const state = crypto.randomBytes(16).toString('hex');
  // Armazena o state na sessão ou em cookie de curta duração para prevenir CSRF
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

  // Troca o code pelo token
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

  // Busca informações do usuário
  const octokit = new Octokit({ auth: access_token });
  const { data: githubUser } = await octokit.rest.users.getAuthenticated();

  // Upsert do usuário no nosso banco
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

## Padrões Comuns e Boas Práticas

- **Access tokens de 15 minutos, refresh tokens de 7 dias** — minimize o raio de impacto de um roubo de token
- **Rotacione refresh tokens a cada uso** — detecte ataques de reutilização
- **Armazene refresh tokens no banco de dados** — permite revogação instantânea
- **Use cookies `httpOnly` para refresh tokens em aplicações browser** — previne roubo via XSS
- **Nunca coloque dados sensíveis em payloads JWT** — JWTs são codificados em base64, não criptografados
- **Valide o parâmetro `state` do OAuth** — previne ataques CSRF no fluxo OAuth
- **Use comparação de tempo constante para validação de tokens** — previne ataques de timing

---

## Anti-Padrões a Evitar

- Gerar tokens com `Math.random()` — use `crypto.randomBytes()`
- Armazenar senhas de usuário em texto plano ou com MD5/SHA-1
- Access tokens de longa duração (horas ou dias) sem rotação de refresh
- Verificar roles apenas no cliente — sempre imponha no servidor
- Confiar no `userId` do corpo da requisição em vez de extraí-lo do token
- Usar o mesmo secret para access e refresh tokens
- Não registrar eventos de autenticação (login, logout, tentativas falhas, troca de senha)

---

## Debugging e Resolução de Problemas

**Erros "jwt expired" imediatamente após emitir um token**
Verifique se o relógio do servidor está preciso. Dessincronia de relógio entre serviços causa falhas de validação de JWT. Use a opção `leeway` na biblioteca JWT para permitir uma pequena variação.

**Refresh token "invalid or reused" após um único uso**
Verifique se você não está fazendo múltiplas requisições paralelas com o mesmo refresh token. No lado do cliente, enfileire as chamadas de renovação de token para prevenir race conditions.

**Redirect URI mismatch no OAuth**
O redirect URI na sua URL de autorização deve corresponder exatamente ao que está registrado nas configurações da aplicação no provedor OAuth — incluindo protocolo, domínio, porta e caminho.

---

## Cenários do Mundo Real

**Cenário: Proteção de rotas exclusivas para admins**

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

Registrar o hook no nível do plugin significa que todas as rotas dentro do plugin exigem autenticação + role admin, sem repetir o `preHandler` em cada rota.

---

## Leitura Complementar

- [JWT.io Introduction](https://jwt.io/introduction)
- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [OAuth 2.0 Security Best Practices (RFC 9700)](https://datatracker.ietf.org/doc/html/rfc9700)
- Track 07: [CORS & CSRF](cors-csrf.md)
- Track 07: [Gerenciamento de Secrets](secrets-management.md)

---

## Resumo

Autenticação e autorização são preocupações distintas que devem ser implementadas corretamente. Autenticação (provar identidade) se baseia em hashing de senhas com bcrypt, JWTs de curta duração e rotação de refresh tokens. Autorização (impor permissões) se baseia em escrever todas as queries com escopo pelo usuário autenticado e implementar verificações RBAC ou ABAC como middleware no servidor. O erro mais perigoso em ambas as áreas é confiar no cliente: sempre extraia a identidade de um token verificado, sempre verifique permissões no servidor e sempre escreva queries com escopo por essa identidade verificada.
