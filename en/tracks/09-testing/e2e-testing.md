# End-to-End Testing

## Overview

End-to-end (E2E) tests simulate real user interactions in a real browser, verifying that the entire application stack works together from the user's perspective. They test the login flow, the checkout process, the dashboard rendering — exactly as a user would experience them. Playwright is the leading E2E testing tool in 2025, with excellent TypeScript support, multi-browser testing, and a powerful API. This chapter covers setting up Playwright, the Page Object Model pattern, and integrating E2E tests into CI.

---

## Prerequisites

- Track 09: Testing Fundamentals
- Basic React/HTML knowledge
- Familiarity with async/await

---

## Hands-On Examples

### Playwright setup

```bash
npm install --save-dev @playwright/test
npx playwright install  # installs browser binaries
```

```typescript
// playwright.config.ts
import { defineConfig, devices } from '@playwright/test';

export default defineConfig({
  testDir: './e2e',
  timeout: 30000,
  retries: process.env.CI ? 2 : 0,  // retry flaky tests in CI
  workers: process.env.CI ? 2 : undefined,

  use: {
    baseURL: process.env.E2E_BASE_URL ?? 'http://localhost:3000',
    trace: 'on-first-retry',   // save trace for debugging flaky tests
    screenshot: 'only-on-failure',
  },

  projects: [
    { name: 'chromium', use: { ...devices['Desktop Chrome'] } },
    { name: 'firefox', use: { ...devices['Desktop Firefox'] } },
    // Mobile
    { name: 'mobile-chrome', use: { ...devices['Pixel 7'] } },
  ],

  // Start the dev server before tests
  webServer: {
    command: 'npm run dev',
    url: 'http://localhost:3000',
    reuseExistingServer: !process.env.CI,
  },
});
```

### Basic E2E tests

```typescript
// e2e/auth.spec.ts
import { test, expect } from '@playwright/test';

test.describe('Authentication', () => {
  test('user can log in with valid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email-input"]', 'alice@example.com');
    await page.fill('[data-testid="password-input"]', 'SecurePassword123!');
    await page.click('[data-testid="login-button"]');

    // Wait for redirect to dashboard
    await expect(page).toHaveURL('/dashboard');
    await expect(page.getByText('Welcome, Alice')).toBeVisible();
  });

  test('shows error with invalid credentials', async ({ page }) => {
    await page.goto('/login');

    await page.fill('[data-testid="email-input"]', 'alice@example.com');
    await page.fill('[data-testid="password-input"]', 'wrong-password');
    await page.click('[data-testid="login-button"]');

    await expect(page.getByText('Invalid credentials')).toBeVisible();
    await expect(page).toHaveURL('/login');
  });

  test('redirects unauthenticated users to login', async ({ page }) => {
    await page.goto('/dashboard');
    await expect(page).toHaveURL('/login');
  });
});
```

### Page Object Model (POM)

POM encapsulates page interactions in classes, preventing duplication and making tests readable:

```typescript
// e2e/pages/login.page.ts
import { Page, Locator } from '@playwright/test';

export class LoginPage {
  private readonly emailInput: Locator;
  private readonly passwordInput: Locator;
  private readonly submitButton: Locator;
  private readonly errorMessage: Locator;

  constructor(private page: Page) {
    this.emailInput = page.getByTestId('email-input');
    this.passwordInput = page.getByTestId('password-input');
    this.submitButton = page.getByTestId('login-button');
    this.errorMessage = page.getByRole('alert');
  }

  async goto() {
    await this.page.goto('/login');
  }

  async login(email: string, password: string) {
    await this.emailInput.fill(email);
    await this.passwordInput.fill(password);
    await this.submitButton.click();
  }

  async getErrorMessage(): Promise<string | null> {
    return this.errorMessage.textContent();
  }
}
```

```typescript
// e2e/pages/dashboard.page.ts
import { Page, expect } from '@playwright/test';

export class DashboardPage {
  constructor(private page: Page) {}

  async goto() {
    await this.page.goto('/dashboard');
  }

  async isLoaded() {
    await expect(this.page).toHaveURL('/dashboard');
  }

  async getWelcomeText(): Promise<string | null> {
    return this.page.getByRole('heading', { name: /welcome/i }).textContent();
  }

  async clickNewProject() {
    await this.page.getByTestId('new-project-button').click();
  }
}
```

```typescript
// e2e/auth.spec.ts — using POM
import { test, expect } from '@playwright/test';
import { LoginPage } from './pages/login.page.js';
import { DashboardPage } from './pages/dashboard.page.js';

test('full login flow', async ({ page }) => {
  const loginPage = new LoginPage(page);
  const dashboardPage = new DashboardPage(page);

  await loginPage.goto();
  await loginPage.login('alice@example.com', 'SecurePassword123!');

  await dashboardPage.isLoaded();
  const welcome = await dashboardPage.getWelcomeText();
  expect(welcome).toContain('Alice');
});
```

### Authentication state — avoid logging in on every test

