# Test-Driven Development

## Overview

Test-Driven Development (TDD) is a development practice where you write the test before the implementation. The cycle is short: write a failing test (Red), make it pass with the minimal code (Green), then clean up the code without breaking the test (Refactor). This discipline shapes better design — code that is hard to test is usually poorly designed. TDD is not always appropriate, but knowing when and how to apply it is a mark of an experienced engineer.

---

## Prerequisites

- Track 09: Unit Testing
- Vitest setup and basic test writing

---

## Core Concepts

### The red-green-refactor cycle

```
RED → Write a failing test (the spec)
  ↓
GREEN → Write the minimum code to make it pass
  ↓
REFACTOR → Clean up without breaking the test
  ↓
(repeat)
```

**Red:** The test fails because the feature does not exist yet. This confirms your test actually tests something.

**Green:** Write the simplest code that makes the test pass — even if it's hacky. The test is now green; behavior is correct.

**Refactor:** Now clean up the implementation — extract functions, remove duplication, improve naming. The test ensures you haven't broken anything.

### When TDD pays off

- **Business rules** with many edge cases — the tests become executable specifications
- **APIs and service layers** — tests define the contract before implementation
- **Bug fixes** — write a failing test that reproduces the bug, then fix it (regression protection)
- **Data transformations** — pure functions with well-defined inputs/outputs

### When TDD is less useful

- **Exploratory prototyping** — you don't know the design yet; write tests after
- **UI components with complex rendering** — test behavior, not rendering details
- **Boilerplate** — CRUD endpoints with no logic; write the code, test the integration

---

## Hands-On Examples

### TDD kata: shopping cart

Build a shopping cart module using TDD. We define behavior through tests before writing any implementation.

**Iteration 1 — add items**

```typescript
// src/lib/cart.test.ts
import { describe, it, expect } from 'vitest';
import { Cart } from './cart.js';

describe('Cart', () => {
  it('starts empty', () => {
    const cart = new Cart();
    expect(cart.items).toHaveLength(0);
    expect(cart.total).toBe(0);
  });
});
```

Run: RED — `Cart` does not exist.

```typescript
// src/lib/cart.ts — minimum to make it green
export class Cart {
  items: CartItem[] = [];
  get total() { return 0; }
}
```

GREEN. Now expand:

```typescript
// Next test
it('adds an item to the cart', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 9.99, quantity: 1 });

  expect(cart.items).toHaveLength(1);
  expect(cart.items[0].name).toBe('Widget');
});
```

RED. Implement `add`:

```typescript
interface CartItem {
  id: string;
  name: string;
  price: number;
  quantity: number;
}

export class Cart {
  items: CartItem[] = [];

  get total(): number {
    return this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
  }

  add(item: CartItem): void {
    this.items.push(item);
  }
}
```

GREEN.

**Iteration 2 — quantity and totals**

```typescript
it('calculates total for multiple items', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 2 });
  cart.add({ id: '2', name: 'Gadget', price: 5, quantity: 3 });

  expect(cart.total).toBe(35); // 20 + 15
});

it('increments quantity when same item is added again', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 1 });
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 1 });

  expect(cart.items).toHaveLength(1);
  expect(cart.items[0].quantity).toBe(2);
});
```

RED for the second test. Update `add`:

```typescript
add(item: CartItem): void {
  const existing = this.items.find((i) => i.id === item.id);
  if (existing) {
    existing.quantity += item.quantity;
  } else {
    this.items.push({ ...item });
  }
}
```

GREEN.

**Iteration 3 — remove and discounts**

```typescript
it('removes an item from the cart', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 10, quantity: 1 });
  cart.remove('1');

  expect(cart.items).toHaveLength(0);
});

it('applies a percentage discount', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 100, quantity: 1 });
  cart.applyDiscount(10); // 10%

  expect(cart.total).toBe(90);
});

it('throws if discount is out of range', () => {
  const cart = new Cart();
  expect(() => cart.applyDiscount(-1)).toThrow('Discount must be between 0 and 100');
  expect(() => cart.applyDiscount(101)).toThrow('Discount must be between 0 and 100');
});
```

