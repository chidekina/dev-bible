# Testing Challenges

> Practical testing exercises covering unit, integration, E2E, TDD, and performance testing. Each exercise reflects a real production testing scenario. Stack assumed: Node.js 20+, TypeScript, Vitest, Fastify, Playwright, MSW. Target skill level: beginner to advanced.

---

## Exercise 1 — Unit Tests for a Pure Function (Easy)

**Scenario:** The following utility function has no tests. Write a comprehensive unit test suite for it.

**Given code:**
```typescript
export function paginate<T>(items: T[], page: number, limit: number): {
  data: T[];
  meta: { page: number; limit: number; total: number; totalPages: number; hasNext: boolean; hasPrev: boolean };
} {
  if (page < 1) throw new Error('Page must be >= 1');
  if (limit < 1 || limit > 100) throw new Error('Limit must be between 1 and 100');
  const total = items.length;
  const totalPages = Math.ceil(total / limit);
  const start = (page - 1) * limit;
  const data = items.slice(start, start + limit);
  return {
    data,
    meta: { page, limit, total, totalPages, hasNext: page < totalPages, hasPrev: page > 1 },
  };
}
```

**Requirements:**
- Achieve 100% branch and statement coverage on the `paginate` function.
- Use Vitest with `describe`/`it` blocks, not flat `test` calls.
- Each `it` tests one behavior — no omnibus "it works" tests.
- Use `expect.assertions(N)` in tests that verify thrown errors.

**Acceptance Criteria:**
- [ ] Happy path: first page, middle page, last page all return correct `data` slices.
- [ ] Edge: `page = 1, limit = 100, items.length = 0` returns `{ data: [], meta: { total: 0, totalPages: 0, hasNext: false, hasPrev: false } }`.
- [ ] `page < 1` throws `'Page must be >= 1'`.
- [ ] `limit = 101` throws `'Limit must be between 1 and 100'`.
- [ ] `hasNext` and `hasPrev` are `true`/`false` on the correct pages.
- [ ] Vitest coverage report shows 100% coverage.

**Hints:**
1. Use `it.each` for parameterized happy-path cases: `[[1, 10, 100], [2, 10, 100], [10, 10, 100]]` (page, limit, total items).
2. For error cases: `expect(() => paginate([], 0, 10)).toThrow('Page must be >= 1')`.
3. Generate test items with `Array.from({ length: N }, (_, i) => i + 1)`.
4. Group tests: `describe('pagination math')`, `describe('error cases')`, `describe('meta flags')`.

---

## Exercise 2 — Integration Tests for a Fastify Route (Medium)

**Scenario:** Write integration tests for a `POST /users` route against a real (in-memory) SQLite database. No mocking of the DB layer.

**Requirements:**
- Use `better-sqlite3` as the in-memory database for tests.
- Each test gets a fresh database (set up in `beforeEach`, torn down in `afterEach`).
- Tests build the Fastify app using the same factory function as production — no test-specific app instances.
- Test at the HTTP layer (use `fastify.inject`) — not by calling service functions directly.
- Cover: success case, validation failure, duplicate email conflict.

**Acceptance Criteria:**
- [ ] `POST /users` with valid data returns `201` and the created user (without password hash).
- [ ] `POST /users` with a missing `email` field returns `422` with a field-level error.
- [ ] `POST /users` with an already-registered email returns `409 Conflict`.
- [ ] Each test is independent — creating a user in test 1 does not affect test 2.
- [ ] The test DB is created with migrations applied (same migration files as production).

**Hints:**
1. App factory: `export function buildApp(db: Database) { const fastify = Fastify(); /* register routes with injected db */; return fastify; }`.
2. In `beforeEach`: `const db = new Database(':memory:'); applyMigrations(db); app = buildApp(db); await app.ready();`.
3. In `afterEach`: `await app.close(); db.close();`.
4. `fastify.inject` example: `const res = await app.inject({ method: 'POST', url: '/users', payload: { email: 'a@b.com', password: 'secret' } })`.

---

## Exercise 3 — Playwright E2E for a Login Flow (Medium)

**Scenario:** Write a Playwright E2E test suite that covers the login flow of a web application: valid login, wrong password, and redirect after login.

**Requirements:**
- Tests run against a locally running server (started before the test suite, stopped after).
- Use Playwright's Page Object Model — create a `LoginPage` class with locator helpers.
- Cover: successful login → redirect to `/dashboard`, wrong password → error message, empty form → HTML5 validation.
- Tests must be independent — each test starts from a clean state (fresh user account).
- Run tests in headed mode locally and headless in CI.

