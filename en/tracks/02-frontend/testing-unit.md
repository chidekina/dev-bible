# Testing: Unit & Integration

## 1. What & Why

Testing is not about achieving 100% code coverage — it's about having confidence that your software works from a user's perspective. The right tests catch real bugs, survive refactors (because they test behavior, not implementation), and document what the code is supposed to do.

The guiding principle: **test what the user sees and does, not how the code is structured internally.** A test that breaks when you rename a private variable is a bad test. A test that breaks when the button stops submitting the form is a good test.

### The Testing Pyramid

```
        /\
       /E2E\          Few — slow, expensive, high confidence
      /------\
     /Integrat\       Some — test multiple units together
    /----------\
   /    Unit    \     Many — fast, isolated, low-level
  /______________\
```

Unit tests: isolated function or component tests. Fast, deterministic, catch logic bugs.
Integration tests: multiple units working together (component + API mock, service + repo). Catch wiring bugs.
E2E tests: full browser, real flows. Catch integration gaps and render bugs.

---

## 2. Core Concepts

- **Vitest** — Vite-native test runner, Jest-compatible API, fast
- **@testing-library/react** — renders components and provides user-centric query methods
- **MSW (Mock Service Worker)** — intercepts HTTP requests at the network level; no axios mocking needed
- **AAA pattern** — Arrange (set up), Act (do the thing), Assert (verify)
- **Query priority** — prefer queries that reflect accessibility over implementation details
- **Mocking** — replace external dependencies with controllable substitutes

---

## 3. How It Works

### Vitest Setup

```ts
// vitest.config.ts
import { defineConfig } from "vitest/config";
import react from "@vitejs/plugin-react";
import tsconfigPaths from "vite-tsconfig-paths";

export default defineConfig({
  plugins: [react(), tsconfigPaths()],
  test: {
    environment: "jsdom",           // browser-like environment
    globals: true,                  // no need to import describe/it/expect
    setupFiles: ["./src/test/setup.ts"],
    coverage: {
      provider: "v8",
      reporter: ["text", "lcov", "html"],
      exclude: ["**/*.d.ts", "**/*.config.*", "**/test/**"],
    },
  },
});
```

```ts
// src/test/setup.ts
import "@testing-library/jest-dom"; // extends expect with .toBeInTheDocument() etc.
import { server } from "./msw/server";

// MSW server lifecycle
beforeAll(() => server.listen({ onUnhandledRequest: "error" }));
afterEach(() => server.resetHandlers()); // reset per-test overrides
afterAll(() => server.close());
```

---

## 4. Code Examples

### Basic Vitest Assertions

```ts
// src/lib/price.ts
export function formatPrice(amount: number, currency = "USD"): string {
  return new Intl.NumberFormat("en-US", { style: "currency", currency }).format(
    amount / 100 // input in cents
  );
}

export function applyDiscount(price: number, percent: number): number {
  if (percent < 0 || percent > 100) throw new Error("Invalid discount");
  return Math.round(price * (1 - percent / 100));
}

// src/lib/__tests__/price.test.ts
import { describe, it, expect } from "vitest";
import { formatPrice, applyDiscount } from "../price";

describe("formatPrice", () => {
  it("formats cents as USD by default", () => {
    expect(formatPrice(1099)).toBe("$10.99");
  });

  it("formats zero", () => {
    expect(formatPrice(0)).toBe("$0.00");
  });

  it("supports other currencies", () => {
    expect(formatPrice(1000, "EUR")).toContain("€");
  });
});

describe("applyDiscount", () => {
  it("applies 10% discount", () => {
    expect(applyDiscount(1000, 10)).toBe(900);
  });

  it("rounds to nearest cent", () => {
    expect(applyDiscount(999, 10)).toBe(899); // 999 * 0.9 = 899.1 → 899
  });

  it("throws for negative discount", () => {
    expect(() => applyDiscount(100, -1)).toThrow("Invalid discount");
  });

  it("throws for discount > 100", () => {
    expect(() => applyDiscount(100, 101)).toThrow("Invalid discount");
  });
});
```

