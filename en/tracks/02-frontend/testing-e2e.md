# Testing: End-to-End (E2E) with Playwright

## 1. What & Why

End-to-end tests simulate real users in a real browser. They click buttons, fill in forms, navigate pages, and assert on what they see — exactly as a user would. No mocking of the application layer, no simulated DOM. A real browser renders the real application.

**Why E2E when unit tests exist?**

Unit tests verify that individual pieces of code work in isolation. E2E tests catch bugs that only appear when pieces work together:

- The API endpoint exists but the component sends the wrong URL
- The form validation passes but the backend rejects the data
- The redirect works but lands on a broken page
- Authentication works but the session cookie isn't set on the right domain

These are integration bugs — they live in the gaps between units. E2E tests are the only test type that catches them reliably.

**The tradeoff:** E2E tests are slower (10–60 seconds per test vs milliseconds for unit tests), require a running application, and can be flaky if written poorly. Write few, high-value E2E tests for critical user flows. Don't try to replace unit tests.

---

## 2. Core Concepts

- **Playwright** — Microsoft's E2E testing tool; runs tests in Chromium, Firefox, and WebKit (Safari engine)
- **Locators** — the way Playwright finds elements; prefer accessibility-based queries
- **Auto-waiting** — Playwright waits for elements to be ready before interacting; no manual `sleep()`
- **Page Object Model (POM)** — class per page with typed methods; tests read like English
- **Fixtures** — Playwright's dependency injection for reusable setup (logged-in sessions, seeded data)
- **Traces** — Playwright records screenshots + network + DOM snapshots on failure; essential for CI debugging

---

## 3. How It Works

### Playwright Config

```ts
// playwright.config.ts
import { defineConfig, devices } from "@playwright/test";

export default defineConfig({
  testDir: "./e2e",
  fullyParallel: true,       // run test files in parallel
  forbidOnly: !!process.env.CI, // fail if test.only is committed
  retries: process.env.CI ? 2 : 0, // retry flaky tests in CI
  workers: process.env.CI ? 1 : undefined,
  reporter: [
    ["list"],                  // console output
    ["html", { open: "never" }], // HTML report at playwright-report/
    ...(process.env.CI ? [["github"] as ["github"]] : []), // GitHub annotations
  ],
  use: {
    baseURL: process.env.BASE_URL ?? "http://localhost:3000",
    trace: "on-first-retry",  // record trace on first retry
    screenshot: "only-on-failure",
    video: "retain-on-failure",
  },
  projects: [
    { name: "chromium", use: { ...devices["Desktop Chrome"] } },
    { name: "firefox", use: { ...devices["Desktop Firefox"] } },
    { name: "webkit", use: { ...devices["Desktop Safari"] } },
    { name: "mobile-chrome", use: { ...devices["Pixel 5"] } },
  ],
  webServer: {
    command: "npm run build && npm run start",
    url: "http://localhost:3000",
    reuseExistingServer: !process.env.CI, // reuse dev server locally
    timeout: 120_000,
  },
});
```

---

## 4. Code Examples

### Basic Test Structure

```ts
// e2e/auth.spec.ts
import { test, expect } from "@playwright/test";

test.describe("Authentication", () => {
  test("user can log in with valid credentials", async ({ page }) => {
    // Navigate
    await page.goto("/login");

    // Fill in form — using accessibility-first locators
    await page.getByLabel("Email").fill("alice@example.com");
    await page.getByLabel("Password").fill("secure-password-123");

    // Submit
    await page.getByRole("button", { name: "Sign in" }).click();

    // Assert — Playwright auto-waits for the URL to change
    await expect(page).toHaveURL("/dashboard");
    await expect(page.getByRole("heading", { name: "Dashboard" })).toBeVisible();
  });

  test("shows error for invalid credentials", async ({ page }) => {
    await page.goto("/login");

    await page.getByLabel("Email").fill("wrong@example.com");
    await page.getByLabel("Password").fill("wrong-password");
    await page.getByRole("button", { name: "Sign in" }).click();

    // Playwright auto-waits for the alert to appear
    await expect(page.getByRole("alert")).toContainText("Invalid credentials");
    await expect(page).toHaveURL("/login"); // stayed on login page
  });

  test("redirects to login when accessing protected route", async ({ page }) => {
    await page.goto("/dashboard");
    await expect(page).toHaveURL(/\/login/);
  });
});
```

