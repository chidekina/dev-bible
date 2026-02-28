# Track 09 — Testing

Build confidence in your code through a disciplined testing strategy. Well-tested software ships faster because you refactor without fear and debug less in production.

**Estimated time:** 2–3 weeks

---

## Topics

1. [Testing Fundamentals](testing-fundamentals.md) — test pyramid, test doubles, what to test and why
2. [Unit Testing](unit-testing.md) — vitest, pure functions, isolation, coverage thresholds
3. [Integration Testing](integration-testing.md) — database tests, HTTP tests with supertest, test containers
4. [End-to-End Testing](e2e-testing.md) — Playwright, page object model, CI integration
5. [Test-Driven Development](test-driven-development.md) — red-green-refactor cycle, when TDD pays off
6. [Mocking & Stubs](mocking-stubs.md) — vi.mock, manual mocks, MSW for HTTP, avoiding over-mocking
7. [Performance Testing](performance-testing.md) — k6, load vs stress vs soak testing, SLOs

---

## Prerequisites

- Track 01 — Foundations (JavaScript/TypeScript)
- Track 03 — Backend (Fastify, databases) for integration testing chapters
- Track 02 — Frontend (React) for E2E testing chapters

---

## What You'll Build

- A full test suite for a CRUD API: unit tests for service logic, integration tests hitting a real SQLite DB, E2E tests with Playwright
- A TDD kata: implement a shopping cart with tests written first
- A load test script with k6 that validates a `/health` endpoint holds under 500 RPS
- A CI pipeline configuration running all three test layers on every pull request
