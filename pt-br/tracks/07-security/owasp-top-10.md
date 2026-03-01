# OWASP Top 10

## Visão Geral

O OWASP Top 10 é um documento de conscientização publicado pelo Open Web Application Security Project. Representa os dez riscos de segurança mais críticos em aplicações web, compilados a partir de dados reais de vulnerabilidades de milhares de organizações. Entender esses riscos — e saber como mitigá-los em um stack Node.js/TypeScript — é o ponto de partida para construir software seguro.

Este capítulo cobre cada categoria com um exemplo concreto de como ela se manifesta em uma aplicação típica Fastify/Express e como eliminá-la.

---

## Pré-requisitos

- Fundamentos de HTTP (requisições, respostas, headers, cookies)
- Node.js e TypeScript básicos
- Familiaridade com bancos de dados SQL e REST APIs

---

## Conceitos Fundamentais

As categorias do OWASP Top 10 (2021) são:

| # | Categoria | Causa Raiz Comum |
|---|----------|------------------|
| A01 | Broken Access Control | Verificações de autorização ausentes |
| A02 | Cryptographic Failures | Criptografia fraca, armazenamento em texto plano |
| A03 | Injection | Input do usuário não sanitizado em queries/comandos |
| A04 | Insecure Design | Sem modelagem de ameaças, requisitos de segurança ausentes |
| A05 | Security Misconfiguration | Configurações padrão, erros verbosos, portas abertas |
| A06 | Vulnerable and Outdated Components | Dependências sem patches de segurança |
| A07 | Identification and Authentication Failures | Senhas fracas, sem MFA, gerenciamento de sessão quebrado |
| A08 | Software and Data Integrity Failures | Atualizações não verificadas, deserialização |
| A09 | Security Logging and Monitoring Failures | Sem logs de auditoria, brechas não detectadas |
| A10 | Server-Side Request Forgery (SSRF) | Aceitar URLs controladas pelo usuário para requisições do servidor |

---

## Exemplos Práticos

### A01 — Broken Access Control

A categoria mais comum. Ocorre quando usuários conseguem acessar recursos ou executar ações que não deveriam ser permitidas.

**Padrão vulnerável:**

```typescript
// GET /api/invoices/:id — sem verificação de propriedade
fastify.get('/api/invoices/:id', async (request, reply) => {
  const { id } = request.params as { id: string };
  // Qualquer usuário autenticado pode ler qualquer fatura adivinhando o ID
  const invoice = await db.invoice.findUnique({ where: { id } });
  return invoice;
});
```

**Padrão corrigido — sempre impor propriedade:**

```typescript
fastify.get('/api/invoices/:id', { preHandler: [requireAuth] }, async (request, reply) => {
  const { id } = request.params as { id: string };
  const userId = request.user.id;

  const invoice = await db.invoice.findUnique({
    where: { id, userId }, // restrito ao usuário autenticado
  });

  if (!invoice) {
    return reply.status(404).send({ error: 'Invoice not found' });
  }

  return invoice;
});
```

Regra: nunca confie no cliente para dizer o que ele possui. Sempre escreva queries com escopo pela identidade autenticada.

---

### A02 — Cryptographic Failures

Armazenar dados sensíveis em texto plano ou usar hashing fraco.

**Vulnerável — MD5 para senhas:**

```typescript
import crypto from 'crypto';

// NUNCA faça isso — MD5 é quebrado e não tem salt
const hash = crypto.createHash('md5').update(password).digest('hex');
```

**Corrigido — bcrypt com fator de trabalho:**

```typescript
import bcrypt from 'bcrypt';

const SALT_ROUNDS = 12; // maior = mais lento = mais resistente a força bruta

export async function hashPassword(password: string): Promise<string> {
  return bcrypt.hash(password, SALT_ROUNDS);
}

export async function verifyPassword(password: string, hash: string): Promise<boolean> {
  return bcrypt.compare(password, hash);
}
```

Também criptografe dados em repouso quando forem sensíveis (PII, dados financeiros) e utilize HTTPS para proteger dados em trânsito.

---

### A03 — Injection

O clássico SQL injection, mas também NoSQL injection, command injection e LDAP injection.

**Vulnerável — interpolação de string em query SQL:**

```typescript
// Se userId = "1 OR 1=1", isso retorna todos os usuários
const result = await db.query(`SELECT * FROM users WHERE id = ${userId}`);
```

**Corrigido — queries parametrizadas:**

```typescript
// Prisma parametriza automaticamente
const user = await db.user.findUnique({ where: { id: userId } });

// SQL raw quando necessário — use tagged template ou $queryRaw com Prisma
const users = await db.$queryRaw`SELECT * FROM users WHERE id = ${userId}`;
```

**Valide o input na fronteira com Zod:**

