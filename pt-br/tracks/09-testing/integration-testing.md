# Testes de Integração

## Visão Geral

Testes de integração verificam que múltiplos componentes funcionam corretamente juntos. Onde os testes unitários isolam cada peça, os testes de integração as conectam: o route handler conversa com o serviço real, o serviço conversa com o banco de dados real. Esses testes capturam uma categoria inteira de bugs que os testes unitários perdem: queries SQL mal configuradas, schemas do Prisma quebrados, códigos de status HTTP errados, middleware faltando. Este capítulo cobre testes de integração em um contexto Fastify/TypeScript/SQLite.

---

## Pré-requisitos

- Track 09: Fundamentos de Testes e Testes Unitários
- Conceitos básicos de Fastify (rotas, plugins, hooks)
- Fundamentos de Prisma e SQL

---

## Exemplos Práticos

### Testes de integração HTTP com o cliente de teste do Fastify

```bash
npm install --save-dev @types/supertest
```

```typescript
// src/app.ts — exporta a instância do Fastify para testes
import Fastify from 'fastify';
import { userRoutes } from './routes/users.js';
import { authMiddleware } from './plugins/auth.js';

export function buildApp() {
  const app = Fastify({ logger: false }); // desativa logger nos testes

  app.register(authMiddleware);
  app.register(userRoutes, { prefix: '/api/users' });

  return app;
}
```

```typescript
// src/routes/users.test.ts
import { describe, it, expect, beforeAll, afterAll, beforeEach } from 'vitest';
import { buildApp } from '../app.js';
import { db } from '../lib/db.js';

const app = buildApp();

beforeAll(async () => {
  await app.ready();
  // Executa migrations no banco de dados de teste
  await db.$executeRaw`DELETE FROM users`;
});

afterAll(async () => {
  await app.close();
  await db.$disconnect();
});

describe('GET /api/users/:id', () => {
  it('returns the user when found', async () => {
    // Insere um usuário no banco de dados de teste
    const user = await db.user.create({
      data: { email: 'alice@test.com', name: 'Alice', passwordHash: 'hash' },
    });

    const response = await app.inject({
      method: 'GET',
      url: `/api/users/${user.id}`,
      headers: { Authorization: `Bearer ${generateTestToken(user.id)}` },
    });

    expect(response.statusCode).toBe(200);

    const body = response.json();
    expect(body.id).toBe(user.id);
    expect(body.email).toBe('alice@test.com');
    expect(body.passwordHash).toBeUndefined(); // campo sensível removido
  });

  it('returns 404 for a non-existent user', async () => {
    const response = await app.inject({
      method: 'GET',
      url: '/api/users/non-existent-id',
      headers: { Authorization: `Bearer ${generateTestToken('any-id')}` },
    });

    expect(response.statusCode).toBe(404);
  });

  it('returns 401 without authentication', async () => {
    const response = await app.inject({
      method: 'GET',
      url: '/api/users/any-id',
    });

    expect(response.statusCode).toBe(401);
  });
});

describe('POST /api/users', () => {
  it('creates a new user', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      payload: {
        email: 'bob@test.com',
        name: 'Bob',
        password: 'SecurePassword123!',
      },
    });

    expect(response.statusCode).toBe(201);

    const body = response.json();
    expect(body.email).toBe('bob@test.com');
    expect(body.id).toBeDefined();

    // Verificar se foi realmente persistido
    const saved = await db.user.findUnique({ where: { id: body.id } });
    expect(saved).not.toBeNull();
    expect(saved?.email).toBe('bob@test.com');
  });

  it('returns 400 for invalid email', async () => {
    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      payload: { email: 'not-an-email', name: 'Bob', password: 'SecurePass123!' },
    });

    expect(response.statusCode).toBe(400);
  });

  it('returns 409 for duplicate email', async () => {
    await db.user.create({ data: { email: 'dup@test.com', name: 'Dup', passwordHash: 'x' } });

    const response = await app.inject({
      method: 'POST',
      url: '/api/users',
      payload: { email: 'dup@test.com', name: 'Another', password: 'Pass123!' },
    });

    expect(response.statusCode).toBe(409);
  });
});
```

### Configuração do banco de dados de teste (SQLite para testes de integração rápidos)

Use SQLite nos testes para evitar precisar de uma instância real do Postgres:

```typescript
// vitest.config.ts
export default defineConfig({
  test: {
    globals: true,
    environment: 'node',
    setupFiles: ['./src/test/setup.ts'],
  },
});
```

```typescript
// src/test/setup.ts
import { execSync } from 'child_process';

// Executa migrations do Prisma contra o banco SQLite de teste
process.env.DATABASE_URL = 'file:./test.db';

execSync('npx prisma migrate reset --force --skip-seed', {
  env: { ...process.env, DATABASE_URL: 'file:./test.db' },
});
```

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql" // ou "sqlite" para testes
  url      = env("DATABASE_URL")
}
```

### Testes de integração de repositório com banco de dados

```typescript
// src/repositories/user.repository.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { UserRepository } from './user.repository.js';
import { db } from '../lib/db.js';

const repo = new UserRepository(db);

beforeEach(async () => {
  await db.user.deleteMany(); // estado limpo antes de cada teste
});

