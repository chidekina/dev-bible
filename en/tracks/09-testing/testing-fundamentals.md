# Testing Fundamentals

## Overview

Testing is the practice of verifying that software behaves as intended. Without a test suite, every change is a gamble — you push to production and hope nothing breaks. With a good test suite, you can refactor fearlessly, onboard new engineers safely, and ship with confidence. This chapter covers the philosophy and vocabulary of testing: the test pyramid, types of test doubles, what to test, and how to think about coverage.

---

## Prerequisites

- JavaScript/TypeScript fundamentals
- Basic familiarity with async/await
- Understanding of functions and modules

---

## Core Concepts

### The test pyramid

```
        /\
       /  \    E2E (few, slow, high confidence)
      /----\
     /      \  Integration (moderate, catches wiring bugs)
    /--------\
   /          \ Unit (many, fast, cheap to write)
  /____________\
```

| Layer | Speed | Cost | Confidence | Isolation |
|-------|-------|------|------------|-----------|
| Unit | Milliseconds | Low | Low (tests logic in isolation) | High |
| Integration | Seconds | Medium | Medium (tests real connections) | Medium |
| E2E | Minutes | High | High (tests the full system) | Low |

The pyramid shape reflects the recommended ratio: many fast unit tests at the base, fewer integration tests in the middle, and a small number of E2E tests at the top.

**Inverting the pyramid is a common mistake.** Teams that rely primarily on E2E tests have slow CI, brittle suites, and poor feedback loops.

### Types of test doubles

Test doubles replace real dependencies in unit tests:

| Type | Description | When to use |
|------|-------------|-------------|
| **Stub** | Returns a hardcoded value | Provide canned responses to queries |
| **Mock** | Verifies that a method was called with specific arguments | Test behavior, not just state |
| **Spy** | Records calls without changing behavior | Verify side effects |
| **Fake** | Working implementation, simplified (e.g., in-memory DB) | Integration-like tests without real infrastructure |
| **Dummy** | Placeholder with no behavior | Fill required parameters you don't care about |

### What to test

Test the behavior, not the implementation. Ask: "if I change the implementation without changing the behavior, should this test break?" If yes, you are testing implementation details.

**Test:**
- Public API of functions and classes
- Edge cases (empty arrays, null values, maximum values)
- Error handling paths
- Business rules and invariants

**Do not test:**
- Private methods directly
- Framework internals
- Simple getters/setters with no logic
- Third-party library behavior

---

## Hands-On Examples

### Setting up vitest

```bash
npm install --save-dev vitest @vitest/ui
```

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';

export default defineConfig({
  test: {
    globals: true,           // no need to import describe/it/expect
    environment: 'node',
    coverage: {
      provider: 'v8',
      reporter: ['text', 'lcov'],
      thresholds: {
        lines: 80,
        functions: 80,
        branches: 75,
      },
    },
  },
});
```

```json
// package.json scripts
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage",
    "test:ui": "vitest --ui"
  }
}
```

### Anatomy of a test

```typescript
// src/lib/math.test.ts
import { describe, it, expect } from 'vitest';
import { add, divide } from './math.js';

describe('add', () => {
  it('returns the sum of two positive numbers', () => {
    // Arrange
    const a = 3;
    const b = 4;

    // Act
    const result = add(a, b);

    // Assert
    expect(result).toBe(7);
  });

  it('handles negative numbers', () => {
    expect(add(-1, 1)).toBe(0);
    expect(add(-5, -3)).toBe(-8);
  });
});

describe('divide', () => {
  it('divides two numbers', () => {
    expect(divide(10, 2)).toBe(5);
  });

  it('throws when dividing by zero', () => {
    expect(() => divide(10, 0)).toThrow('Cannot divide by zero');
  });
});
```

The AAA pattern (Arrange-Act-Assert) is the standard structure. Each test should verify exactly one behavior.

### Test doubles in vitest

```typescript
// src/services/user.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { UserService } from './user.service.js';
import { EmailService } from './email.service.js';
import { UserRepository } from '../repositories/user.repository.js';

// Mock the entire module
vi.mock('../repositories/user.repository.js');
vi.mock('./email.service.js');