```typescript
import { z } from 'zod';

const ParamsSchema = z.object({
  id: z.string().uuid(), // rejeita qualquer coisa que não seja um UUID
});

fastify.get('/users/:id', async (request, reply) => {
  const { id } = ParamsSchema.parse(request.params);
  // id é garantidamente um UUID — sem espaço para injection
  return db.user.findUnique({ where: { id } });
});
```

---

### A04 — Insecure Design

Os requisitos de segurança nunca foram definidos. A arquitetura não tem rate limiting, sem controles contra fraude, sem separação de operações sensíveis.

**Checklist de mitigação durante o design:**

- Modele ameaças para cada endpoint de API: quem o chama, o que pode fazer, qual é o pior caso se for abusado?
- Aplique rate limiting em endpoints de autenticação
- Exija autenticação step-up para ações sensíveis (troca de senha, pagamento, exportação)
- Registre todas as operações sensíveis em um log de auditoria

```typescript
import rateLimit from '@fastify/rate-limit';

// Rate limiting rigoroso em endpoints de autenticação
await fastify.register(rateLimit, {
  max: 5,
  timeWindow: '1 minute',
  keyGenerator: (request) => request.ip,
  errorResponseBuilder: () => ({
    error: 'Muitas tentativas de login. Tente novamente em 1 minuto.',
  }),
});

fastify.post('/auth/login', async (request, reply) => {
  // ...
});
```

---

### A05 — Security Misconfiguration

Credenciais padrão, listagem de diretórios habilitada, mensagens de erro verbosas expondo stack traces.

**Vulnerável — stack trace na resposta em produção:**

```typescript
fastify.setErrorHandler((error, request, reply) => {
  // Envia o stack trace completo ao cliente — expõe internos
  reply.status(500).send({ error: error.message, stack: error.stack });
});
```

**Corrigido — sanitize as respostas de erro em produção:**

```typescript
fastify.setErrorHandler((error, request, reply) => {
  const isProd = process.env.NODE_ENV === 'production';

  fastify.log.error({ err: error, requestId: request.id }, 'Erro não tratado');

  reply.status(error.statusCode ?? 500).send({
    error: isProd ? 'Internal Server Error' : error.message,
    requestId: request.id, // útil para correlacionar logs sem expor internos
  });
});
```

---

### A06 — Vulnerable and Outdated Components

Usar dependências com CVEs conhecidos.

```bash
# Execute no CI a cada pull request
npm audit --audit-level=high

# Ou use uma ferramenta dedicada
npx snyk test
```

Veja o capítulo [Auditoria de Dependências](dependency-auditing.md) para um guia completo de integração com CI.

---

### A07 — Identification and Authentication Failures

Tokens de sessão fracos, sem bloqueio de conta, senhas armazenadas sem hashing.

Mitigações principais:
- Use JWTs de curta duração (15–60 minutos) com refresh tokens
- Implemente bloqueio de conta após N tentativas falhas
- Imponha políticas de senhas fortes via Zod
- Faça hash de senhas com bcrypt (veja A02)

```typescript
const PasswordSchema = z.string()
  .min(12)
  .regex(/[A-Z]/, 'Deve conter letra maiúscula')
  .regex(/[0-9]/, 'Deve conter um número')
  .regex(/[^A-Za-z0-9]/, 'Deve conter um caractere especial');
```

---

### A08 — Software and Data Integrity Failures

Deserialização não confiável, atualizações sem assinatura, pipelines de CI/CD sem verificações de integridade de artefatos.

No Node.js, isso se manifesta como:
- Deserializar JSON não confiável com `eval()` ou `new Function()`
- Aceitar objetos serializados e chamar métodos neles sem validação
- Usar `npm install` sem lockfile (permite ataques de substituição)

Sempre faça commit do `package-lock.json` ou `bun.lockb` no controle de versão. Fixe versões exatas de dependências em sistemas críticos.

---

### A09 — Security Logging and Monitoring Failures

Sem logs, brechas passam despercebidas por meses.

```typescript
import pino from 'pino';

const logger = pino({
  level: 'info',
  redact: ['password', 'token', 'authorization', 'cookie'], // nunca logue secrets
});

// Registre eventos de autenticação
async function loginUser(email: string, password: string) {
  const user = await db.user.findUnique({ where: { email } });

  if (!user || !(await verifyPassword(password, user.passwordHash))) {
    logger.warn({ email }, 'Tentativa de login falhou');
    throw new Error('Invalid credentials');
  }

  logger.info({ userId: user.id }, 'Usuário logado');
  return createSession(user);
}
```

---

### A10 — Server-Side Request Forgery (SSRF)

O servidor busca uma URL fornecida pelo usuário, permitindo que atacantes alcancem serviços internos.

**Vulnerável:**

```typescript
fastify.post('/fetch-url', async (request, reply) => {
  const { url } = request.body as { url: string };
  // Atacante envia url = "http://169.254.169.254/latest/meta-data/" (metadata AWS)
  const response = await fetch(url);
  return response.text();
});
```

**Corrigido — abordagem com allowlist:**