### Locator Strategies

```ts
// Priority order: getByRole → getByLabel → getByPlaceholder → getByText → getByTestId

// getByRole — most preferred: mirrors the accessibility tree
page.getByRole("button", { name: "Submit" });
page.getByRole("heading", { name: "Dashboard", level: 1 });
page.getByRole("textbox", { name: "Email" });
page.getByRole("link", { name: "Learn more" });
page.getByRole("checkbox", { name: "Remember me" });
page.getByRole("combobox", { name: "Country" });
page.getByRole("listitem"); // all list items

// getByLabel — for form fields with associated labels
page.getByLabel("Email address");
page.getByLabel("Search products");

// getByPlaceholder — when no label is present
page.getByPlaceholder("Enter your email");

// getByText — for content
page.getByText("Welcome back, Alice");
page.getByText(/\d+ items/); // regex

// getByAltText — for images
page.getByAltText("Company logo");

// getByTestId — last resort: add data-testid to elements that have no accessible text
page.getByTestId("product-card-123");

// Chaining — scope within a parent
const productCard = page.getByRole("article").filter({ hasText: "Widget Pro" });
await productCard.getByRole("button", { name: "Add to cart" }).click();

// nth — when multiple matching elements exist
await page.getByRole("listitem").nth(0).click(); // first item
await page.getByRole("listitem").last().click();  // last item
```

> ⚠️ **Avoid `page.locator("div.btn-primary")` CSS selectors.** They couple tests to styling and break on any refactor. They also don't validate accessibility. Use role-based locators.

### Auto-Waiting — No Sleep Needed

```ts
// Playwright auto-waits for elements before interacting:
// - Attached to DOM
// - Visible
// - Stable (not animating)
// - Enabled (not disabled)
// - Receives events (not covered by overlay)

// WRONG — manual sleeps are fragile and slow
await page.click("button");
await page.waitForTimeout(2000); // arbitrary sleep
expect(await page.textContent(".result")).toContain("Success");

// CORRECT — use expect() with auto-retry
await page.getByRole("button", { name: "Submit" }).click();
// Playwright retries this assertion until it passes or times out
await expect(page.getByRole("alert")).toContainText("Success");
await expect(page.getByText("Order #12345")).toBeVisible();

// Waiting for navigation
await page.getByRole("link", { name: "Products" }).click();
await expect(page).toHaveURL("/products");

// Waiting for network
const responsePromise = page.waitForResponse("/api/orders");
await page.getByRole("button", { name: "Place Order" }).click();
const response = await responsePromise;
expect(response.status()).toBe(201);

// Waiting for an element to disappear
await expect(page.getByRole("status", { name: "Loading" })).not.toBeVisible();
```

### Page Object Model (POM)