**Acceptance Criteria:**
- [ ] `LoginPage` class has methods: `goto()`, `fillEmail(email)`, `fillPassword(password)`, `submit()`, `getErrorMessage()`.
- [ ] Successful login test asserts: URL changes to `/dashboard` and user name appears in the nav bar.
- [ ] Wrong password test asserts: error message `"Invalid email or password"` is visible.
- [ ] Tests create a test user via API (`POST /test-helpers/users`) before each test — not via UI.
- [ ] CI config (`playwright.config.ts`) uses `headless: true` and runs on `ubuntu-latest`.

**Hints:**
1. Page Object:
   ```typescript
   class LoginPage {
     constructor(private page: Page) {}
     goto() { return this.page.goto('/login'); }
     fillEmail(email: string) { return this.page.fill('[data-testid="email"]', email); }
   }
   ```
2. Use `test.beforeEach` to create a fresh test user via `request.post('/test-helpers/users', { data: { email, password } })`.
3. `test.use({ baseURL: 'http://localhost:3000' })` in `playwright.config.ts`.
4. Use `await expect(page).toHaveURL('/dashboard')` — Playwright auto-waits for navigation.

---

## Exercise 4 — TDD Kata: Shopping Cart (Medium)

**Scenario:** Implement a shopping cart using Test-Driven Development. Write the tests first, then implement only enough code to make each test pass. Do not write any implementation code before the corresponding test.

**Requirements (implement in this order):**
1. An empty cart has 0 items and a total of 0.
2. Adding an item increases the item count.
3. Adding the same item twice increases its quantity (not adds a duplicate).
4. `removeItem` decreases quantity by 1; removing the last unit removes the item.
5. `getTotal` returns the sum of `price * quantity` for all items.
6. A 10% discount coupon reduces the total by 10%.
7. Applying an invalid coupon throws `InvalidCouponError`.

**Acceptance Criteria:**
- [ ] Git history shows a commit per red-green-refactor cycle (at least 7 commits for 7 behaviors).
- [ ] No implementation code exists before its corresponding failing test.
- [ ] Final implementation is < 60 lines of code (constraints force simplicity).
- [ ] All 7 behaviors have tests; coverage is 100%.
- [ ] The `Cart` class has no dependencies — it is a pure in-memory data structure.

**Hints:**
1. TDD cycle: write a failing test → write minimal code to pass → refactor → commit → repeat.
2. Start with: `it('starts empty', () => { const cart = new Cart(); expect(cart.itemCount).toBe(0); })`.
3. `Cart` internal state: `items: Map<string, { price: number; quantity: number }>`.
4. Coupon validation: a hardcoded set of valid coupons (`{ 'SAVE10': 0.10 }`) is sufficient for the kata.

---

## Exercise 5 — Mock an External HTTP API with MSW (Medium)

**Scenario:** Your service calls a third-party weather API. Tests currently make real HTTP requests, making them slow and flaky. Replace them with MSW (Mock Service Worker) mocks.

**Requirements:**
- Set up MSW server-side handlers for `GET https://api.weather.com/v1/current?city=<city>`.
- Happy path handler returns `{ city, temperature, unit: 'celsius', condition: 'sunny' }`.
- Add error handlers: 404 for unknown cities, 500 for `city=error`.
- Tests for your `getWeather(city: string)` service function must use MSW — no `jest.mock` or `vi.mock` on the HTTP client.
- MSW server starts before the test suite and resets handlers between tests.

