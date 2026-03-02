# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.0.0/), and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

---

## [2.3.0] — 2026-03-02

### Added
- Track 14 — Mobile: 3 new chapters (mobile-testing, mobile-security, native-device-apis) in EN + PT-BR
- Deepened 6 existing mobile chapters (pwa-fundamentals, react-native-basics, react-native-advanced, mobile-navigation, mobile-state-management, mobile-deployment) — +150–280 lines each covering: background sync, share target API, Hermes engine, Metro config, Reanimated 3, New Architecture, Skia, typed navigation, universal links, bottom sheet, WatermelonDB, background fetch, conflict resolution, internal distribution, staged rollouts, OTA rollback, EAS secrets

### Stats
- Track 14 chapters: 6 → 9
- Total chapters: 115 → 124
- Total files: 280 → 298

---

## [2.2.0] - 2026-03-02

### Added

#### Track 14: Mobile (7 topics — EN)
- PWA Fundamentals (service workers, Workbox, offline-first, push notifications)
- React Native Basics (Expo, core components, StyleSheet, navigation)
- React Native Advanced (Reanimated, Gesture Handler, native modules, Hermes)
- Mobile Navigation (stack/tab/drawer, auth flow, deep links, universal links)
- Mobile State Management (Zustand + MMKV, React Query offline, sync patterns)
- Mobile Deployment (EAS Build/Submit/Update, app signing, CI/CD)

#### Reference Cheatsheets — 4 new (EN)
- React cheatsheet (hooks, context, compound components, TypeScript, performance)
- PostgreSQL cheatsheet (JSONB, full-text search, partitioning, replication, pg_stat)
- Regex cheatsheet (character classes, lookaheads, named groups, common patterns)
- Bash scripting cheatsheet (functions, traps, retry loops, parallel execution)

### Changed
- README updated: 15 tracks, 121 EN chapters, 16 cheatsheets, 300 total files

---

## [2.1.0] - 2026-03-02

### Added

#### Reference Cheatsheets — 4 new (EN + PT-BR)
- Kubernetes cheatsheet (kubectl, Helm, RBAC, troubleshooting)
- Terraform cheatsheet (workflow, state, loops, modules, backends)
- Helm cheatsheet (repos, install/upgrade, chart dev, hooks, helmfile)
- AI/LLM cheatsheet (model comparison, SDK quick-ref, RAG, prompt patterns, cost)

#### Exercises — 7 new files (EN + PT-BR)
- Security challenges (12 exercises: bcrypt, JWT, CSRF, rate limiting, RBAC, etc.)
- System Design challenges (10 katas: rate limiter, fanout, circuit breaker, etc.)
- Testing challenges (12 exercises: unit, integration, E2E, TDD, MSW, k6, Pact)
- Performance challenges (10 exercises: profiling, N+1, caching, Core Web Vitals)
- Accessibility challenges (10 exercises: axe-core, ARIA, focus trap, jest-axe)
- Kubernetes challenges (10 exercises: manifests, probes, HPA, RBAC, Helm)
- AI/LLM challenges (10 exercises: RAG, tool use, streaming, model routing)

### Changed
- README badges updated: 115 topics, 280 files, 12 cheatsheets, 13 exercise files
- Reference section expanded with links to all 12 cheatsheets
- Exercises section expanded with links to all 13 exercise files

---

## [2.0.0] - 2026-03-02

### Added

#### Track 07: Security (8 topics — EN + PT-BR)
- OWASP Top 10
- Authentication & Authorization
- Secrets Management
- HTTPS & TLS
- CORS & CSRF
- SQL Injection & XSS
- Security Headers
- Dependency Auditing

#### Track 08: System Design (9 topics — EN + PT-BR)
- CAP Theorem
- Load Balancing
- Caching Strategies
- Database Scaling
- Message Queues
- API Gateways
- CDN
- Distributed Systems
- Design Interviews

#### Track 09: Testing (7 topics — EN + PT-BR)
- Testing Fundamentals
- Unit Testing
- Integration Testing
- E2E Testing
- Test-Driven Development
- Mocking & Stubs
- Performance Testing

