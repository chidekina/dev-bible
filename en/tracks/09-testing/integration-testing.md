# Integration Testing

## Overview

Integration tests verify that multiple components work correctly together. Where unit tests isolate each piece, integration tests connect them: the route handler talks to the real service, the service talks to the real database. These tests catch a whole class of bugs that unit tests miss: misconfigured SQL queries, broken Prisma schemas, wrong HTTP status codes, missing middleware. This chapter covers integration testing in a Fastify/TypeScript/SQLite context.

---

## Prerequisites

- Track 09: Testing Fundamentals and Unit Testing
- Fastify basics (routes, plugins, hooks)
- Prisma and SQL fundamentals

---

## Hands-On Examples

### HTTP integration tests with Fastify's test client

```bash
npm install --save-dev @types/supertest
```

```typescript
// src/app.ts — export the Fastify instance for testing
import Fastify from 'fastify';
import { userRoutes } from './routes/users.js';
import { authMiddleware } from './plugins/auth.js';

export function buildApp() {
  const app = Fastify({ logger: false }); // disable logger in tests

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
  // Run migrations on test database
  await db.$executeRaw`DELETE FROM users`;
});

afterAll(async () => {
  await app.close();
  await db.$disconnect();
});

describe('GET /api/users/:id', () => {
  it('returns the user when found', async () => {
    // Seed a user in the test database
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
    expect(body.passwordHash).toBeUndefined(); // sensitive field stripped
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

    // Verify it was actually persisted
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

### Test database setup (SQLite for fast integration tests)

Use SQLite in tests to avoid needing a real Postgres instance:

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

// Run Prisma migrations against the test SQLite database
process.env.DATABASE_URL = 'file:./test.db';

execSync('npx prisma migrate reset --force --skip-seed', {
  env: { ...process.env, DATABASE_URL: 'file:./test.db' },
});
```

```prisma
// prisma/schema.prisma
datasource db {
  provider = "postgresql" // or "sqlite" for tests
  url      = env("DATABASE_URL")
}
```

### Database repository integration tests

```typescript
// src/repositories/user.repository.test.ts
import { describe, it, expect, beforeEach } from 'vitest';
import { UserRepository } from './user.repository.js';
import { db } from '../lib/db.js';

const repo = new UserRepository(db);

beforeEach(async () => {
  await db.user.deleteMany(); // clean slate before each test
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
      ).rejects.toThrow(); // unique constraint violation
    });
  });
});
```

### Test helpers and factories

Avoid repetitive test data setup with factory functions:

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
// Usage in tests
const adminUser = await createTestUser({ role: 'admin' });
const token = generateTestToken(adminUser.id, 'admin');

const response = await app.inject({
  method: 'DELETE',
  url: `/api/users/${targetUser.id}`,
  headers: { Authorization: `Bearer ${token}` },
});
```

---

## Common Patterns & Best Practices

- **Clean the database before each test** (`deleteMany` in `beforeEach`) — prevents test pollution
- **Test the full HTTP cycle** — status code, response body, database state after write
- **Use `app.inject()`** (Fastify) — no real network call, but exercises the full middleware stack
- **Test edge cases** — empty lists, missing fields, unauthorized access, duplicate records
- **Keep test data minimal** — create only what the test needs; no massive shared fixtures

---

## Anti-Patterns to Avoid

- Sharing state between tests (user created in test A used in test B) — tests break depending on run order
- Testing against a shared dev database — writes from tests pollute development data
- Skipping the database assertion — if the test only checks the HTTP response, it misses broken persistence
- Huge `beforeAll` fixtures with 50 users — hard to understand which data each test uses

---

## Debugging & Troubleshooting

**"Foreign key constraint fails in tests"**
Delete in the right order — child records before parent records. Or use `db.$executeRaw'PRAGMA foreign_keys = OFF'` (SQLite) to skip FK checks during cleanup.

**"Tests pass individually but fail in parallel"**
Tests share the database and are mutating the same rows. Add `pool: { maxWorkers: 1 }` to vitest config to run integration tests sequentially, or use transaction rollback instead of `deleteMany`.

**"app.inject returns 500 with no useful message"**
Enable logging temporarily: `const app = Fastify({ logger: true })` and check the console output. Or add a global error handler that logs the full error.

---

## Further Reading

- [Fastify testing guide](https://fastify.dev/docs/latest/Guides/Testing/)
- [Prisma testing guide](https://www.prisma.io/docs/guides/testing/unit-testing)
- Track 09: [Testing Fundamentals](testing-fundamentals.md)
- Track 09: [E2E Testing](e2e-testing.md)

---

## Summary

Integration tests fill the gap between unit tests and E2E tests. They verify that your route handlers, services, and database repositories work together correctly — catching bugs that isolated unit tests miss. Use `app.inject()` for HTTP integration tests and a fast SQLite test database (or transaction-wrapped tests) for repository tests. Clean the database before each test, test both success and failure paths, and verify that writes actually persisted in the database. The effort pays off: integration tests catch the category of bugs that cause most production incidents.
