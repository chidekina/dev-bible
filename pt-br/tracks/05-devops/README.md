# Track 05 — DevOps

Containerização, CI/CD, servidores web, deploy e observabilidade. Construa a infraestrutura para entregar software com confiança.

**Tempo estimado:** 2–3 semanas

---

## Tópicos

1. [Docker Fundamentals](docker-fundamentals.md) — imagens, containers, Dockerfile, builds multi-stage
2. [Docker Compose](docker-compose.md) — services, volumes, networks, healthchecks, padrões de produção
3. [GitHub Actions & CI/CD](github-actions-cicd.md) — workflows, secrets, matrix builds, pipelines de deploy
4. [Nginx & Reverse Proxy](nginx-reverse-proxy.md) — configuração, proxy_pass, SSL, rate limiting, load balancing
5. [Deploy em VPS](vps-deployment.md) — configuração do servidor, Docker em VPS, Traefik, deploys sem downtime
6. [Hardening de VPS](vps-hardening.md) — SSH, firewall, fail2ban, segurança do Docker, ferramentas de auditoria
7. [Monitoramento & Observabilidade](monitoring-observability.md) — métricas, logs, traces, Prometheus, Grafana, Pino

---

## Ordem Sugerida

**1 → 2 → 5 → 6 → 4 → 3 → 7**

> Aprenda Docker antes de CI/CD — os pipelines fazem build e push de imagens Docker.

---

## Prática

→ [Labs de DevOps](../../exercises/devops-labs.md)