**Acceptance Criteria:**
- [ ] `getWeather('London')` returns the mocked payload (no real HTTP call).
- [ ] `getWeather('unknown-city')` throws a `WeatherNotFoundError`.
- [ ] `getWeather('error')` throws a `WeatherServiceError`.
- [ ] Real HTTP requests are blocked during tests (MSW's `onUnhandledRequest: 'error'` mode).
- [ ] Adding a new test that needs a different response uses `server.use(http.get(...))` to override the default handler for that test only.

**Hints:**
1. MSW setup for Node.js (not browser): `import { setupServer } from 'msw/node'`.
2. Handler: `http.get('https://api.weather.com/v1/current', ({ request }) => { const url = new URL(request.url); const city = url.searchParams.get('city'); return HttpResponse.json({ city, temperature: 20 }); })`.
3. In `beforeAll`: `server.listen({ onUnhandledRequest: 'error' })`. In `afterEach`: `server.resetHandlers()`. In `afterAll`: `server.close()`.
4. Per-test override: `server.use(http.get('https://api.weather.com/v1/current', () => HttpResponse.error()))`.

---

## Exercise 6 — Testing Async Error Handling (Medium)

**Scenario:** A service function fetches user data, sends a welcome email, and logs the event. Write tests that verify correct error handling when each step fails independently.

**Given code:**
```typescript
async function onboardUser(userId: string): Promise<void> {
  const user = await userRepo.findById(userId);
  if (!user) throw new UserNotFoundError(userId);
  await emailService.sendWelcome(user.email);
  await analytics.track('user_onboarded', { userId });
}
```

**Requirements:**
- Test: user not found → `UserNotFoundError` is thrown, email is not sent.
- Test: email service throws → error propagates, analytics is not called.
- Test: analytics throws → error propagates (email was already sent, no rollback).
- Test: happy path → email is sent and analytics is tracked.
- All dependencies (`userRepo`, `emailService`, `analytics`) are injected — no module-level mocking.

**Acceptance Criteria:**
- [ ] `userRepo.findById` returning `null` causes `onboardUser` to throw `UserNotFoundError`.
- [ ] `emailService.sendWelcome` is never called when the user is not found.
- [ ] `analytics.track` is never called when `emailService.sendWelcome` throws.
- [ ] Each test uses Vitest spies (`vi.fn()`) injected via the constructor or function arguments.
- [ ] No `try/catch` in test bodies — use `expect(promise).rejects.toThrow(UserNotFoundError)`.

**Hints:**
1. Inject dependencies: `function onboardUser(userId, deps: { userRepo, emailService, analytics })`.
2. Mock `userRepo.findById`: `vi.fn().mockResolvedValue(null)` for not-found case.
3. Mock email failure: `vi.fn().mockRejectedValue(new Error('SMTP timeout'))`.
4. Assert not called: `expect(emailService.sendWelcome).not.toHaveBeenCalled()`.

---

## Exercise 7 — Achieve 80% Coverage on a Given Module (Medium)

**Scenario:** A colleague wrote a `discountCalculator.ts` module with 0% test coverage. Your task is to reach 80% line + branch coverage without modifying the implementation.

**Given code:**
```typescript
export function calculateDiscount(
  price: number,
  userType: 'regular' | 'vip' | 'employee',
  promoCode?: string
): number {
  if (price <= 0) return 0;
  let discount = 0;
  if (userType === 'vip') discount += 0.15;
  if (userType === 'employee') discount += 0.30;
  if (promoCode === 'SUMMER20') discount += 0.20;
  if (promoCode === 'FLASH50') discount = 0.50; // overrides other discounts
  if (discount > 0.50) discount = 0.50; // cap at 50%
  return Math.round(price * (1 - discount) * 100) / 100;
}
```

**Requirements:**
- Write tests to reach at least 80% line coverage and 80% branch coverage.
- Document in a comment which branches you intentionally skipped and why.
- Use `vitest --coverage` to verify the coverage thresholds.
- Configure `vitest.config.ts` to enforce 80% thresholds — build fails below threshold.

**Acceptance Criteria:**
- [ ] `vitest --coverage` reports >= 80% lines and >= 80% branches for `discountCalculator.ts`.
- [ ] The VIP + SUMMER20 combination (capped at 50%) is tested.
- [ ] The `FLASH50` override (ignores other discounts) is tested.
- [ ] `vitest.config.ts` `coverage.thresholds` is set to `{ lines: 80, branches: 80 }`.
- [ ] No implementation changes — only new test file.

**Hints:**
1. Count branches: each `if` is 2 branches (true/false). Map them out before writing tests.
2. VIP (15%) + SUMMER20 (20%) = 35% — below cap. Employee (30%) + SUMMER20 (20%) = 50% — exactly at cap. Test both.
3. `vitest.config.ts`: `coverage: { provider: 'v8', thresholds: { lines: 80, branches: 80 } }`.
4. The "intentionally skipped" branches (e.g., `price === 0` edge) should be documented: `// Branch: price exactly 0 — same as negative, redundant to test separately`.

---

## Exercise 8 — Snapshot Test a React Component (Easy)

**Scenario:** Write snapshot tests for a `UserCard` React component to catch unintentional UI regressions.

**Requirements:**
- The `UserCard` component renders: avatar, name, email, role badge, and an optional "Admin" label.
- Write snapshot tests for: default state, with admin label, with missing avatar (fallback initials).
- Use Vitest + `@testing-library/react` with jsdom.
- When the component intentionally changes, update snapshots with `--updateSnapshot` — never manually edit snapshot files.
- Add one behavioral test (not a snapshot): clicking the "Edit" button calls an `onEdit` prop.

**Acceptance Criteria:**
- [ ] Three snapshot tests, each in its own `it` block with a descriptive name.
- [ ] Snapshot files are committed to the repository (`./__snapshots__/UserCard.test.tsx.snap`).
- [ ] The behavioral test uses `fireEvent.click` and `expect(onEdit).toHaveBeenCalledOnce()`.
- [ ] Snapshots use `toMatchInlineSnapshot` for small components or `toMatchSnapshot` for larger ones — choose appropriately.
- [ ] Snapshot tests pass on CI without regenerating (snapshots are committed).

**Hints:**
1. Setup: `import { render } from '@testing-library/react'`. Use `container.firstChild` or `asFragment()` as the snapshot subject.
2. `toMatchInlineSnapshot()` is preferred for components under ~20 lines of rendered HTML — the snapshot is inline in the test file and easier to review in PRs.
3. Mock images: provide a `src` string — jsdom does not load actual images, so the `src` attribute is sufficient for snapshot accuracy.
4. Update snapshots: `npx vitest --updateSnapshot`. Review the diff carefully before committing.

---

## Exercise 9 — Load Test an Endpoint with k6 (Medium)

**Scenario:** Define a k6 load test for the `GET /products` endpoint with explicit SLOs (Service Level Objectives). The test must fail if SLOs are not met.

**Requirements:**
- Ramp up to 500 virtual users over 1 minute, hold for 3 minutes, ramp down over 1 minute.
- SLOs: p95 response time < 200ms, p99 < 500ms, error rate < 0.1%.
- The k6 script must use `check` assertions and custom thresholds.
- Parametrize the base URL so the same script works in staging and production.
- Output a summary report (k6's default JSON output) suitable for CI artifact upload.

**Acceptance Criteria:**
- [ ] k6 script uses `stages` to define the ramp-up/hold/ramp-down pattern.
- [ ] `thresholds` are defined for `http_req_duration` (p95 and p99) and `http_req_failed`.
- [ ] The script exits with a non-zero code when any threshold is breached (k6 default behavior).
- [ ] Base URL is read from `__ENV.BASE_URL` with a fallback to `http://localhost:3000`.
- [ ] A GitHub Actions workflow runs the k6 test on push to `main` and uploads the summary JSON.

**Hints:**
1. k6 stages:
   ```javascript
   export const options = {
     stages: [
       { duration: '1m', target: 500 },
       { duration: '3m', target: 500 },
       { duration: '1m', target: 0 },
     ],
     thresholds: {
       'http_req_duration': ['p(95)<200', 'p(99)<500'],
       'http_req_failed': ['rate<0.001'],
     },
   };
   ```
2. Base URL: `const BASE_URL = __ENV.BASE_URL || 'http://localhost:3000';`.
3. CI: `k6 run --out json=report.json script.js` then `actions/upload-artifact` with `path: report.json`.
4. Add `check(res, { 'status 200': r => r.status === 200 })` to track check failures separately from HTTP errors.

---

## Exercise 10 — Fix a Flaky Test (Medium)

**Scenario:** The following test passes sometimes and fails other times. Diagnose the flakiness and fix it without removing any assertions.

**Given flaky test:**
```typescript
it('processes jobs in order', async () => {
  const results: number[] = [];
  const queue = new JobQueue();

  queue.add(() => new Promise(res => setTimeout(() => { results.push(1); res(undefined); }, 100)));
  queue.add(() => new Promise(res => setTimeout(() => { results.push(2); res(undefined); }, 50)));
  queue.add(() => new Promise(res => setTimeout(() => { results.push(3); res(undefined); }, 10)));

  await new Promise(res => setTimeout(res, 200));

  expect(results).toEqual([1, 2, 3]);
});
```

**Requirements:**
- Diagnose: explain in a code comment why the test is flaky.
- Fix the test so it is deterministic regardless of CPU load or timing variations.
- The fix must not use `setTimeout` for synchronization — use the queue's own API.
- If the `JobQueue` class needs a new method to support a clean wait, add it.

**Acceptance Criteria:**
- [ ] The fixed test passes 100 times in a row (`vitest --repeat 100`).
- [ ] A comment explains the root cause of the flakiness.
- [ ] The fix does not increase test duration unnecessarily.
- [ ] The assertion `expect(results).toEqual([1, 2, 3])` is preserved unchanged.
- [ ] The `JobQueue` class processes jobs sequentially (one at a time) — tests should verify this contract.

**Hints:**
1. Root cause: the test uses a magic timeout (`200ms`) to "wait for completion." Under high CPU load, jobs may not finish in time, causing a false failure.
2. Fix: add a `queue.drain(): Promise<void>` method that resolves when all enqueued jobs have completed. `await queue.drain()` replaces the `setTimeout` wait.
3. Sequential guarantee: if the queue processes jobs in order, job 2 starts only after job 1 completes — the completion order is deterministic regardless of each job's duration.
4. Fake timers: alternatively, use `vi.useFakeTimers()` to control `setTimeout` deterministically. But a `drain()` API is cleaner.

---

## Exercise 11 — Test a BullMQ Job Processor (Medium)

**Scenario:** Write tests for a BullMQ worker that processes `send-email` jobs. Tests must not require a real Redis instance.

**Requirements:**
- The worker function: `async function processEmailJob(job: Job<EmailPayload>): Promise<void>` calls `emailService.send(job.data)`.
- Tests cover: successful processing, `emailService` throws → job should fail with the error, malformed payload → job throws `ValidationError`.
- Use `ioredis-mock` or test the processor function directly (without BullMQ's `Worker` class).
- Mock `emailService` using Vitest spies.

**Acceptance Criteria:**
- [ ] Happy path: `emailService.send` is called with the correct payload.
- [ ] Email failure: calling `processEmailJob` with a failing `emailService` throws the email error (BullMQ marks it as failed).
- [ ] Invalid payload (missing `to` field): throws `ValidationError` before calling `emailService.send`.
- [ ] Tests do not start a real BullMQ `Worker` instance or connect to Redis.
- [ ] Test file runs in under 500ms (no real async delays).

**Hints:**
1. Test the processor function directly: `await processEmailJob(mockJob)` — no need to instantiate a `Worker`.
2. Mock job: `const mockJob = { data: { to: 'a@b.com', subject: 'Hi', body: 'Hello' }, id: '1' } as Job<EmailPayload>`.
3. Mock email service: `const emailService = { send: vi.fn().mockResolvedValue(undefined) }`. Inject via DI.
4. Validation: run `emailPayloadSchema.parse(job.data)` at the top of the processor. Zod throws `ZodError` — wrap in `ValidationError` for a cleaner API.

---

## Exercise 12 — Contract Tests with Pact (Hard)

**Scenario:** Your frontend (consumer) calls a `GET /users/:id` endpoint on the backend (provider). Write a Pact consumer contract test and a provider verification test.

**Requirements:**
- Consumer test: define the expected request/response for `GET /users/1` and generate a Pact file.
- Provider test: load the Pact file and verify the real backend satisfies the contract.
- The contract covers: happy path (200 + user object), not found (404 + error body).
- Pact files are stored in `./pacts/` and committed to the repository.
- Provider verification runs in CI after the backend build step.

**Acceptance Criteria:**
- [ ] Consumer test generates a `frontend-backend.json` Pact file in `./pacts/`.
- [ ] The Pact file defines interactions for both the 200 and 404 cases.
- [ ] Provider verification test starts the Fastify server and replays the Pact interactions against it.
- [ ] If the backend changes the response shape (e.g., renames `email` to `emailAddress`), the provider verification fails.
- [ ] A GitHub Actions job runs provider verification after `npm run build` succeeds.

**Hints:**
1. Consumer test setup: use `@pact-foundation/pact` `PactV3`. Define the interaction:
   ```typescript
   await provider.addInteraction({ uponReceiving: 'a request for user 1', withRequest: { method: 'GET', path: '/users/1' }, willRespondWith: { status: 200, body: like({ id: 1, name: string('Alice') }) } });
   ```
2. `like()` matcher: matches any value of the same type — avoids hard-coding test data.
3. Provider verification: use `Verifier` from `@pact-foundation/pact`. Point it at the running server and the `./pacts/` directory.
4. CI: run consumer tests in the frontend job (generates the Pact file), then run provider verification in the backend job using the committed Pact file.