### Testing React Components

```tsx
// src/components/login-form.tsx
"use client";
import { useState } from "react";

interface Props {
  onSubmit: (email: string, password: string) => Promise<void>;
}

export function LoginForm({ onSubmit }: Props) {
  const [email, setEmail] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      await onSubmit(email, password);
    } catch (err) {
      setError(err instanceof Error ? err.message : "Login failed");
    } finally {
      setLoading(false);
    }
  }

  return (
    <form onSubmit={handleSubmit} aria-label="Login form">
      <div>
        <label htmlFor="email">Email</label>
        <input
          id="email"
          type="email"
          value={email}
          onChange={(e) => setEmail(e.target.value)}
          required
        />
      </div>
      <div>
        <label htmlFor="password">Password</label>
        <input
          id="password"
          type="password"
          value={password}
          onChange={(e) => setPassword(e.target.value)}
          required
        />
      </div>
      {error && <p role="alert">{error}</p>}
      <button type="submit" disabled={loading}>
        {loading ? "Signing in…" : "Sign in"}
      </button>
    </form>
  );
}
```

```tsx
// src/components/__tests__/login-form.test.tsx
import { render, screen, waitFor } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { vi, describe, it, expect, beforeEach } from "vitest";
import { LoginForm } from "../login-form";

// ─── Query Priority Guide ────────────────────────────────────────────────────
// Most preferred → least preferred:
// getByRole > getByLabelText > getByPlaceholderText > getByText
// > getByDisplayValue > getByAltText > getByTitle > getByTestId

describe("LoginForm", () => {
  const mockSubmit = vi.fn();

  beforeEach(() => {
    mockSubmit.mockReset();
  });

  it("renders email and password fields and a submit button", () => {
    // Arrange
    render(<LoginForm onSubmit={mockSubmit} />);

    // Assert — using accessibility-first queries
    expect(screen.getByLabelText(/email/i)).toBeInTheDocument();
    expect(screen.getByLabelText(/password/i)).toBeInTheDocument();
    expect(screen.getByRole("button", { name: /sign in/i })).toBeInTheDocument();
  });

  it("calls onSubmit with email and password when submitted", async () => {
    // Arrange
    const user = userEvent.setup();
    mockSubmit.mockResolvedValue(undefined);
    render(<LoginForm onSubmit={mockSubmit} />);

    // Act
    await user.type(screen.getByLabelText(/email/i), "alice@example.com");
    await user.type(screen.getByLabelText(/password/i), "password123");
    await user.click(screen.getByRole("button", { name: /sign in/i }));

    // Assert
    await waitFor(() => {
      expect(mockSubmit).toHaveBeenCalledWith("alice@example.com", "password123");
      expect(mockSubmit).toHaveBeenCalledTimes(1);
    });
  });

  it("shows loading state while submitting", async () => {
    const user = userEvent.setup();
    // Never resolves during this test
    mockSubmit.mockImplementation(() => new Promise(() => {}));
    render(<LoginForm onSubmit={mockSubmit} />);

    await user.type(screen.getByLabelText(/email/i), "a@b.com");
    await user.type(screen.getByLabelText(/password/i), "pass");
    await user.click(screen.getByRole("button", { name: /sign in/i }));

    expect(screen.getByRole("button", { name: /signing in/i })).toBeDisabled();
  });

  it("displays error message when login fails", async () => {
    const user = userEvent.setup();
    mockSubmit.mockRejectedValue(new Error("Invalid credentials"));
    render(<LoginForm onSubmit={mockSubmit} />);

    await user.type(screen.getByLabelText(/email/i), "a@b.com");
    await user.type(screen.getByLabelText(/password/i), "wrong");
    await user.click(screen.getByRole("button", { name: /sign in/i }));

    // findBy* returns a Promise — for async assertions
    expect(await screen.findByRole("alert")).toHaveTextContent("Invalid credentials");
  });

  it("re-enables submit button after error", async () => {
    const user = userEvent.setup();
    mockSubmit.mockRejectedValue(new Error("Error"));
    render(<LoginForm onSubmit={mockSubmit} />);

    await user.type(screen.getByLabelText(/email/i), "a@b.com");
    await user.type(screen.getByLabelText(/password/i), "wrong");
    await user.click(screen.getByRole("button", { name: /sign in/i }));

    await screen.findByRole("alert"); // wait for error
    expect(screen.getByRole("button", { name: /sign in/i })).not.toBeDisabled();
  });
});
```