Implement `remove` and `applyDiscount`:

```typescript
export class Cart {
  items: CartItem[] = [];
  private discount = 0;

  get total(): number {
    const subtotal = this.items.reduce((sum, item) => sum + item.price * item.quantity, 0);
    return Math.round(subtotal * (1 - this.discount / 100) * 100) / 100;
  }

  add(item: CartItem): void {
    const existing = this.items.find((i) => i.id === item.id);
    if (existing) {
      existing.quantity += item.quantity;
    } else {
      this.items.push({ ...item });
    }
  }

  remove(id: string): void {
    this.items = this.items.filter((i) => i.id !== id);
  }

  applyDiscount(percent: number): void {
    if (percent < 0 || percent > 100) {
      throw new Error('Discount must be between 0 and 100');
    }
    this.discount = percent;
  }
}
```

GREEN. The tests serve as living specification — reading them tells you exactly what the cart is supposed to do.

### TDD for bug fixes

A user reports: "Applying a discount twice stacks — 10% + 10% = 20%."

1. Write a failing test that reproduces the bug:

```typescript
it('replaces existing discount when applied twice', () => {
  const cart = new Cart();
  cart.add({ id: '1', name: 'Widget', price: 100, quantity: 1 });
  cart.applyDiscount(10);
  cart.applyDiscount(10); // applying twice should not stack

  expect(cart.total).toBe(90); // not 81
});
```

2. Run: RED — confirms the bug exists (total is 81).
3. The fix is already in the implementation above (`this.discount = percent` replaces, not adds).
4. GREEN — bug is fixed, and the test prevents regression.

---

## Common Patterns & Best Practices

- **One failing test at a time** — do not write multiple tests before making the first green
- **Test names are specifications** — "returns 90 when a 10% discount is applied to a $100 cart"
- **Commit on green** — small, frequent commits with a passing test suite
- **The refactor step is not optional** — green code with duplication is technical debt; clean it immediately
- **Tests drive design** — if a test is hard to write, the code is probably poorly designed

---

## Anti-Patterns to Avoid

- **Test-after development** calling itself TDD — "I'll write the tests when I'm done" misses the design feedback loop
- **Skipping the refactor step** — accumulating green-but-messy code
- **Large test cases** — each test should test exactly one scenario; break large tests into smaller ones
- **Abandoning TDD because the first test is hard to write** — that difficulty is feedback; redesign the interface

---

## Debugging & Troubleshooting

**"My TDD cycle is too slow"**
Each iteration should be under 5 minutes. If a cycle is taking 20 minutes, you wrote too much at once. Shrink the step: write the most trivial test possible, make it green, then move to the next.

**"I don't know where to start"**
Start with the simplest possible behavior: "an empty cart has zero items". TDD grows features from the simplest case outward.

---

## Further Reading

- [Kent Beck: Test-Driven Development by Example](https://www.amazon.com/Test-Driven-Development-Kent-Beck/dp/0321146530)
- [Martin Fowler: Is TDD Dead?](https://martinfowler.com/articles/is-tdd-dead/) (video series)
- [Growing Object-Oriented Software, Guided by Tests](http://www.growing-object-oriented-software.com/)
- Track 09: [Unit Testing](unit-testing.md)

---

## Summary

TDD flips the development sequence: test, then code, then clean. The discipline has two payoffs — better design (code that is easy to test tends to be modular and decoupled) and immediate regression protection (you always have a passing test suite). The red-green-refactor cycle keeps the steps small and the feedback fast. TDD is most valuable for complex business logic, API contracts, and bug fixes. Applied consistently, it produces a test suite that is also a precise specification of the system's behavior.