```typescript
// e2e/auth.setup.ts — run once to save auth state
import { test as setup } from '@playwright/test';
import { LoginPage } from './pages/login.page.js';

setup('authenticate', async ({ page }) => {
  const loginPage = new LoginPage(page);
  await loginPage.goto();
  await loginPage.login(process.env.TEST_USER_EMAIL!, process.env.TEST_USER_PASSWORD!);

  // Save authenticated state (cookies + localStorage) to a file
  await page.context().storageState({ path: 'e2e/.auth/user.json' });
});
```

```typescript
// playwright.config.ts
export default defineConfig({
  projects: [
    {
      name: 'setup',
      testMatch: '**/auth.setup.ts',
    },
    {
      name: 'authenticated',
      use: {
        storageState: 'e2e/.auth/user.json', // reuse auth state
      },
      dependencies: ['setup'],
    },
  ],
});
```

### Testing forms and interactions

```typescript
// e2e/create-project.spec.ts
import { test, expect } from '@playwright/test';

test.use({ storageState: 'e2e/.auth/user.json' });

test('create a new project', async ({ page }) => {
  await page.goto('/dashboard');
  await page.getByTestId('new-project-button').click();

  // Fill out the form
  await page.getByLabel('Project Name').fill('My Awesome Project');
  await page.getByLabel('Description').fill('A test project');
  await page.getByRole('combobox', { name: 'Category' }).selectOption('web');

  // Submit
  await page.getByRole('button', { name: 'Create Project' }).click();

  // Verify success
  await expect(page.getByRole('alert')).toContainText('Project created');
  await expect(page).toHaveURL(/\/projects\/.+/);
  await expect(page.getByRole('heading')).toContainText('My Awesome Project');
});

test('shows validation errors for empty form', async ({ page }) => {
  await page.goto('/projects/new');
  await page.getByRole('button', { name: 'Create Project' }).click();

  await expect(page.getByText('Project name is required')).toBeVisible();
});
```

### Running E2E tests in CI

```yaml
# .github/workflows/e2e.yml
name: E2E Tests

on:
  pull_request:
    branches: [main]

jobs:
  e2e:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'

      - name: Install dependencies
        run: npm ci

      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium

      - name: Run E2E tests
        env:
          E2E_BASE_URL: ${{ secrets.STAGING_URL }}
          TEST_USER_EMAIL: ${{ secrets.TEST_USER_EMAIL }}
          TEST_USER_PASSWORD: ${{ secrets.TEST_USER_PASSWORD }}
        run: npx playwright test --project=chromium

      - name: Upload test artifacts on failure
        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: playwright-report
          path: playwright-report/
```

---

## Common Patterns & Best Practices

- **`data-testid` attributes** — stable selectors that do not change when styling changes
- **`getByRole` and `getByLabel`** — semantic selectors that also validate accessibility
- **Save auth state** — log in once per test run, reuse the state across tests
- **Run in headless mode in CI** — headed mode for local debugging
- **Retry on failure** in CI — E2E tests have inherent flakiness; 2 retries catch transient issues
- **Trace on failure** — Playwright traces are invaluable for debugging CI failures

---

## Anti-Patterns to Avoid

- E2E tests for logic that belongs in unit tests — E2E is for user flows, not business rules
- Relying on timing (`waitForTimeout(2000)`) instead of waiting for specific elements (`waitForSelector`)
- CSS class or text selectors that break when the UI changes — use `data-testid` or ARIA roles
- Not cleaning up test data — E2E tests should create and clean up their own data

---

## Debugging & Troubleshooting

**"Test passes locally but fails in CI"**
Network or rendering timing differences. Use `await page.waitForLoadState('networkidle')` or `await expect(element).toBeVisible()` instead of fixed timeouts. Check if the CI environment has the right env vars.

**"Locator error: found 3 elements matching"**
Your selector is too broad. Make it more specific: `page.getByTestId('submit-button')` is better than `page.getByRole('button')`.

**"How do I debug a failing test?"**
Run with `--headed` and `--slow-mo=500`:
`npx playwright test --headed --slow-mo=500 e2e/auth.spec.ts`
Or open the Playwright UI: `npx playwright test --ui`

---

## Further Reading

- [Playwright documentation](https://playwright.dev/docs/intro)
- [Playwright best practices](https://playwright.dev/docs/best-practices)
- [Page Object Model pattern](https://playwright.dev/docs/pom)
- Track 09: [Testing Fundamentals](testing-fundamentals.md)

---

## Summary

E2E tests sit at the top of the test pyramid: few in number, high in confidence. Playwright is the tool of choice for Node.js projects — TypeScript-native, multi-browser, and with excellent developer tooling. The Page Object Model keeps tests maintainable as the UI evolves. Saving authentication state and reusing it across tests dramatically reduces E2E suite runtime. In CI, combine E2E with trace recording and artifact upload to make failures diagnosable without reproducing locally. Limit E2E tests to critical user journeys — login, signup, checkout, core workflow — and leave edge cases and business logic to unit and integration tests.