### Query Methods Reference

```tsx
// getBy* — throws if not found or multiple found. Use for elements that must exist.
screen.getByRole("button", { name: /submit/i });
screen.getByLabelText("Email");
screen.getByText("Welcome back");
screen.getByTestId("spinner"); // last resort — couples test to implementation

// queryBy* — returns null if not found. Use for asserting absence.
expect(screen.queryByRole("alert")).not.toBeInTheDocument();
expect(screen.queryByText("Error")).toBeNull();

// findBy* — async, returns Promise. Use when element appears after async operation.
const error = await screen.findByRole("alert"); // waits up to 1000ms
const item = await screen.findByText("New item", {}, { timeout: 3000 });

// getAllBy*, queryAllBy*, findAllBy* — return arrays
const items = screen.getAllByRole("listitem");
expect(items).toHaveLength(3);
```

### Mocking with vi

```ts
// vi.fn() — creates a mock function
import { vi, expect, it } from "vitest";

const mockFn = vi.fn();
mockFn("hello");

expect(mockFn).toHaveBeenCalledWith("hello");
expect(mockFn).toHaveBeenCalledTimes(1);

// Mock return values
mockFn.mockReturnValue(42);
mockFn.mockReturnValueOnce(1).mockReturnValueOnce(2); // different on each call

// Mock async functions
const mockAsync = vi.fn().mockResolvedValue({ data: "ok" });
const mockFailing = vi.fn().mockRejectedValue(new Error("Network error"));

// vi.spyOn() — replace a method on an object, restore later
import * as utils from "./utils";
const spy = vi.spyOn(utils, "generateId").mockReturnValue("test-id-123");
// ... run test ...
spy.mockRestore(); // restore original

// vi.mock() — mock an entire module
vi.mock("@/lib/db", () => ({
  db: {
    user: {
      findMany: vi.fn().mockResolvedValue([
        { id: "1", name: "Alice", email: "alice@test.com" },
      ]),
      create: vi.fn().mockImplementation((args) => ({
        id: "new-id",
        ...args.data,
      })),
    },
  },
}));

// Use in tests after mocking
import { db } from "@/lib/db";

it("returns users from db", async () => {
  const users = await fetchUsers();
  expect(db.user.findMany).toHaveBeenCalled();
  expect(users).toHaveLength(1);
});
```

### MSW (Mock Service Worker)

```ts
// src/test/msw/handlers.ts
import { http, HttpResponse } from "msw";

export const handlers = [
  http.get("/api/users", () => {
    return HttpResponse.json([
      { id: "1", name: "Alice", email: "alice@test.com" },
      { id: "2", name: "Bob", email: "bob@test.com" },
    ]);
  }),

  http.post("/api/users", async ({ request }) => {
    const body = await request.json() as { name: string; email: string };
    return HttpResponse.json(
      { id: "new-id", ...body },
      { status: 201 }
    );
  }),

  http.get("/api/users/:id", ({ params }) => {
    if (params.id === "404") {
      return HttpResponse.json({ error: "Not found" }, { status: 404 });
    }
    return HttpResponse.json({ id: params.id, name: "Alice" });
  }),
];

// src/test/msw/server.ts
import { setupServer } from "msw/node";
import { handlers } from "./handlers";

export const server = setupServer(...handlers);
```

