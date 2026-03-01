# Track 10 — Performance

Meça antes de otimizar. Performance é entender onde o tempo e os bytes estão sendo gastos e fazer escolhas deliberadas para melhorar a experiência do usuário e reduzir os custos de infraestrutura.

**Tempo estimado:** 2–3 semanas

---

## Tópicos

1. [Métricas de Performance Web](web-performance-metrics.md) — Core Web Vitals, LCP, CLS, INP, TTFB e como medi-los
2. [Otimização de JavaScript](javascript-optimization.md) — bundle splitting, tree-shaking, lazy loading, evitando layout thrashing
3. [Otimização de Queries no Banco de Dados](database-query-optimization.md) — EXPLAIN ANALYZE, indexes, queries N+1, batching de queries
4. [HTTP Caching](caching-http.md) — Cache-Control, ETags, stale-while-revalidate, service workers
5. [Ferramentas de Profiling](profiling-tools.md) — Chrome DevTools, Node.js --prof, clinic.js, flame graphs com 0x
6. [Otimização de Imagens](image-optimization.md) — WebP/AVIF, imagens responsivas, lazy loading, entrega via CDN

---

## Pré-requisitos

- Track 02 — Frontend (React, bundlers)
- Track 03 — Backend (bancos de dados, HTTP)
- Track 05 — DevOps (conceitos de CDN do Track 08 também ajudam)

---

## O Que Você Vai Construir

- Uma linha de base com auditoria Lighthouse e um relatório antes/depois aplicando otimizações em uma página Next.js
- Uma estratégia de index para uma tabela Postgres com queries lentas, validada com EXPLAIN ANALYZE
- Um flame graph Node.js identificando um hot path em um handler Fastify, com o gargalo resolvido
- Uma estratégia de HTTP caching para uma REST API com ETags e headers Cache-Control
- Um pipeline de imagens usando Sharp que converte uploads para WebP com tamanhos responsivos