```ts
// e2e/pages/login-page.ts
import { Page, Locator, expect } from "@playwright/test";

export class LoginPage {
  readonly page: Page;
  readonly emailInput: Locator;
  readonly passwordInput: Locator;
  readonly submitButton: Locator;
  readonly errorAlert: Locator;

  constructor(page: Page) {
    this.page = page;
    this.emailInput = page.getByLabel("Email");
    this.passwordInput = page.getByLabel("Password");
    this.submitButton = page.getByRole("button", { name: "Sign in" });
    this.errorAlert = page.getByRole("alert");
  }

  async goto() {
    await this.page.goto("/login");
    await expect(this.page).toHaveTitle(/Login/);
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async expectError(message: string | RegExp) {
    await expect(this.errorAlert).toContainText(message);
  }
}

// e2e/pages/dashboard-page.ts
import { Page, Locator, expect } from "@playwright/test";

export class DashboardPage {
  readonly page: Page;
  readonly heading: Locator;
  readonly navMenu: Locator;

  constructor(page: Page) {
    this.page = page;
    this.heading = page.getByRole("heading", { level: 1 });
    this.navMenu = page.getByRole("navigation");
  }

  async goto() {
    await this.page.goto("/dashboard");
    await expect(this.page).toHaveURL("/dashboard");
  }

  async getWelcomeMessage() {
    return this.page.getByText(/Welcome,/).textContent();
  }
}

// e2e/auth.spec.ts — tests now read like English
import { test, expect } from "@playwright/test";
import { LoginPage } from "./pages/login-page";
import { DashboardPage } from "./pages/dashboard-page";

test("successful login redirects to dashboard", async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login("alice@example.com", "password123");

  await expect(page).toHaveURL("/dashboard");
  await expect(dashboardPage.heading).toBeVisible();
});

test("invalid credentials show error", async ({ page }) => {
  const loginPage = new LoginPage(page);

  await loginPage.goto();
  await loginPage.login("wrong@example.com", "badpass");
  await loginPage.expectError("Invalid credentials");
});
```

### Fixtures

```ts
// e2e/fixtures.ts — reusable, typed test setup
import { test as base, expect, Page } from "@playwright/test";
import { LoginPage } from "./pages/login-page";
import { DashboardPage } from "./pages/dashboard-page";

// Define the shape of your fixtures
type Fixtures = {
  authenticatedPage: Page;         // page already logged in as a regular user
  adminPage: Page;                  // page logged in as admin
  loginPage: LoginPage;
  dashboardPage: DashboardPage;
};

// Extend the base test with custom fixtures
export const test = base.extend<Fixtures>({
  // authenticatedPage — log in once via API (faster than UI login)
  authenticatedPage: async ({ browser }, use) => {
    // Create a new browser context
    const context = await browser.newContext();
    const page = await context.newPage();

    // Log in via API — much faster than filling a form
    const response = await page.request.post("/api/auth/login", {
      data: { email: "alice@example.com", password: "password123" },
    });
    const { token } = await response.json();

    // Set auth cookie/storage
    await context.addCookies([{
      name: "session",
      value: token,
      domain: "localhost",
      path: "/",
    }]);

    await use(page);
    await context.close();
  },

  // adminPage — same pattern, different user
  adminPage: async ({ browser }, use) => {
    const context = await browser.newContext();
    const page = await context.newPage();

    const response = await page.request.post("/api/auth/login", {
      data: { email: "admin@example.com", password: "admin-pass" },
    });
    const { token } = await response.json();
    await context.addCookies([{ name: "session", value: token, domain: "localhost", path: "/" }]);

    await use(page);
    await context.close();
  },

  loginPage: async ({ page }, use) => {
    await use(new LoginPage(page));
  },

  dashboardPage: async ({ page }, use) => {
    await use(new DashboardPage(page));
  },
});

export { expect };

// e2e/dashboard.spec.ts — uses fixture; no login boilerplate
import { test, expect } from "./fixtures";

test("dashboard shows user name", async ({ authenticatedPage }) => {
  await authenticatedPage.goto("/dashboard");
  await expect(authenticatedPage.getByText("Alice")).toBeVisible();
});

test("admin can access user management", async ({ adminPage }) => {
  await adminPage.goto("/admin/users");
  await expect(adminPage.getByRole("heading", { name: "User Management" })).toBeVisible();
});
```

### Global Setup and Teardown

