# Unit Testing

## Overview

Unit tests verify that a small, isolated piece of code — a function, a class method, a utility — behaves correctly given specific inputs. They are the fastest tests to write and run, and they give the most granular feedback when something breaks. A well-written unit test suite is the backbone of confident, fast-moving engineering. This chapter covers unit testing with vitest in a Node.js/TypeScript context: structure, patterns, async tests, and error testing.

---

## Prerequisites

- Track 09: Testing Fundamentals (test pyramid, test doubles)
- TypeScript basics
- Vitest setup from the fundamentals chapter

---

## Core Concepts

### What makes a good unit test

1. **Fast** — runs in milliseconds
2. **Isolated** — no I/O, no real network, no real database
3. **Deterministic** — same result every run, regardless of environment
4. **Focused** — tests exactly one behavior
5. **Self-describing** — the test name tells you what it verifies

### The function under test

A pure function (same input → same output, no side effects) is the easiest thing to unit test. The discipline of writing testable code often drives better design.

---

## Hands-On Examples

### Testing pure functions

```typescript
// src/lib/validation.ts
export function isValidEmail(email: string): boolean {
  return /^[^\s@]+@[^\s@]+\.[^\s@]+$/.test(email);
}

export function formatCurrency(amount: number, currency: string): string {
  return new Intl.NumberFormat('en-US', { style: 'currency', currency }).format(amount);
}

export function clamp(value: number, min: number, max: number): number {
  if (min > max) throw new Error('min cannot be greater than max');
  return Math.min(Math.max(value, min), max);
}
```

```typescript
// src/lib/validation.test.ts
import { describe, it, expect } from 'vitest';
import { isValidEmail, formatCurrency, clamp } from './validation.js';

describe('isValidEmail', () => {
  it('returns true for valid email addresses', () => {
    expect(isValidEmail('alice@example.com')).toBe(true);
    expect(isValidEmail('user+tag@domain.co.uk')).toBe(true);
  });

  it('returns false for invalid email addresses', () => {
    expect(isValidEmail('')).toBe(false);
    expect(isValidEmail('not-an-email')).toBe(false);
    expect(isValidEmail('@example.com')).toBe(false);
    expect(isValidEmail('user@')).toBe(false);
  });
});

describe('formatCurrency', () => {
  it('formats USD amounts', () => {
    expect(formatCurrency(1000, 'USD')).toBe('$1,000.00');
    expect(formatCurrency(0.5, 'USD')).toBe('$0.50');
  });

  it('formats EUR amounts', () => {
    expect(formatCurrency(42, 'EUR')).toBe('€42.00');
  });
});

describe('clamp', () => {
  it('returns the value when within range', () => {
    expect(clamp(5, 0, 10)).toBe(5);
    expect(clamp(0, 0, 10)).toBe(0);
    expect(clamp(10, 0, 10)).toBe(10);
  });

  it('clamps to min when value is too low', () => {
    expect(clamp(-5, 0, 10)).toBe(0);
  });

  it('clamps to max when value is too high', () => {
    expect(clamp(15, 0, 10)).toBe(10);
  });

  it('throws when min is greater than max', () => {
    expect(() => clamp(5, 10, 0)).toThrow('min cannot be greater than max');
  });
});
```

### Testing async functions

```typescript
// src/services/pricing.service.ts
export class PricingService {
  constructor(private exchangeRateApi: ExchangeRateApi) {}

  async convertPrice(amount: number, from: string, to: string): Promise<number> {
    if (from === to) return amount;

    const rate = await this.exchangeRateApi.getRate(from, to);
    if (!rate) throw new Error(`Exchange rate not available: ${from}/${to}`);

    return Math.round(amount * rate * 100) / 100;
  }
}
```

```typescript
// src/services/pricing.service.test.ts
import { describe, it, expect, vi, beforeEach } from 'vitest';
import { PricingService } from './pricing.service.js';

const mockExchangeRateApi = {
  getRate: vi.fn(),
};

let service: PricingService;

beforeEach(() => {
  vi.clearAllMocks();
  service = new PricingService(mockExchangeRateApi as any);
});

describe('PricingService.convertPrice', () => {
  it('returns the same amount when currencies match', async () => {
    const result = await service.convertPrice(100, 'USD', 'USD');
    expect(result).toBe(100);
    expect(mockExchangeRateApi.getRate).not.toHaveBeenCalled();
  });

  it('converts using the exchange rate', async () => {
    mockExchangeRateApi.getRate.mockResolvedValue(5.0); // 1 USD = 5 BRL

    const result = await service.convertPrice(100, 'USD', 'BRL');
    expect(result).toBe(500);
  });

  it('throws when exchange rate is unavailable', async () => {
    mockExchangeRateApi.getRate.mockResolvedValue(null);

    await expect(service.convertPrice(100, 'USD', 'XYZ')).rejects.toThrow(
      'Exchange rate not available: USD/XYZ'
    );
  });

  it('rounds to 2 decimal places', async () => {
    mockExchangeRateApi.getRate.mockResolvedValue(1.333);

    const result = await service.convertPrice(10, 'USD', 'EUR');
    expect(result).toBe(13.33); // not 13.330...
  });
});
```

### Testing with spies (tracking calls without changing behavior)

```typescript
// src/lib/audit.ts
import { logger } from './logger.js';

export function auditLog(action: string, userId: string, metadata?: Record<string, unknown>) {
  logger.info({ action, userId, ...metadata }, 'Audit event');
}
```

