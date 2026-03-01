# Track 08 — System Design

Aprenda a arquitetar sistemas que sobrevivem a tráfego real, falhas de hardware e crescimento de time. System design é a ponte entre escrever código e construir infraestrutura que dura.

**Tempo estimado:** 3–4 semanas

---

## Tópicos

1. [Teorema CAP](cap-theorem.md) — consistência, disponibilidade, tolerância a partições e os trade-offs que definem sistemas distribuídos
2. [Load Balancing](load-balancing.md) — algoritmos, health checks, sticky sessions, camada 4 vs camada 7
3. [Caching Strategies](caching-strategies.md) — cache-aside, write-through, write-behind, TTL, políticas de evição
4. [Database Scaling](database-scaling.md) — replicação, sharding, read replicas, connection pooling
5. [Message Queues](message-queues.md) — producers, consumers, dead-letter queues, entrega at-least-once
6. [API Gateways](api-gateways.md) — roteamento, rate limiting, offload de autenticação, transformação de requisições
7. [CDN](cdn.md) — edge caching, invalidação de cache, origin pull vs push, geo-routing
8. [Sistemas Distribuídos](distributed-systems.md) — consenso, clock skew, idempotência, padrão saga
9. [Entrevistas de Design](design-interviews.md) — framework, estimativas, resoluções das perguntas mais comuns

---

## Pré-requisitos

- Track 03 — Backend (bancos de dados, APIs REST)
- Track 04 — Architecture (padrões, microsserviços)
- Track 05 — DevOps (containers, fundamentos de rede)

---

## O que você vai construir

- Um diagrama de arquitetura multi-tier para um encurtador de URLs suportando 100M requisições/dia
- Uma camada de caching com Redis usando o padrão cache-aside com gerenciamento de TTL em Node.js
- Um consumer de message queue com lógica de retry e tratamento de dead-letter queue (BullMQ)
- Uma estratégia de roteamento para read replica em uma aplicação baseada em Prisma
- Um documento completo de system design walkthrough para uma aplicação de chat em tempo real