```ts
// e2e/global-setup.ts — runs once before all tests
import { chromium, FullConfig } from "@playwright/test";

export default async function globalSetup(config: FullConfig) {
  // Seed test database
  await fetch(`${config.projects[0].use.baseURL}/api/test/seed`, {
    method: "POST",
    headers: { "x-test-secret": process.env.TEST_SECRET ?? "" },
  });

  // Pre-generate auth state to avoid logging in in every test
  const browser = await chromium.launch();
  const page = await browser.newPage();

  await page.request.post("/api/auth/login", {
    data: { email: "alice@example.com", password: "password123" },
  });

  // Save storage state (cookies + localStorage)
  await page.context().storageState({ path: "e2e/.auth/user.json" });

  await browser.close();
}

// e2e/global-teardown.ts — runs once after all tests
export default async function globalTeardown() {
  // Clean up test database
  await fetch("http://localhost:3000/api/test/cleanup", { method: "POST" });
}

// playwright.config.ts — register global setup
export default defineConfig({
  globalSetup: "./e2e/global-setup.ts",
  globalTeardown: "./e2e/global-teardown.ts",
  // ...
});

// Use saved auth state in a project
// playwright.config.ts
{
  name: "authenticated",
  use: {
    storageState: "e2e/.auth/user.json", // pre-authenticated
  },
  testMatch: "e2e/authenticated/**/*.spec.ts",
}
```

### CI Integration

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on: [push, pull_request]

jobs:
  e2e:
    timeout-minutes: 30
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: 20
          cache: "npm"

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Build application
        run: npm run build

      - name: Run E2E tests
        run: npx playwright test
        env:
          BASE_URL: http://localhost:3000
          TEST_SECRET: ${{ secrets.TEST_SECRET }}

      - name: Upload test report
        uses: actions/upload-artifact@v4
        if: always() # upload even on failure
        with:
          name: playwright-report
          path: playwright-report/
          retention-days: 30

      - name: Upload traces on failure
        uses: actions/upload-artifact@v4
        if: failure()
        with:
          name: playwright-traces
          path: test-results/
```

```ts
// Run specific tests in CI
npx playwright test --project=chromium           // one browser only
npx playwright test e2e/auth.spec.ts             // specific file
npx playwright test --grep "login"               // tests matching regex
npx playwright test --reporter=github            // GitHub annotations
```

### Visual Regression Testing

```ts
// e2e/visual.spec.ts
import { test, expect } from "@playwright/test";

test("homepage looks correct", async ({ page }) => {
  await page.goto("/");
  // Captures a screenshot and compares to a stored baseline
  await expect(page).toHaveScreenshot("homepage.png");
});

test("product card visual regression", async ({ page }) => {
  await page.goto("/products");
  const card = page.getByRole("article").first();
  // Screenshot just the component
  await expect(card).toHaveScreenshot("product-card.png", {
    maxDiffPixels: 100, // allow small differences (antialiasing, etc.)
  });
});

// Update baselines when intentional visual changes are made:
// npx playwright test --update-snapshots
```

> ⚠️ **Visual regression tests are sensitive to fonts, OS, and browser rendering differences.** Run them in a consistent Docker environment in CI. Use `maxDiffPixels` or `threshold` to tolerate minor antialiasing differences.

### API Testing with Playwright

```ts
// e2e/api/users.spec.ts — test the API directly
import { test, expect } from "@playwright/test";

test.describe("Users API", () => {
  test("GET /api/users returns a list", async ({ request }) => {
    const response = await request.get("/api/users");

    expect(response.status()).toBe(200);
    const body = await response.json();
    expect(body).toMatchObject({
      users: expect.arrayContaining([
        expect.objectContaining({ id: expect.any(String), name: expect.any(String) }),
      ]),
    });
  });

  test("POST /api/users creates a user", async ({ request }) => {
    const response = await request.post("/api/users", {
      data: { name: "Test User", email: `test-${Date.now()}@example.com` },
    });

    expect(response.status()).toBe(201);
    const user = await response.json();
    expect(user.id).toBeDefined();
    expect(user.name).toBe("Test User");
  });

  test("GET /api/users/:id returns 404 for unknown id", async ({ request }) => {
    const response = await request.get("/api/users/nonexistent-id");
    expect(response.status()).toBe(404);
  });
});
```

### Debugging Flaky Tests

```ts
// Strategies for debugging flaky E2E tests:

