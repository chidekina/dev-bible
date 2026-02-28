# Mocking & Stubs

## Overview

Mocking and stubbing are the techniques that make unit tests possible. Without them, every test would require a real database, a live email service, and a running third-party API — making tests slow, brittle, and dependent on external state. This chapter covers the mechanics of mocking in vitest, when to use mocks versus alternatives, and the critical discipline of not over-mocking. It also covers MSW (Mock Service Worker) for mocking HTTP in frontend tests.

---

## Prerequisites

- Track 09: Testing Fundamentals (test doubles vocabulary)
- Track 09: Unit Testing
- Basic understanding of ES modules

---

## Core Concepts

### The mocking spectrum

```
No mocking (real dependencies)
  → Integration test territory
  → Catches real wiring bugs
  → Slow, requires infrastructure

Fake (in-memory implementation)
  → Fast, realistic behavior
  → Good for repositories, storage

Stub (returns hardcoded values)
  → Fast, minimal setup
  → Good for external APIs, config

Mock (verifies call behavior)
  → Checks HOW code interacts with dependencies
  → Use sparingly — couples test to implementation

Full mocking (mock everything)
  → Tests test nothing real
  → Anti-pattern
```

### The "Don't mock what you don't own" rule

Avoid mocking types you don't control (e.g., `fetch`, `axios`, Prisma's `PrismaClient`). Mock at the boundary you own — mock your repository interface, not Prisma directly. This decouples tests from library internals.

---

## Hands-On Examples

### Module mocking with vi.mock

```typescript
// src/services/notification.service.ts
import { sendEmail } from '../lib/mailer.js'; // external dependency

export async function notifyUser(userId: string, message: string) {
  const user = await getUserById(userId);
  await sendEmail({ to: user.email, subject: 'Notification', body: message });
  return { sent: true };
}
```

```typescript
// src/services/notification.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';

// vi.mock is hoisted — runs before imports
vi.mock('../lib/mailer.js', () => ({
  sendEmail: vi.fn().mockResolvedValue({ id: 'email-123' }),
}));

vi.mock('../lib/user.js', () => ({
  getUserById: vi.fn().mockResolvedValue({ id: 'u1', email: 'alice@example.com' }),
}));

import { notifyUser } from './notification.service.js';
import { sendEmail } from '../lib/mailer.js';
import { getUserById } from '../lib/user.js';

beforeEach(() => vi.clearAllMocks());

describe('notifyUser', () => {
  it('sends an email to the user', async () => {
    await notifyUser('u1', 'Hello!');

    expect(sendEmail).toHaveBeenCalledWith({
      to: 'alice@example.com',
      subject: 'Notification',
      body: 'Hello!',
    });
  });

  it('returns sent:true on success', async () => {
    const result = await notifyUser('u1', 'Hello!');
    expect(result.sent).toBe(true);
  });
});
```

### Spying on methods without replacing them

```typescript
import { vi, describe, it, expect, afterEach } from 'vitest';
import { logger } from '../lib/logger.js';

describe('logger spy', () => {
  afterEach(() => vi.restoreAllMocks());

  it('logs the correct message', () => {
    const spy = vi.spyOn(logger, 'info');

    doSomethingThatLogs();

    expect(spy).toHaveBeenCalledWith(expect.objectContaining({ action: 'user.login' }), 'Audit');
  });
});
```

`vi.spyOn` wraps the method — the real implementation still runs, but calls are recorded.

### Factory-style mock constructors

```typescript
// Create reusable mock factories
function createMockUserRepo() {
  return {
    findById: vi.fn<[string], Promise<User | null>>().mockResolvedValue(null),
    findByEmail: vi.fn<[string], Promise<User | null>>().mockResolvedValue(null),
    create: vi.fn<[CreateUserDto], Promise<User>>(),
    update: vi.fn<[string, Partial<User>], Promise<User>>(),
    delete: vi.fn<[string], Promise<void>>().mockResolvedValue(undefined),
  };
}

describe('UserService', () => {
  it('throws when user not found', async () => {
    const mockRepo = createMockUserRepo();
    mockRepo.findById.mockResolvedValue(null); // explicit for clarity

    const service = new UserService(mockRepo);
    await expect(service.getUser('non-existent')).rejects.toThrow('User not found');
  });

  it('updates and returns the user', async () => {
    const mockRepo = createMockUserRepo();
    const existingUser = { id: '1', email: 'a@b.com', name: 'Alice', role: 'user' } as User;
    const updatedUser = { ...existingUser, name: 'Alicia' };

    mockRepo.findById.mockResolvedValue(existingUser);
    mockRepo.update.mockResolvedValue(updatedUser);

    const service = new UserService(mockRepo);
    const result = await service.updateUser('1', { name: 'Alicia' });

    expect(result.name).toBe('Alicia');
    expect(mockRepo.update).toHaveBeenCalledWith('1', { name: 'Alicia' });
  });
});
```