```typescript
import { describe, it, expect, vi, afterEach } from 'vitest';
import { logger } from './logger.js';
import { auditLog } from './audit.js';

vi.mock('./logger.js', () => ({
  logger: { info: vi.fn() },
}));

describe('auditLog', () => {
  afterEach(() => vi.clearAllMocks());

  it('logs the action and userId', () => {
    auditLog('login', 'user-123');

    expect(logger.info).toHaveBeenCalledOnce();
    expect(logger.info).toHaveBeenCalledWith(
      { action: 'login', userId: 'user-123' },
      'Audit event'
    );
  });

  it('includes metadata in the log', () => {
    auditLog('update', 'user-456', { field: 'email', oldValue: 'a@b.com' });

    expect(logger.info).toHaveBeenCalledWith(
      { action: 'update', userId: 'user-456', field: 'email', oldValue: 'a@b.com' },
      'Audit event'
    );
  });
});
```

### Testing with fake timers

```typescript
// src/lib/debounce.ts
export function debounce<T extends (...args: any[]) => any>(fn: T, delayMs: number): T {
  let timer: ReturnType<typeof setTimeout>;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn(...args), delayMs);
  } as T;
}
```

```typescript
import { describe, it, expect, vi, beforeEach, afterEach } from 'vitest';
import { debounce } from './debounce.js';

describe('debounce', () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it('delays calling the function', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 300);

    debounced('a');
    expect(fn).not.toHaveBeenCalled(); // not called yet

    vi.advanceTimersByTime(300);
    expect(fn).toHaveBeenCalledOnce();
    expect(fn).toHaveBeenCalledWith('a');
  });

  it('only calls once if invoked multiple times within the delay', () => {
    const fn = vi.fn();
    const debounced = debounce(fn, 300);

    debounced('a');
    debounced('b');
    debounced('c');

    vi.advanceTimersByTime(300);
    expect(fn).toHaveBeenCalledOnce();
    expect(fn).toHaveBeenCalledWith('c'); // only the last call
  });
});
```

### Testing error boundaries

```typescript
// src/lib/result.ts — neverthrow-style Result type
export type Result<T, E = Error> = { ok: true; value: T } | { ok: false; error: E };

export function parseJson<T>(json: string): Result<T, SyntaxError> {
  try {
    return { ok: true, value: JSON.parse(json) as T };
  } catch (e) {
    return { ok: false, error: e as SyntaxError };
  }
}
```

```typescript
describe('parseJson', () => {
  it('returns ok:true for valid JSON', () => {
    const result = parseJson<{ name: string }>('{"name":"Alice"}');
    expect(result.ok).toBe(true);
    if (result.ok) {
      expect(result.value.name).toBe('Alice');
    }
  });

  it('returns ok:false for invalid JSON', () => {
    const result = parseJson('not valid json');
    expect(result.ok).toBe(false);
    if (!result.ok) {
      expect(result.error).toBeInstanceOf(SyntaxError);
    }
  });
});
```

---

## Common Patterns & Best Practices

- **`beforeEach` for shared setup** — reset mocks, create fresh instances before each test
- **`afterEach` for cleanup** — restore timers, clear mocks, close connections
- **`vi.clearAllMocks()` in beforeEach** — prevents mock call counts from leaking between tests
- **Parametrized tests with `it.each`** — avoid repetitive test cases

```typescript
describe('clamp', () => {
  it.each([
    [5, 0, 10, 5],   // [value, min, max, expected]
    [-5, 0, 10, 0],
    [15, 0, 10, 10],
    [0, 0, 10, 0],
    [10, 0, 10, 10],
  ])('clamp(%i, %i, %i) = %i', (value, min, max, expected) => {
    expect(clamp(value, min, max)).toBe(expected);
  });
});
```

---

## Anti-Patterns to Avoid

- Mocking everything — tests that mock every dependency test nothing real
- Long tests that assert many unrelated things — split into focused tests
- Shared mutable state between tests — use `beforeEach` to reset
- Testing private methods — expose them through the public API or extract them to a separate module
- Not testing the unhappy path — only testing success cases misses most real bugs

---

## Debugging & Troubleshooting

**"My mock is not being called"**
Check the import path in `vi.mock()` — it must exactly match the path used in the file under test. Use relative paths consistently.

**"Test leaks state to the next test"**
Add `vi.clearAllMocks()` or `vi.resetAllMocks()` in `beforeEach`. For global state, explicitly reset it.

**"Async test times out"**
Make sure you `await` every async operation. If the test hangs, you may have an unresolved promise or a real I/O call that never resolves. Add a timeout: `it('...', async () => { ... }, 5000)`.

---

## Further Reading

- [Vitest: mocking guide](https://vitest.dev/guide/mocking.html)
- [Vitest: fake timers](https://vitest.dev/guide/mocking.html#timers)
- [Kent C. Dodds: Testing implementation details](https://kentcdodds.com/blog/testing-implementation-details)
- Track 09: [Testing Fundamentals](testing-fundamentals.md)
- Track 09: [Test-Driven Development](test-driven-development.md)

---

## Summary

Unit tests are the cheapest, fastest, and most plentiful layer of the test pyramid. Write them for all functions with non-trivial logic. Test pure functions directly (no mocking needed). Use `vi.fn()` for dependencies that make I/O calls or have side effects. Test both happy paths and error paths. Use `it.each` for parametrized cases. Keep each test focused on a single behavior, with a name that reads as a specification. The resulting test suite is not just a safety net — it is living documentation of what the code is supposed to do.