#### Track 10: Performance (6 topics — EN + PT-BR)
- Web Performance Metrics
- JavaScript Optimization
- Database Query Optimization
- HTTP Caching
- Profiling Tools
- Image Optimization

#### Track 11: Accessibility (5 topics — EN + PT-BR)
- WCAG Standards
- Semantic HTML
- ARIA Roles
- Keyboard Navigation
- Testing Accessibility

#### Track 12: Kubernetes (7 topics — EN + PT-BR)
- Kubernetes Concepts
- kubectl Basics
- Pods & Deployments
- Services & Ingress
- ConfigMaps & Secrets
- Helm Basics
- K8s in Production

#### Track 13: AI/LLM Integration (7 topics — EN + PT-BR)
- LLM Fundamentals
- Prompt Engineering
- RAG & Embeddings
- Tool Use & Function Calling
- AI Integration Patterns
- Fine-Tuning Concepts
- AI Cost Optimization

### Changed
- README updated with all 14 tracks, new learning paths (Security, AI/LLM, Full-Stack Senior)
- Badge counts updated: 110 topics per language, 248 total files

---

## [1.0.0] - 2026-02-28

### Added

#### Track 00: CS Fundamentals (16 topics — EN + PT-BR)
- Big O Notation
- Arrays & Strings
- Linked Lists
- Stacks & Queues
- Hashmaps & Sets
- Trees & Binary Search
- Graphs
- Heaps
- Tries
- Segment Trees (Advanced)
- Sorting Algorithms
- Searching Algorithms
- Recursion & Backtracking
- Dynamic Programming
- Greedy Algorithms
- Common Patterns (sliding window, two pointers, fast/slow, etc.)

#### Track 01: Foundations (6 topics — EN + PT-BR)
- How the Web Works
- HTML & CSS
- JavaScript
- TypeScript
- Linux & Shell
- Git

#### Track 02: Frontend (8 topics — EN + PT-BR)
- React Hooks
- React Patterns
- React Performance
- Next.js App Router
- Next.js SSR/SSG
- Testing (Unit)
- Testing (E2E)
- Core Web Vitals

#### Track 03: Backend (9 topics — EN + PT-BR)
- Node.js Internals
- REST API Design
- GraphQL
- Auth (JWT & OAuth)
- Databases (SQL)
- Databases (NoSQL)
- ORM Patterns
- Caching & Redis
- Message Queues

#### Track 04: Architecture (9 topics — EN + PT-BR)
- SOLID Principles
- Clean Architecture
- Hexagonal Architecture
- Domain-Driven Design
- Design Patterns (Creational)
- Design Patterns (Structural)
- Design Patterns (Behavioral)
- Microservices vs Monolith
- API Versioning & Contracts

#### Track 05: DevOps (7 topics — EN + PT-BR)
- Docker Fundamentals
- Docker Compose
- GitHub Actions CI/CD
- Nginx Reverse Proxy
- VPS Deployment
- VPS Hardening
- Monitoring & Observability

#### Track 06: Cloud (11 topics — EN + PT-BR)
- Cloud Concepts (IaaS/PaaS/SaaS/FaaS)
- AWS Core Services
- AWS Architecture
- AWS Cost Optimization
- Azure Core Services
- Azure Architecture
- GCP Core Services
- GCP Architecture
- Cloud Comparison (AWS vs Azure vs GCP)
- Cloud Security
- Cloud Cost Optimization

#### Reference Cheatsheets (8 files — EN + PT-BR)
- TypeScript Cheatsheet
- SQL Cheatsheet
- Docker Cheatsheet
- Git Cheatsheet
- Linux Cheatsheet
- HTTP Status Codes
- Big O Cheatsheet
- Cloud Services Map

#### Exercises & Labs (6 files — EN + PT-BR)
- CS Challenges (30 algorithmic problems with solutions)
- Frontend Challenges (15 practical React/browser challenges)
- Backend Challenges (15 Fastify/Node.js/SQL challenges)
- Architecture Katas (10 system design challenges)
- DevOps Labs (8 hands-on labs)
- Cloud Labs (8 multi-cloud deployment labs)

### Stats
- 160 total files
- ~75,000 lines of content
- Bilingual: English + Portuguese (PT-BR)
- Every topic follows the 10-section template