```tsx
// src/components/__tests__/user-list.test.tsx
import { render, screen } from "@testing-library/react";
import { server } from "../../test/msw/server";
import { http, HttpResponse } from "msw";
import { UserList } from "../user-list";

describe("UserList", () => {
  it("displays users from the API", async () => {
    render(<UserList />);

    // findBy* waits for async render
    expect(await screen.findByText("Alice")).toBeInTheDocument();
    expect(await screen.findByText("Bob")).toBeInTheDocument();
  });

  it("shows loading state initially", () => {
    render(<UserList />);
    expect(screen.getByRole("status")).toBeInTheDocument(); // <div role="status">Loading</div>
  });

  it("shows error when API fails", async () => {
    // Override handler for this specific test
    server.use(
      http.get("/api/users", () =>
        HttpResponse.json({ error: "Server error" }, { status: 500 })
      )
    );

    render(<UserList />);
    expect(await screen.findByRole("alert")).toHaveTextContent(/failed/i);
  });
});
```

### Testing with Context Providers

```tsx
// src/test/test-utils.tsx — custom render with all providers
import { render, RenderOptions } from "@testing-library/react";
import { ReactElement, ReactNode } from "react";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { AuthProvider } from "@/providers/auth";
import { ThemeProvider } from "@/providers/theme";

function createTestQueryClient() {
  return new QueryClient({
    defaultOptions: {
      queries: { retry: false }, // don't retry in tests
      mutations: { retry: false },
    },
  });
}

function AllProviders({ children }: { children: ReactNode }) {
  const queryClient = createTestQueryClient();
  return (
    <QueryClientProvider client={queryClient}>
      <AuthProvider>
        <ThemeProvider>{children}</ThemeProvider>
      </AuthProvider>
    </QueryClientProvider>
  );
}

// Override the default render
export function renderWithProviders(
  ui: ReactElement,
  options?: Omit<RenderOptions, "wrapper">
) {
  return render(ui, { wrapper: AllProviders, ...options });
}

// Re-export everything from testing-library for convenience
export * from "@testing-library/react";
export { renderWithProviders as render }; // shadow the original
```

### Testing Hooks

```tsx
// src/hooks/__tests__/use-debounce.test.ts
import { renderHook, act } from "@testing-library/react";
import { vi, beforeEach, afterEach, it, expect } from "vitest";
import { useDebounce } from "../use-debounce";

describe("useDebounce", () => {
  beforeEach(() => vi.useFakeTimers());
  afterEach(() => vi.useRealTimers());

  it("returns initial value immediately", () => {
    const { result } = renderHook(() => useDebounce("hello", 500));
    expect(result.current).toBe("hello");
  });

  it("debounces value updates", () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: "initial" } }
    );

    rerender({ value: "updated" });
    expect(result.current).toBe("initial"); // still old value

    act(() => vi.advanceTimersByTime(500));
    expect(result.current).toBe("updated"); // now updated
  });

  it("resets timer on rapid changes", () => {
    const { result, rerender } = renderHook(
      ({ value }) => useDebounce(value, 500),
      { initialProps: { value: "a" } }
    );

    rerender({ value: "ab" });
    act(() => vi.advanceTimersByTime(300)); // not enough time
    rerender({ value: "abc" });
    act(() => vi.advanceTimersByTime(300)); // timer reset — still not enough
    expect(result.current).toBe("a");

    act(() => vi.advanceTimersByTime(200)); // now 500ms from "abc"
    expect(result.current).toBe("abc");
  });
});
```

### AAA Pattern in Practice

