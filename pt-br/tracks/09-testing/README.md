# Track 09 — Testes

Construa confiança no seu código com uma estratégia de testes disciplinada. Software bem testado é entregue mais rápido porque você refatora sem medo e depura menos em produção.

**Tempo estimado:** 2–3 semanas

---

## Tópicos

1. [Fundamentos de Testes](testing-fundamentals.md) — pirâmide de testes, test doubles, o que testar e por quê
2. [Testes Unitários](unit-testing.md) — vitest, funções puras, isolamento, thresholds de coverage
3. [Testes de Integração](integration-testing.md) — testes com banco de dados, testes HTTP com supertest, test containers
4. [Testes End-to-End](e2e-testing.md) — Playwright, page object model, integração com CI
5. [Desenvolvimento Guiado por Testes](test-driven-development.md) — ciclo red-green-refactor, quando o TDD vale a pena
6. [Mocking & Stubs](mocking-stubs.md) — vi.mock, mocks manuais, MSW para HTTP, evitando over-mocking
7. [Testes de Performance](performance-testing.md) — k6, load vs stress vs soak testing, SLOs

---

## Pré-requisitos

- Track 01 — Fundamentos (JavaScript/TypeScript)
- Track 03 — Backend (Fastify, bancos de dados) para os capítulos de testes de integração
- Track 02 — Frontend (React) para os capítulos de testes E2E

---

## O que você vai construir

- Uma suíte de testes completa para uma API CRUD: testes unitários para a lógica de serviço, testes de integração contra um SQLite real, testes E2E com Playwright
- Uma kata de TDD: implemente um carrinho de compras escrevendo os testes primeiro
- Um script de load test com k6 que valida se um endpoint `/health` aguenta 500 RPS
- Uma configuração de pipeline de CI executando as três camadas de testes em todo pull request
