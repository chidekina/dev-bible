# Track 08 — System Design

Learn to architect systems that survive real traffic, hardware failures, and team growth. System design bridges the gap between writing code and building infrastructure that lasts.

**Estimated time:** 3–4 weeks

---

## Topics

1. [CAP Theorem](cap-theorem.md) — consistency, availability, partition tolerance and the trade-offs that define distributed systems
2. [Load Balancing](load-balancing.md) — algorithms, health checks, sticky sessions, layer 4 vs layer 7
3. [Caching Strategies](caching-strategies.md) — cache-aside, write-through, write-behind, TTL, eviction policies
4. [Database Scaling](database-scaling.md) — replication, sharding, read replicas, connection pooling
5. [Message Queues](message-queues.md) — producers, consumers, dead-letter queues, at-least-once delivery
6. [API Gateways](api-gateways.md) — routing, rate limiting, auth offloading, request transformation
7. [CDN](cdn.md) — edge caching, cache invalidation, origin pull vs push, geo-routing
8. [Distributed Systems](distributed-systems.md) — consensus, clock skew, idempotency, saga pattern
9. [Design Interviews](design-interviews.md) — framework, estimation, common question walkthroughs

---

## Prerequisites

- Track 03 — Backend (databases, REST APIs)
- Track 04 — Architecture (patterns, microservices)
- Track 05 — DevOps (containers, networking basics)

---

## What You'll Build

- A multi-tier architecture diagram for a URL shortener supporting 100M requests/day
- A caching layer using Redis with cache-aside pattern and TTL management in Node.js
- A message queue consumer with retry logic and dead-letter queue handling (BullMQ)
- A database read-replica routing strategy for a Prisma-based application
- A full system design walkthrough document for a real-time chat application