// 1. Run in headed mode with slow motion — watch what's happening
// npx playwright test --headed --slow-mo=500

// 2. Open the Playwright Inspector for step-by-step debugging
// PWDEBUG=1 npx playwright test

// 3. Record a trace and analyze it
test("flaky test", async ({ page }) => {
  await page.tracing.start({ screenshots: true, snapshots: true });

  // ... test code ...

  await page.tracing.stop({ path: "trace.zip" });
  // npx playwright show-trace trace.zip
});

// 4. Use waitFor with explicit conditions instead of relying on auto-wait timing
await page.waitForFunction(() => {
  const el = document.querySelector("[data-loaded]");
  return el !== null;
});

// 5. Increase timeout for a specific assertion
await expect(page.getByText("Slow content")).toBeVisible({ timeout: 10_000 });

// 6. Screenshot on every step in debugging mode
if (process.env.DEBUG_SCREENSHOTS) {
  await page.screenshot({ path: `debug-${Date.now()}.png` });
}
```

---

## 5. Common Mistakes & Pitfalls

**1. Using arbitrary sleeps instead of auto-waiting**
```ts
// WRONG — fragile: fails if the page loads faster or slower
await page.click("button");
await page.waitForTimeout(3000); // 3 second sleep

// CORRECT — Playwright retries until condition is met
await page.getByRole("button").click();
await expect(page.getByText("Success")).toBeVisible();
```

**2. Not using Page Object Model for multi-step flows**
```ts
// WRONG — copy-paste login code in every test (10 tests = 10 copies to maintain)
test("checkout flow", async ({ page }) => {
  await page.goto("/login");
  await page.getByLabel("Email").fill("alice@example.com");
  await page.getByLabel("Password").fill("pass");
  await page.getByRole("button", { name: "Sign in" }).click();
  // ...
});

// CORRECT — extract to fixture, login once via API
test("checkout flow", async ({ authenticatedPage }) => {
  await authenticatedPage.goto("/cart");
  // ...
});
```

**3. Testing everything with E2E (too expensive)**
```ts
// WRONG — E2E test for a utility function
test("formatPrice formats 1099 as $10.99", async ({ page }) => {
  await page.goto("/product/123");
  await expect(page.getByText("$10.99")).toBeVisible();
});
// This is a unit test in disguise — test formatPrice() directly in Vitest

// CORRECT — reserve E2E for full user flows that can't be tested otherwise
test("user can complete checkout from cart to confirmation", async ({ authenticatedPage }) => {
  // Full checkout flow — fills cart, enters shipping, pays, sees confirmation
});
```

**4. Hardcoding test data that can collide between parallel test runs**
```ts
// WRONG — if two tests run in parallel, they fight over the same user
await page.getByLabel("Email").fill("testuser@example.com");

// CORRECT — unique data per test
const uniqueEmail = `test-${Date.now()}-${Math.random().toString(36).slice(2)}@example.com`;
await page.getByLabel("Email").fill(uniqueEmail);
```

**5. Forgetting to handle cookies/state between tests**
```ts
// By default, each test gets a fresh browser context (isolated cookies/storage)
// But if you share a context across tests in a describe block, state leaks

// WRONG — shared context leaks login state
let page: Page;
test.beforeAll(async ({ browser }) => {
  page = await browser.newPage();
  await login(page);
});

// CORRECT — use fixtures which handle context lifecycle cleanly
test("dashboard", async ({ authenticatedPage }) => { ... });
```

---

## 6. When to Use / Not Use

| Type | Use E2E for | Use unit/integration instead |
|------|-------------|------------------------------|
| Auth flows | Login, logout, protected route redirect | Auth utility functions, JWT validation |
| Critical paths | Checkout, signup, payment | Cart calculation logic, price formatting |
| Integrations | API → UI wiring, form → database | Individual API handler, form validation |
| Cross-browser | Layout differences, browser-specific behavior | Logic that doesn't depend on the browser |
| Visual | Final UI appearance, regression | CSS unit tests (just use visual E2E) |

---

## 7. Real-World Scenario

Complete E2E tests for an e-commerce checkout flow.

```ts
// e2e/checkout.spec.ts
import { test, expect } from "./fixtures"; // uses authenticatedPage fixture