```tsx
// Always structure tests as: Arrange → Act → Assert

it("adds item to cart", async () => {
  // ARRANGE
  const user = userEvent.setup();
  const onAddToCart = vi.fn().mockResolvedValue(undefined);
  render(<ProductCard product={{ id: "p1", name: "Widget", price: 999 }} onAddToCart={onAddToCart} />);

  // ACT
  await user.click(screen.getByRole("button", { name: /add to cart/i }));

  // ASSERT
  expect(onAddToCart).toHaveBeenCalledWith("p1");
  expect(screen.getByText(/added!/i)).toBeInTheDocument();
});
```

---

## 5. Common Mistakes & Pitfalls

**1. Testing implementation details (fragile tests)**
```tsx
// WRONG — test breaks when you rename the variable or change component structure
it("has the right state", () => {
  const { result } = renderHook(() => useCart());
  expect(result.current.cartItems.length).toBe(0); // testing internals
});

// CORRECT — test behavior from the user's perspective
it("shows empty cart message when no items", () => {
  render(<Cart />);
  expect(screen.getByText(/your cart is empty/i)).toBeInTheDocument();
});
```

**2. Not using userEvent (prefer over fireEvent)**
```tsx
// WRONG — fireEvent.click doesn't simulate focus, hover, or keyboard events
fireEvent.click(screen.getByRole("button"));

// CORRECT — userEvent simulates real user interactions
const user = userEvent.setup();
await user.click(screen.getByRole("button")); // triggers hover, focus, pointer events
await user.type(input, "hello"); // fires keydown, keypress, keyup per character
```

**3. Using getByTestId when a better query exists**
```tsx
// WRONG — couples test to DOM structure attribute
screen.getByTestId("submit-button");

// CORRECT — accessible query that also validates the button is properly labeled
screen.getByRole("button", { name: /submit/i });
```

**4. Asserting on snapshot instead of behavior**
```tsx
// AVOID for component tests — snapshot tests break on every styling change
expect(screen.getByRole("form")).toMatchSnapshot();

// PREFER — explicit, self-documenting assertions
expect(screen.getByRole("button", { name: /submit/i })).toBeEnabled();
expect(screen.getByLabelText(/email/i)).toHaveValue("alice@test.com");
```

**5. Missing act() warnings**
```tsx
// React warns: "act() was not called"
// This means state updates happened outside act() — usually async operations
it("loads data", async () => {
  render(<DataComponent />);
  // WRONG — not awaiting the DOM update
  expect(screen.getByText("Loaded")).toBeInTheDocument(); // fails or warns

  // CORRECT — findBy* wraps in act() automatically
  expect(await screen.findByText("Loaded")).toBeInTheDocument();
});
```

---

## 6. When to Use / Not Use