describe('UserRepository', () => {
  describe('findByEmail', () => {
    it('returns the user when email exists', async () => {
      await db.user.create({
        data: { email: 'test@example.com', name: 'Test', passwordHash: 'hash' },
      });

      const user = await repo.findByEmail('test@example.com');
      expect(user?.email).toBe('test@example.com');
    });

    it('returns null when email does not exist', async () => {
      const user = await repo.findByEmail('nobody@example.com');
      expect(user).toBeNull();
    });
  });

  describe('create', () => {
    it('persists a user with all required fields', async () => {
      const user = await repo.create({
        email: 'new@example.com',
        name: 'New User',
        passwordHash: 'bcrypt-hash',
      });

      expect(user.id).toBeDefined();
      expect(user.createdAt).toBeInstanceOf(Date);

      const fromDb = await db.user.findUnique({ where: { id: user.id } });
      expect(fromDb).not.toBeNull();
    });

    it('throws on duplicate email', async () => {
      await repo.create({ email: 'dup@example.com', name: 'First', passwordHash: 'h' });

      await expect(
        repo.create({ email: 'dup@example.com', name: 'Second', passwordHash: 'h' })
      ).rejects.toThrow(); // violação de unique constraint
    });
  });
});
```

### Helpers e factories de teste

Evite repetição na configuração de dados de teste com funções factory:

```typescript
// src/test/factories.ts
import { db } from '../lib/db.js';
import { hashPassword } from '../lib/password.js';

interface UserOverrides {
  email?: string;
  name?: string;
  role?: string;
  password?: string;
}

export async function createTestUser(overrides: UserOverrides = {}) {
  const password = overrides.password ?? 'TestPassword123!';
  return db.user.create({
    data: {
      email: overrides.email ?? `user-${Date.now()}@test.com`,
      name: overrides.name ?? 'Test User',
      role: overrides.role ?? 'user',
      passwordHash: await hashPassword(password),
    },
  });
}

export function generateTestToken(userId: string, role = 'user'): string {
  return jwt.sign({ sub: userId, role }, process.env.JWT_ACCESS_SECRET!, { expiresIn: '15m' });
}
```

```typescript
// Uso nos testes
const adminUser = await createTestUser({ role: 'admin' });
const token = generateTestToken(adminUser.id, 'admin');

const response = await app.inject({
  method: 'DELETE',
  url: `/api/users/${targetUser.id}`,
  headers: { Authorization: `Bearer ${token}` },
});
```

---

## Padrões Comuns e Boas Práticas

- **Limpe o banco antes de cada teste** (`deleteMany` no `beforeEach`) — evita poluição entre testes
- **Teste o ciclo HTTP completo** — código de status, corpo da resposta, estado do banco após escrita
- **Use `app.inject()`** (Fastify) — sem chamada de rede real, mas exercita toda a pilha de middleware
- **Teste casos extremos** — listas vazias, campos faltando, acesso não autorizado, registros duplicados
- **Mantenha os dados de teste mínimos** — crie apenas o que o teste precisa; sem fixtures enormes compartilhadas

---

## Anti-Padrões a Evitar

- Compartilhar estado entre testes (usuário criado no teste A usado no teste B) — testes quebram dependendo da ordem de execução
- Testar contra um banco de desenvolvimento compartilhado — escritas dos testes poluem os dados de desenvolvimento
- Pular a verificação no banco de dados — se o teste verifica apenas a resposta HTTP, ele perde persistência quebrada
- Fixtures enormes no `beforeAll` com 50 usuários — difícil entender quais dados cada teste usa

---

## Depuração e Resolução de Problemas

**"Foreign key constraint falha nos testes"**
Delete na ordem correta — registros filhos antes dos registros pai. Ou use `db.$executeRaw'PRAGMA foreign_keys = OFF'` (SQLite) para ignorar verificações de FK durante a limpeza.

**"Testes passam individualmente mas falham em paralelo"**
Os testes compartilham o banco e estão mutando as mesmas linhas. Adicione `pool: { maxWorkers: 1 }` na configuração do vitest para rodar testes de integração sequencialmente, ou use rollback de transação em vez de `deleteMany`.

**"app.inject retorna 500 sem mensagem útil"**
Ative o logging temporariamente: `const app = Fastify({ logger: true })` e verifique a saída do console. Ou adicione um error handler global que registra o erro completo.

---

## Leitura Complementar

- [Guia de testes do Fastify](https://fastify.dev/docs/latest/Guides/Testing/)
- [Guia de testes do Prisma](https://www.prisma.io/docs/guides/testing/unit-testing)
- Track 09: [Fundamentos de Testes](testing-fundamentals.md)
- Track 09: [Testes E2E](e2e-testing.md)

---

## Resumo

Testes de integração preenchem a lacuna entre testes unitários e testes E2E. Eles verificam que seus route handlers, serviços e repositórios de banco de dados funcionam corretamente juntos — capturando bugs que testes unitários isolados perdem. Use `app.inject()` para testes de integração HTTP e um banco SQLite de teste rápido (ou testes com transações encapsuladas) para testes de repositório. Limpe o banco antes de cada teste, teste tanto caminhos de sucesso quanto de falha, e verifique que as escritas foram realmente persistidas no banco. O esforço vale a pena: testes de integração capturam a categoria de bugs que causam a maioria dos incidentes em produção.