test.describe("Checkout Flow", () => {
  test.beforeEach(async ({ authenticatedPage }) => {
    // Seed cart via API for consistent starting state
    await authenticatedPage.request.post("/api/test/seed-cart", {
      data: { items: [{ productId: "widget-pro", quantity: 2 }] },
    });
  });

  test("user can view cart and proceed to checkout", async ({ authenticatedPage: page }) => {
    await page.goto("/cart");

    // Verify cart contents
    await expect(page.getByText("Widget Pro")).toBeVisible();
    await expect(page.getByText("Qty: 2")).toBeVisible();
    await expect(page.getByText("$39.98")).toBeVisible(); // 2 × $19.99

    // Proceed to checkout
    await page.getByRole("button", { name: "Proceed to Checkout" }).click();
    await expect(page).toHaveURL("/checkout");
  });

  test("user can complete checkout with shipping and payment", async ({ authenticatedPage: page }) => {
    await page.goto("/checkout");

    // Shipping form
    await page.getByLabel("Full Name").fill("Alice Smith");
    await page.getByLabel("Address").fill("123 Main St");
    await page.getByLabel("City").fill("New York");
    await page.getByLabel("ZIP Code").fill("10001");

    // Payment — test card number
    const cardFrame = page.frameLocator("[data-testid='stripe-card-frame']");
    await cardFrame.getByPlaceholder("Card number").fill("4242 4242 4242 4242");
    await cardFrame.getByPlaceholder("MM / YY").fill("12 / 26");
    await cardFrame.getByPlaceholder("CVC").fill("123");

    // Place order
    await page.getByRole("button", { name: "Place Order" }).click();

    // Wait for confirmation
    await expect(page).toHaveURL(/\/orders\/\w+\/confirmation/, { timeout: 15_000 });
    await expect(page.getByRole("heading", { name: "Order Confirmed!" })).toBeVisible();
    await expect(page.getByText("Widget Pro")).toBeVisible();

    // Verify order number appears
    const orderNumber = await page.getByTestId("order-number").textContent();
    expect(orderNumber).toMatch(/^ORD-\d+/);
  });

  test("shows validation errors for incomplete shipping", async ({ authenticatedPage: page }) => {
    await page.goto("/checkout");

    // Submit without filling shipping
    await page.getByRole("button", { name: "Place Order" }).click();

    await expect(page.getByText("Full name is required")).toBeVisible();
    await expect(page.getByText("Address is required")).toBeVisible();
    await expect(page).toHaveURL("/checkout"); // didn't navigate away
  });

  test("empty cart redirects to products page", async ({ authenticatedPage: page }) => {
    // Clear cart
    await page.request.delete("/api/cart");

    await page.goto("/checkout");
    await expect(page).toHaveURL("/products");
    await expect(page.getByText("Your cart is empty")).toBeVisible();
  });
});
```

---

## 8. Interview Questions

**Q1: Why not just write unit tests?**

Unit tests verify isolated behavior. They cannot catch bugs that exist in the connections between units: a component sending the wrong API URL, a backend rejecting valid-looking data, authentication working but the session not persisting across redirects. E2E tests exercise the real application end-to-end — they're the only tests that guarantee the user flow actually works in a real browser.

**Q2: What is the Page Object Model?**

A pattern where each page of the application has a corresponding class with typed locators and methods. Tests create page objects and call their methods rather than duplicating locator code across tests. Benefits: when a locator changes, you update it in one place; tests read like English; test logic is separated from navigation and interaction details.

**Q3: How does Playwright auto-waiting work?**

Before every interaction (`.click()`, `.fill()`, etc.), Playwright checks that the element is: attached to the DOM, visible, stable (not animating), enabled (not disabled), and not covered by another element. If any condition isn't met, Playwright retries until the default timeout (usually 30 seconds). For assertions, `expect()` also retries automatically. This eliminates the need for manual `waitForTimeout()` calls.

**Q4: How do you debug a flaky E2E test?**

First, run in headed mode with `--headed --slow-mo=500` to watch the test execute slowly. If that's not enough, run with `PWDEBUG=1` to open the Inspector and step through interactions. On CI, enable traces (`trace: 'on-first-retry'`) and download the trace artifact — Playwright's `show-trace` shows a timeline of network requests, DOM snapshots, and screenshots at each step. Look for race conditions: the test asserting before an async operation completes.

**Q5: What are fixtures in Playwright?**

Fixtures are Playwright's dependency injection system. You extend the base `test` object with custom setup code that runs before each test and teardown code that runs after. Fixtures handle: creating authenticated browser contexts once (not per test), creating page objects, seeding databases, and providing shared configuration. They compose: a `dashboardPage` fixture can depend on an `authenticatedPage` fixture.

**Q6: How do you run E2E tests in CI?**

Install Playwright browsers with `npx playwright install --with-deps` in CI. Run `npx playwright test` against a built application (or a test environment). Configure `--reporter=github` for GitHub Actions annotations. Upload the HTML report and test artifacts (screenshots, traces) as CI artifacts so failures can be investigated without re-running. Set `retries: 2` in CI to handle occasional network flakiness.

**Q7: Visual regression testing — what is it?**

`expect(page).toHaveScreenshot()` captures a screenshot of the page (or an element) and compares it pixel-by-pixel to a stored baseline image. If the diff exceeds the threshold, the test fails. This catches unintended visual changes — layout shifts, color changes, missing icons. Update baselines intentionally with `--update-snapshots`. Run in a fixed environment (Docker) to avoid OS-level rendering differences.

---

## 9. Exercises

**Exercise 1: Write a login test**

Using a real (or mocked) login page, write three E2E tests: (a) successful login redirects to dashboard, (b) wrong password shows error, (c) accessing `/dashboard` without being logged in redirects to `/login`.

**Exercise 2: Build a Page Object Model**

Take the tests from Exercise 1 and extract them to a `LoginPage` class with methods `goto()`, `login(email, password)`, and `expectError(message)`. Rewrite the tests to use the class.

**Exercise 3: Create an authenticated fixture**

Write a `test.extend()` fixture called `loggedInPage` that logs in via `page.request.post('/api/auth/login')` and sets the session cookie before yielding the page. Use it in a test for a protected page.

**Exercise 4: Write a checkout E2E test**

Write an E2E test that: (1) navigates to a product page, (2) adds the product to the cart, (3) proceeds to checkout, (4) fills shipping info, (5) submits the order, (6) asserts on the confirmation page showing the order number.

**Exercise 5: API test**

Using `request` fixture from Playwright (no browser needed), test: (a) `GET /api/products` returns 200 with an array, (b) `POST /api/products` with valid data returns 201 with an id, (c) `GET /api/products/:id` returns 404 for an unknown ID.

---

## 10. Further Reading

- [Playwright Docs](https://playwright.dev/)
- [Playwright — Best Practices](https://playwright.dev/docs/best-practices)
- [Playwright — Page Object Model](https://playwright.dev/docs/pom)
- [Playwright — Fixtures](https://playwright.dev/docs/test-fixtures)
- [Playwright — CI Setup](https://playwright.dev/docs/ci)
- [Playwright — Visual Comparisons](https://playwright.dev/docs/test-snapshots)
- [Kent C. Dodds — Static vs Unit vs Integration vs E2E Tests](https://kentcdodds.com/blog/static-vs-unit-vs-integration-vs-e2e-tests)