### MSW — mocking HTTP requests

MSW intercepts `fetch`/`axios` calls at the network level, without patching the fetch global:

```bash
npm install --save-dev msw
```

```typescript
// src/test/msw-handlers.ts
import { http, HttpResponse } from 'msw';

export const handlers = [
  http.get('https://api.github.com/users/:username', ({ params }) => {
    return HttpResponse.json({
      login: params.username,
      name: 'Test User',
      public_repos: 42,
    });
  }),

  http.post('https://api.stripe.com/v1/charges', () => {
    return HttpResponse.json({ id: 'ch_test_123', status: 'succeeded' });
  }),

  // Simulate error responses
  http.get('https://api.example.com/flaky', () => {
    return new HttpResponse(null, { status: 503 });
  }),
];
```

```typescript
// src/test/msw-setup.ts
import { setupServer } from 'msw/node';
import { handlers } from './msw-handlers.js';

export const server = setupServer(...handlers);

// In vitest setup file:
beforeAll(() => server.listen({ onUnhandledRequest: 'error' }));
afterEach(() => server.resetHandlers()); // restore defaults between tests
afterAll(() => server.close());
```

```typescript
// src/services/github.service.test.ts
import { describe, it, expect } from 'vitest';
import { server } from '../test/msw-setup.js';
import { http, HttpResponse } from 'msw';
import { getGithubUser } from './github.service.js';

describe('getGithubUser', () => {
  it('returns user data from GitHub API', async () => {
    const user = await getGithubUser('octocat');
    expect(user.login).toBe('octocat');
    expect(user.publicRepos).toBe(42);
  });

  it('throws on rate limit', async () => {
    // Override handler for this specific test
    server.use(
      http.get('https://api.github.com/users/:username', () => {
        return new HttpResponse(null, { status: 429 });
      })
    );

    await expect(getGithubUser('anyone')).rejects.toThrow('Rate limited');
  });
});
```

### Partial mocking — mock only what you need

```typescript
// Mock only the problematic export, not the whole module
vi.mock('../lib/db.js', async (importOriginal) => {
  const actual = await importOriginal<typeof import('../lib/db.js')>();
  return {
    ...actual, // keep all real exports
    db: {      // override only the db client
      user: {
        findUnique: vi.fn(),
        create: vi.fn(),
      },
    },
  };
});
```

---

## Common Patterns & Best Practices

- **Mock at the boundary you own** — repository interface, not the ORM
- **Reset mocks in `beforeEach`** — prevents test pollution
- **Use `mockResolvedValue` for async** and `mockReturnValue` for sync
- **Assert on the behavior, not the mock call count** when possible
- **Use MSW for HTTP** — more realistic than patching `fetch`; catches URL mismatches
- **Type your mocks** — `vi.fn<[argType], ReturnType>()` catches type errors in mock setup

---

## Anti-Patterns to Avoid

- Mocking every dependency in every test — over-mocking creates tests that test the mocks
- Verifying mock call count when it's an implementation detail — prefer asserting the outcome
- Not resetting mocks between tests — one test's mock return value bleeds into the next
- Mocking modules that are pure functions with no side effects — just call the real function

---

## Debugging & Troubleshooting

**"vi.mock is ignored — real module is still called"**
`vi.mock` must be at the top level of the file (it is hoisted by Vitest). If it's inside a `describe` or `it`, it won't work. Also check that the module path matches the import in the file under test.

**"MSW is catching requests it shouldn't"**
Set `onUnhandledRequest: 'error'` in `server.listen()` — this makes unhandled requests throw, helping you discover where real requests are sneaking through.

**"Mock returns undefined"**
You forgot `.mockResolvedValue(...)` or `.mockReturnValue(...)`. A `vi.fn()` without a configured return value returns `undefined` by default.

---

## Further Reading

- [Vitest mocking guide](https://vitest.dev/guide/mocking.html)
- [MSW documentation](https://mswjs.io/docs/)
- [Kent C. Dodds: Stop mocking fetch](https://kentcdodds.com/blog/stop-mocking-fetch)
- Track 09: [Unit Testing](unit-testing.md)
- Track 09: [Integration Testing](integration-testing.md)

---

## Summary

Mocking is a tool for isolation, not a testing philosophy. Use stubs to provide canned responses, spies to verify side effects, and MSW to intercept HTTP without patching globals. Always reset mocks between tests. Mock at the interface boundary you own — not deep into library internals. The most important restraint is knowing when not to mock: if a function is pure or an integration test would give more confidence, prefer the real thing. Over-mocked tests are maintenance burdens that break when implementation changes, not when behavior changes.