| Approach | Test when | Don't test |
|---------|-----------|-----------|
| Unit test | Pure functions, business logic, utility functions | Simple one-liners, getters/setters |
| Component test | User interactions, conditional rendering, form behavior | Internal state names, CSS classes, DOM structure |
| MSW | Components that fetch data | Exact HTTP request format (unless it's the contract) |
| vi.mock() | Modules with side effects (DB, email, analytics) | Pure functions (test them directly) |
| Snapshot | Stable, non-interactive output (email templates, icons) | Dynamic components that change often |

---

## 7. Real-World Scenario

Testing a `ProductSearch` component that fetches products from an API as the user types.

```tsx
// src/components/product-search.tsx
"use client";
import { useState, useEffect } from "react";

interface Product {
  id: string;
  name: string;
  price: number;
}

export function ProductSearch() {
  const [query, setQuery] = useState("");
  const [products, setProducts] = useState<Product[]>([]);
  const [loading, setLoading] = useState(false);
  const [error, setError] = useState<string | null>(null);

  useEffect(() => {
    if (!query.trim()) {
      setProducts([]);
      return;
    }

    const controller = new AbortController();
    setLoading(true);
    setError(null);

    fetch(`/api/products?q=${encodeURIComponent(query)}`, {
      signal: controller.signal,
    })
      .then((r) => {
        if (!r.ok) throw new Error("Search failed");
        return r.json() as Promise<Product[]>;
      })
      .then(setProducts)
      .catch((err) => {
        if (err.name !== "AbortError") setError(err.message);
      })
      .finally(() => setLoading(false));

    return () => controller.abort();
  }, [query]);

  return (
    <div>
      <label htmlFor="search">Search products</label>
      <input
        id="search"
        value={query}
        onChange={(e) => setQuery(e.target.value)}
        placeholder="Type to search…"
      />
      {loading && <p role="status">Searching…</p>}
      {error && <p role="alert">{error}</p>}
      <ul aria-label="Search results">
        {products.map((p) => (
          <li key={p.id}>{p.name} — ${(p.price / 100).toFixed(2)}</li>
        ))}
      </ul>
    </div>
  );
}
```

```tsx
// src/components/__tests__/product-search.test.tsx
import { render, screen, waitFor, waitForElementToBeRemoved } from "@testing-library/react";
import userEvent from "@testing-library/user-event";
import { server } from "../../test/msw/server";
import { http, HttpResponse } from "msw";
import { ProductSearch } from "../product-search";

const mockProducts = [
  { id: "1", name: "Widget Pro", price: 1999 },
  { id: "2", name: "Widget Lite", price: 999 },
];

describe("ProductSearch", () => {
  it("shows no results initially", () => {
    render(<ProductSearch />);
    expect(screen.getByRole("list", { name: /results/i })).toBeEmptyDOMElement();
  });

  it("fetches and displays results as user types", async () => {
    server.use(
      http.get("/api/products", ({ request }) => {
        const q = new URL(request.url).searchParams.get("q");
        const results = mockProducts.filter((p) =>
          p.name.toLowerCase().includes((q ?? "").toLowerCase())
        );
        return HttpResponse.json(results);
      })
    );

    const user = userEvent.setup();
    render(<ProductSearch />);

    await user.type(screen.getByLabelText(/search/i), "widget");

    expect(await screen.findByText(/Widget Pro/)).toBeInTheDocument();
    expect(screen.getByText(/Widget Lite/)).toBeInTheDocument();
  });

  it("shows loading state while fetching", async () => {
    server.use(
      http.get("/api/products", async () => {
        await new Promise((r) => setTimeout(r, 100));
        return HttpResponse.json([]);
      })
    );

    const user = userEvent.setup();
    render(<ProductSearch />);

    await user.type(screen.getByLabelText(/search/i), "x");
    expect(screen.getByRole("status")).toBeInTheDocument();

    await waitForElementToBeRemoved(() => screen.queryByRole("status"));
  });

  it("shows error when search fails", async () => {
    server.use(
      http.get("/api/products", () =>
        HttpResponse.json({ error: "Server error" }, { status: 500 })
      )
    );

    const user = userEvent.setup();
    render(<ProductSearch />);

    await user.type(screen.getByLabelText(/search/i), "w");
    expect(await screen.findByRole("alert")).toHaveTextContent("Search failed");
  });

  it("clears results when query is cleared", async () => {
    server.use(http.get("/api/products", () => HttpResponse.json(mockProducts)));

    const user = userEvent.setup();
    render(<ProductSearch />);
    const input = screen.getByLabelText(/search/i);

    await user.type(input, "widget");
    await screen.findByText(/Widget Pro/);

    await user.clear(input);
    await waitFor(() => {
      expect(screen.getByRole("list", { name: /results/i })).toBeEmptyDOMElement();
    });
  });
});
```

---

## 8. Interview Questions

**Q1: What is the testing pyramid?**

Three levels of tests organized by speed, cost, and confidence. Unit tests (bottom): fast, isolated, many — test individual functions or components. Integration tests (middle): test multiple units working together — component + mock API. E2E tests (top): slow, expensive, few — test full user flows in a real browser. Fewer tests at higher levels; more at lower levels.

**Q2: Why prefer getByRole over getByTestId?**

`getByRole` queries the accessible role of an element — the same thing a screen reader or accessibility tree sees. It tests that the element is correctly labeled and semantically meaningful. `getByTestId` is an implementation detail that adds `data-testid` attributes to production code for the sole purpose of testing — it doesn't validate accessibility and breaks when the attribute is removed. The query priority mirrors what a user experiences.

**Q3: What is MSW?**

Mock Service Worker intercepts HTTP requests at the network level using a Service Worker (in the browser) or a Node.js interceptor (in tests). Instead of mocking `fetch` or axios directly, MSW lets you define request handlers that respond like a real server. Tests use real network code paths; only the responses are controlled. This means your tests catch URL and serialization bugs that module-level mocks miss.

**Q4: Should you test implementation details?**

No. Tests that couple to implementation (internal state variable names, private methods, DOM class names) break when you refactor even without changing behavior. They create maintenance burden without adding confidence. Test observable behavior: what the user sees, what events are emitted, what API calls are made with what arguments.

**Q5: What is the AAA pattern?**

Arrange → Act → Assert. A structure for organizing test code. Arrange: set up test fixtures, mock dependencies, render the component. Act: perform the user action (click, type, navigate). Assert: verify the expected outcome (text visible, mock called, state updated). This structure makes tests readable and ensures each test has a single, clear purpose.

**Q6: Difference between getBy and findBy?**

`getBy*` is synchronous — it queries the DOM right now and throws if the element isn't there. `findBy*` is asynchronous — it returns a Promise that resolves when the element appears (polling the DOM up to a timeout, default 1000ms). Use `findBy*` after any async operation (API call, state update on a timer). Use `getBy*` for elements that should be in the DOM immediately after render.

**Q7: How do you test a component that fetches data?**

Set up MSW handlers that intercept the component's API calls and return mock data. Render the component, then use `findBy*` queries (async) to wait for the data to appear in the DOM. For error states, override the handler with `server.use()` to return an error response for that specific test. This tests the real fetch code path including error handling, without hitting the network.

---

## 9. Exercises

**Exercise 1: Test a form with validation**

Build a `RegistrationForm` with name, email, and password fields. Name must be at least 2 characters; password at least 8. Write tests for: (a) fields render correctly, (b) submit calls handler with values, (c) shows validation errors when fields are invalid, (d) clears errors when user corrects input.

**Exercise 2: Test a hook with timers**

Write tests for a `useCountdown(seconds: number)` hook that counts down to 0. Use `vi.useFakeTimers()`. Test: initial value is `seconds`, decrements every second, stops at 0, `isExpired` is `true` at 0.

**Exercise 3: Test with MSW**

Build a `UserProfile` component that fetches `/api/users/:id`. Write tests for: (a) renders user name when API succeeds, (b) shows skeleton while loading, (c) shows error when API returns 404, (d) retries when user clicks "Try again".

**Exercise 4: Custom render with providers**

Create a `renderWithProviders` utility that wraps components in a `QueryClient` provider and an `AuthContext` with a mock logged-in user. Use it to test a `DashboardHeader` that reads from `useAuth()` and shows the user's name.

---

## 10. Further Reading

- [Testing Library Docs](https://testing-library.com/docs/react-testing-library/intro/)
- [Vitest Docs](https://vitest.dev/)
- [MSW Docs](https://mswjs.io/docs/)
- [Kent C. Dodds — Testing Implementation Details](https://kentcdodds.com/blog/testing-implementation-details)
- [Kent C. Dodds — Write Tests. Not Too Many. Mostly Integration.](https://kentcdodds.com/blog/write-tests)
- [Kent C. Dodds — Common Mistakes with RTL](https://kentcdodds.com/blog/common-mistakes-with-react-testing-library)
- [Testing Library — Query Priority](https://testing-library.com/docs/queries/about#priority)