describe('UserService', () => {
  let userService: UserService;
  let mockUserRepo: UserRepository;
  let mockEmailService: EmailService;

  beforeEach(() => {
    mockUserRepo = {
      findById: vi.fn(),
      create: vi.fn(),
      update: vi.fn(),
    } as unknown as UserRepository;

    mockEmailService = {
      send: vi.fn(),
    } as unknown as EmailService;

    userService = new UserService(mockUserRepo, mockEmailService);
  });

  describe('createUser', () => {
    it('creates a user and sends a welcome email', async () => {
      // Stub: return value when repo.create is called
      const fakeUser = { id: '1', email: 'alice@example.com', name: 'Alice' };
      vi.mocked(mockUserRepo.create).mockResolvedValue(fakeUser);

      const result = await userService.createUser({ email: 'alice@example.com', name: 'Alice' });

      // Assert state
      expect(result).toEqual(fakeUser);

      // Assert behavior: email was sent
      expect(mockEmailService.send).toHaveBeenCalledOnce();
      expect(mockEmailService.send).toHaveBeenCalledWith({
        to: 'alice@example.com',
        template: 'welcome',
        variables: { name: 'Alice' },
      });
    });

    it('does not send email if user creation fails', async () => {
      vi.mocked(mockUserRepo.create).mockRejectedValue(new Error('DB error'));

      await expect(
        userService.createUser({ email: 'alice@example.com', name: 'Alice' })
      ).rejects.toThrow('DB error');

      expect(mockEmailService.send).not.toHaveBeenCalled();
    });
  });
});
```

### Coverage — what it tells you and what it doesn't

```bash
# Run tests with coverage
npm run test:coverage

# Sample output
----------|---------|----------|---------|---------|
File      | % Stmts | % Branch | % Funcs | % Lines |
----------|---------|----------|---------|---------|
All files |   87.5  |   75.0   |   90.0  |   87.5  |
 math.ts  |   100   |   100    |   100   |   100   |
 user.ts  |   80.0  |   60.0   |   85.7  |   80.0  |
----------|---------|----------|---------|---------|
```

Coverage measures which lines were executed during tests, not whether those lines were correctly tested. 100% coverage does not mean the code is correct — it means every line ran at least once. Treat coverage as a floor (80% minimum), not a goal.

---

## Common Patterns & Best Practices

- **One assertion per test** — when a test fails, you know exactly what broke
- **Descriptive test names** — "creates a user and sends welcome email" not "test createUser"
- **Arrange-Act-Assert structure** — keeps tests readable and focused
- **Test the interface, not the implementation** — tests should survive refactoring
- **Fast tests** — unit tests should run in milliseconds; slow tests get skipped
- **Deterministic tests** — avoid random data, time dependencies, and shared state between tests

---

## Anti-Patterns to Avoid

- Deleting tests to make the suite pass — you are hiding bugs
- Testing framework internals (React's reconciler, Express's routing internals)
- Tests that depend on execution order — every test must be independent
- Using real network calls in unit tests — they are slow, flaky, and test the wrong thing
- Testing private methods — only test the public interface
- Asserting implementation details: "this method was called 3 times" when the behavior is what matters

---

## Debugging & Troubleshooting

**"Tests pass locally but fail in CI"**
Most likely causes: environment differences (missing env vars), test ordering dependencies, or time-dependent tests. Run with `--sequence randomize` locally to catch ordering issues.

**"Coverage report shows a line as uncovered but I know tests call it"**
Check if the line is in a branch not taken. `if/else` — is the `else` branch tested? `try/catch` — is the `catch` block tested?

**"vi.mock is not working — real module is still called"**
Mock declarations must be at the top of the file (hoisted). Do not put `vi.mock` inside `beforeEach` or inside `describe`.

---

## Further Reading

- [Vitest documentation](https://vitest.dev/)
- [Testing Library principles](https://testing-library.com/docs/guiding-principles)
- [The practical test pyramid — Martin Fowler](https://martinfowler.com/articles/practical-test-pyramid.html)
- Track 09: [Unit Testing](unit-testing.md)
- Track 09: [Integration Testing](integration-testing.md)

---

## Summary

Testing is a skill that compounds over time. Start with the test pyramid as a guide: many fast unit tests, some integration tests, few E2E tests. Use test doubles to isolate units. Test behavior and contracts, not implementation details. Coverage is a useful floor but a dangerous ceiling — 80% line coverage with well-designed tests is better than 100% coverage with tests that only check that functions exist. The most important habit is writing tests as part of development, not as an afterthought — tests that come after the code tend to test what the code does, not what it should do.