```typescript
import { URL } from 'url';

const ALLOWED_HOSTS = new Set(['api.example.com', 'cdn.example.com']);

fastify.post('/fetch-url', async (request, reply) => {
  const { url } = request.body as { url: string };

  let parsed: URL;
  try {
    parsed = new URL(url);
  } catch {
    return reply.status(400).send({ error: 'URL inválida' });
  }

  if (!ALLOWED_HOSTS.has(parsed.hostname)) {
    return reply.status(403).send({ error: 'URL não permitida' });
  }

  const response = await fetch(parsed.toString());
  return response.text();
});
```

---

## Padrões Comuns e Boas Práticas

- **Valide todo input na fronteira** — use schemas Zod em todas as rotas Fastify
- **Escreva todas as queries com escopo pela identidade autenticada** — nunca confie em IDs do corpo da requisição
- **Aplique defesa em profundidade** — múltiplas camadas (validação + parametrização por ORM + verificação de autorização)
- **Use Helmet.js** — configura security headers com um único import
- **Execute `npm audit` no CI** — quebre o build em vulnerabilidades de severidade alta
- **Registre eventos de autenticação e autorização** — não as credenciais em si
- **Use variáveis de ambiente para secrets** — nunca hardcode API keys ou senhas

---

## Anti-Padrões a Evitar

- Criar sua própria criptografia — use bcrypt, argon2 ou libsodium
- Confiar no `Content-Type` sem validação
- Usar `eval()` ou `new Function()` em input do usuário
- Retornar stack traces para clientes de API em produção
- Logar senhas, tokens ou PII
- Fazer commit de secrets no controle de versão (mesmo em repos privados)
- Ignorar avisos do `npm audit` porque "vamos corrigir depois"

---

## Debugging e Resolução de Problemas

**"Minha validação Zod não está capturando payloads de injection"**
Verifique se você está chamando `.parse()` (lança exceção em input inválido) e não `.safeParse()` sem tratar o caso de erro. Certifique-se também de que o schema é restrito o suficiente — `z.string()` sozinho aceita fragmentos SQL; você precisa de `z.string().uuid()` ou restrições similares.

**"npm audit reporta vulnerabilidades em dependências transitivas"**
Use `npm audit fix` para problemas corrigíveis automaticamente. Para casos mais complexos, use `overrides` no `package.json` para fixar uma versão segura da dependência transitiva, ou abra uma issue na dependência direta pedindo que ela seja atualizada.

**"Rate limiting bloqueia usuários legítimos"**
Use uma chave mais inteligente que o IP em rotas autenticadas — use `userId` em vez do IP. Aplique limites rígidos apenas em endpoints não autenticados.

---

## Cenários do Mundo Real

**Cenário 1: API SaaS multi-tenant**

Cada recurso (projetos, documentos, faturas) pertence a uma organização. O padrão:

```typescript
// middleware/require-org-access.ts
export async function requireOrgAccess(request: FastifyRequest, reply: FastifyReply) {
  const { orgId } = request.params as { orgId: string };
  const userId = request.user.id;

  const membership = await db.orgMember.findUnique({
    where: { orgId_userId: { orgId, userId } },
  });

  if (!membership) {
    return reply.status(403).send({ error: 'Acesso negado' });
  }

  request.org = { id: orgId, role: membership.role };
}
```

Aplique este middleware em todas as rotas com escopo de organização. Então escreva todas as queries com escopo: `db.project.findMany({ where: { orgId } })`.

**Cenário 2: API pública com rate limiting por tier**

```typescript
fastify.addHook('preHandler', async (request, reply) => {
  const limit = request.user?.tier === 'pro' ? 1000 : 100;
  // Verifique o rate limit no Redis usando user.id como chave
});
```

---

## Leitura Complementar

- [OWASP Top 10 (2021)](https://owasp.org/Top10/)
- [OWASP Cheat Sheet Series](https://cheatsheetseries.owasp.org/)
- [NodeJS Security Handbook — Snyk](https://snyk.io/learn/nodejs-security-best-practice/)
- Track 07: [Autenticação & Autorização](authentication-authorization.md)
- Track 07: [Security Headers](security-headers.md)

---

## Resumo

O OWASP Top 10 não é uma lista de verificação a ser concluída uma vez — é uma mudança de mentalidade. A vulnerabilidade mais comum (Broken Access Control) é corrigida por um único hábito: sempre escreva queries com escopo pela identidade autenticada. Injection (A03) é corrigida nunca interpolando input do usuário em queries ou comandos shell. Falhas criptográficas (A02) são corrigidas usando bcrypt para senhas e HTTPS para transporte. O restante do top 10 se enquadra em dois grupos: higiene de configuração e disciplina operacional (logs, monitoramento, gerenciamento de dependências). Abordar todas as dez categorias é viável em uma aplicação Node.js típica sem nenhuma ferramenta de segurança especializada — apenas aplicação consistente desses padrões.
