# Monitoring & Observability

## Overview

Observability is the ability to understand the internal state of a system from its external outputs. Monitoring tells you **when** something is wrong. Observability tells you **why**.

The three pillars of observability are:
- **Logs** — timestamped records of discrete events
- **Metrics** — numeric measurements aggregated over time
- **Traces** — end-to-end records of a request as it flows through services

Without observability, you are flying blind. You discover problems when users report them, you debug by guessing, and you have no baseline to compare against after a deploy. With it, you see problems before users do, you debug with evidence, and every deploy is validated by data.

This chapter covers practical observability for Node.js/TypeScript applications on a VPS: structured logging with Pino, metrics with Prometheus and Grafana, health checks, alerting, and distributed tracing with OpenTelemetry.

---

## Prerequisites

- A running application (Node.js/TypeScript preferred)
- Docker and Docker Compose
- Basic understanding of HTTP and JSON

---

## Core Concepts

### Logs

Logs are the raw stream of events your application produces. The key shift from `console.log` to production-grade logging:

| `console.log` | Structured logging (Pino) |
|--------------|--------------------------|
| Unstructured text | JSON with consistent fields |
| No severity levels | `info`, `warn`, `error`, `debug` |
| No context | `requestId`, `userId`, `traceId` |
| Hard to search | Easily filtered and aggregated |
| Synchronous | Asynchronous, non-blocking |

### Metrics

Metrics are numeric measurements over time. Four golden signals:

| Signal | Question | Example metric |
|--------|----------|---------------|
| **Latency** | How long do requests take? | `p50`, `p95`, `p99` response time |
| **Traffic** | How many requests per second? | `http_requests_total` |
| **Errors** | What fraction of requests fail? | `http_errors_total / http_requests_total` |
| **Saturation** | How full is the system? | CPU %, memory %, queue depth |

### Traces

A trace records the journey of a single request through your system — from the initial HTTP call through service calls, database queries, and external API calls. Traces are composed of **spans**, each representing one unit of work.

### Health Checks

Health endpoints let orchestrators (Docker, Kubernetes, load balancers) know if your service is ready to receive traffic:

```
GET /health → 200 OK (service is healthy)
GET /ready  → 200 OK (service is ready for traffic)
GET /metrics → 200 (Prometheus scrape endpoint)
```

---

## Hands-On Examples

### Example 1: Structured Logging with Pino

```bash
npm install pino pino-pretty
npm install -D @types/pino
```

`src/lib/logger.ts`:
```typescript
import pino from 'pino';

const isDev = process.env.NODE_ENV !== 'production';

export const logger = pino({
  level: process.env.LOG_LEVEL ?? 'info',

  // In development, use pino-pretty for human-readable output
  ...(isDev && {
    transport: {
      target: 'pino-pretty',
      options: {
        colorize: true,
        translateTime: 'SYS:standard',
        ignore: 'pid,hostname',
      },
    },
  }),

  // Standard fields added to every log entry
  base: {
    service: process.env.SERVICE_NAME ?? 'myapp',
    version: process.env.APP_VERSION ?? 'dev',
    env: process.env.NODE_ENV ?? 'development',
  },

  // Redact sensitive fields (prevent secrets in logs)
  redact: {
    paths: ['*.password', '*.token', '*.secret', '*.authorization', 'req.headers.cookie'],
    censor: '[REDACTED]',
  },
});
```

Usage throughout the application:
```typescript
import { logger } from './lib/logger.js';

// Structured fields first, message second
logger.info({ userId: '123', action: 'login' }, 'User logged in');
logger.error({ err, requestId: req.id }, 'Failed to process payment');
logger.warn({ endpoint: '/api/v1/old', replacement: '/api/v2/new' }, 'Deprecated endpoint called');

// Child loggers inherit parent context
const requestLogger = logger.child({ requestId: 'req_abc123' });
requestLogger.info('Processing request');    // includes requestId automatically
requestLogger.info({ userId: '456' }, 'Found user');  // includes both
```

### Example 2: Request Logging Middleware (Fastify)

```typescript
import Fastify from 'fastify';
import { logger } from './lib/logger.js';
import { randomUUID } from 'crypto';

const app = Fastify({
  loggerInstance: logger,
  genReqId: () => randomUUID(),
});

// Custom serializers for cleaner logs
app.addHook('onRequest', async (req) => {
  req.log.info({
    method: req.method,
    url: req.url,
    userAgent: req.headers['user-agent'],
    ip: req.ip,
  }, 'Incoming request');
});

app.addHook('onResponse', async (req, reply) => {
  req.log.info({
    statusCode: reply.statusCode,
    responseTime: reply.elapsedTime,
  }, 'Request completed');
});

app.addHook('onError', async (req, reply, error) => {
  req.log.error({
    err: error,
    statusCode: reply.statusCode,
  }, 'Request error');
});
```

### Example 3: Health Check Endpoint

```typescript
import type { FastifyInstance } from 'fastify';
import { prisma } from './lib/db.js';

export async function healthRoutes(app: FastifyInstance) {
  // Liveness — is the process running?
  app.get('/health', async () => {
    return {
      status: 'ok',
      uptime: process.uptime(),
      timestamp: new Date().toISOString(),
    };
  });

  // Readiness — can the app serve traffic? (checks dependencies)
  app.get('/ready', async (req, reply) => {
    const checks: Record<string, { status: string; latencyMs?: number }> = {};

    // Check database
    const dbStart = Date.now();
    try {
      await prisma.$queryRaw`SELECT 1`;
      checks.database = { status: 'ok', latencyMs: Date.now() - dbStart };
    } catch (err) {
      checks.database = { status: 'error' };
    }

    const allHealthy = Object.values(checks).every((c) => c.status === 'ok');

    reply.code(allHealthy ? 200 : 503);
    return {
      status: allHealthy ? 'ok' : 'degraded',
      checks,
    };
  });
}
```

### Example 4: Prometheus Metrics

```bash
npm install prom-client
```

`src/lib/metrics.ts`:
```typescript
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

export const registry = new Registry();

// Collect default Node.js metrics (memory, CPU, event loop, etc.)
collectDefaultMetrics({ register: registry });

// Custom application metrics
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total HTTP requests',
  labelNames: ['method', 'route', 'status_code'],
  registers: [registry],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'HTTP request duration in seconds',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [registry],
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Number of active connections',
  registers: [registry],
});

export const dbQueryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Database query duration in seconds',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
  registers: [registry],
});
```

Fastify metrics plugin:
```typescript
// src/plugins/metrics.ts
import type { FastifyInstance } from 'fastify';
import fp from 'fastify-plugin';
import { registry, httpRequestsTotal, httpRequestDuration } from '../lib/metrics.js';

export const metricsPlugin = fp(async (app: FastifyInstance) => {
  // Record metrics on every request
  app.addHook('onResponse', async (req, reply) => {
    const route = req.routeOptions?.url ?? 'unknown';

    // Skip metrics endpoint itself
    if (route === '/metrics') return;

    const labels = {
      method: req.method,
      route,
      status_code: String(reply.statusCode),
    };

    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, reply.elapsedTime / 1000);
  });

  // Expose metrics endpoint for Prometheus to scrape
  app.get('/metrics', async (req, reply) => {
    reply.header('Content-Type', registry.contentType);
    return registry.metrics();
  });
});
```

### Example 5: Prometheus + Grafana Stack

`monitoring/compose.yml`:
```yaml
name: monitoring

services:
  prometheus:
    image: prom/prometheus:v2.48.0
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
      - prometheusdata:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
      - '--web.enable-lifecycle'
    ports:
      - "127.0.0.1:9090:9090"
    restart: unless-stopped

  grafana:
    image: grafana/grafana:10.2.0
    environment:
      GF_SECURITY_ADMIN_PASSWORD: ${GRAFANA_PASSWORD:-admin}
      GF_USERS_ALLOW_SIGN_UP: "false"
      GF_SERVER_ROOT_URL: https://grafana.myapp.com
    volumes:
      - grafanadata:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning:ro
    ports:
      - "127.0.0.1:3001:3000"
    restart: unless-stopped

  alertmanager:
    image: prom/alertmanager:v0.26.0
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml:ro
    ports:
      - "127.0.0.1:9093:9093"
    restart: unless-stopped

  node-exporter:
    image: prom/node-exporter:v1.7.0
    command:
      - '--path.procfs=/host/proc'
      - '--path.sysfs=/host/sys'
      - '--collector.filesystem.ignored-mount-points=^/(sys|proc|dev|host|etc)($$|/)'
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    ports:
      - "127.0.0.1:9100:9100"
    restart: unless-stopped

volumes:
  prometheusdata:
  grafanadata:
```

`monitoring/prometheus.yml`:
```yaml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - 'alerts/*.yml'

scrape_configs:
  - job_name: 'prometheus'
    static_configs:
      - targets: ['localhost:9090']

  - job_name: 'myapp'
    static_configs:
      - targets: ['host.docker.internal:3000']  # or container name
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Example 6: Alerts Configuration

`monitoring/alerts/app.yml`:
```yaml
groups:
  - name: application
    rules:
      - alert: HighErrorRate
        expr: |
          (
            sum(rate(http_requests_total{status_code=~"5.."}[5m]))
            /
            sum(rate(http_requests_total[5m]))
          ) > 0.05
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ humanizePercentage $value }} over the last 5 minutes"

      - alert: SlowResponses
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 response time above 1 second"
          description: "95th percentile response time is {{ humanizeDuration $value }}"

      - alert: ServiceDown
        expr: up{job="myapp"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Application is down"
          description: "myapp has been down for more than 1 minute"

  - name: infrastructure
    rules:
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High memory usage"
          description: "Memory usage is above 90%: {{ humanizePercentage $value }}"

      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Disk space running low"
          description: "Disk {{ $labels.device }} is {{ humanizePercentage $value }} full"
```

`monitoring/alertmanager.yml`:
```yaml
global:
  resolve_timeout: 5m

route:
  group_by: ['alertname']
  group_wait: 10s
  group_interval: 10m
  repeat_interval: 1h
  receiver: 'default'

receivers:
  - name: 'default'
    webhook_configs:
      - url: 'https://hooks.slack.com/services/YOUR/WEBHOOK/URL'
        send_resolved: true
```

---

## Common Patterns & Best Practices

### Correlation IDs

Propagate a request ID through all logs to trace a request across your system:

```typescript
import { randomUUID } from 'crypto';
import { AsyncLocalStorage } from 'async_hooks';

const requestContext = new AsyncLocalStorage<{ requestId: string }>();

// Middleware: attach requestId to every request
app.addHook('onRequest', async (req) => {
  const requestId = req.headers['x-request-id'] as string ?? randomUUID();
  req.id = requestId;
  requestContext.run({ requestId }, () => {});
});

// Utility: get requestId anywhere in the call stack
export function getRequestId(): string | undefined {
  return requestContext.getStore()?.requestId;
}
```

### Log Levels Strategy

```typescript
// Use levels consistently across your team
logger.debug({ query, params }, 'SQL query');           // detailed debugging
logger.info({ userId, action }, 'Business event');     // normal operations
logger.warn({ rateLimitRemaining }, 'Rate limit low'); // something off but not broken
logger.error({ err, context }, 'Operation failed');   // requires attention
logger.fatal({ err }, 'Unrecoverable error');          // process about to exit
```

Set `LOG_LEVEL=debug` in development, `LOG_LEVEL=info` in production. Never ship debug logs to production.

### Sentry for Error Tracking

```bash
npm install @sentry/node
```

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,   // sample 10% of requests for performance monitoring
});

// In Fastify error handler
app.setErrorHandler((error, req, reply) => {
  Sentry.captureException(error, {
    extra: {
      requestId: req.id,
      url: req.url,
      method: req.method,
    },
  });
  logger.error({ err: error, requestId: req.id }, 'Unhandled error');
  reply.code(500).send({ error: 'Internal server error' });
});
```

### Grafana Dashboard as Code

Provision dashboards automatically so they survive container restarts:

`monitoring/grafana/provisioning/dashboards/dashboard.json`:
```json
{
  "title": "Application Overview",
  "panels": [
    {
      "title": "Request Rate",
      "type": "graph",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total[5m])) by (route)",
          "legendFormat": "{{route}}"
        }
      ]
    },
    {
      "title": "Error Rate",
      "type": "stat",
      "targets": [
        {
          "expr": "sum(rate(http_requests_total{status_code=~'5..'}[5m])) / sum(rate(http_requests_total[5m]))"
        }
      ]
    },
    {
      "title": "P95 Latency",
      "type": "graph",
      "targets": [
        {
          "expr": "histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))",
          "legendFormat": "p95 - {{route}}"
        }
      ]
    }
  ]
}
```

---

## Anti-Patterns to Avoid

### Logging Sensitive Data

```typescript
// BAD — password and token in logs
logger.info({ user: { email, password, authToken } }, 'User authenticated');

// GOOD — redact before logging, or log only what's needed
logger.info({ userId: user.id, email: user.email }, 'User authenticated');
```

### Using console.log in Production

`console.log` is synchronous, unstructured, and unmeasurable. Replace it with Pino at the start of every project.

### Alerting on Symptoms, Not Signals

```yaml
# BAD — too noisy, fires on brief spikes
expr: http_errors_total > 0

# GOOD — fires only on sustained error rate
expr: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
for: 2m   # must be true for 2 continuous minutes
```

### Ignoring Cardinality

Each unique combination of label values creates a new time series. High-cardinality labels explode your metrics storage:

```typescript
// BAD — userId as a label creates millions of time series
httpRequests.inc({ userId: req.user.id, route: '/api/users' });

// GOOD — only low-cardinality dimensions as labels
httpRequests.inc({ route: '/api/users', status: '200' });
```

### No Log Retention Limits

Without limits, logs fill your disk. Configure Docker log rotation and/or ship logs to an external service.

---

## Debugging & Troubleshooting

### Finding Errors in Logs

```bash
# Docker container logs
docker compose logs api | jq '. | select(.level >= 50)'   # errors only (pino level 50=error)

# Filter by time
docker compose logs api --since 1h | grep '"level":50'

# Search by request ID
docker compose logs api | grep 'req_abc123'

# Live error monitoring
docker compose logs -f api | jq '. | select(.level >= 40)'  # warn + error
```

### Prometheus Query Examples

```promql
# Request rate (requests per second)
sum(rate(http_requests_total[5m]))

# Error rate as percentage
100 * sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# P95 latency by route
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))

# Memory usage percentage
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# CPU usage
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# Disk usage
100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

### Check if Prometheus is Scraping

```bash
# Check targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job, health, lastScrape}'

# Check specific metric exists
curl -s 'http://localhost:9090/api/v1/query?query=http_requests_total' | jq '.data.result'
```

---

## Real-World Scenarios

### Scenario 1: Debugging a Production Incident

```bash
# 1. Check if service is up
curl https://myapp.com/health

# 2. Check error rate in Prometheus
# http://localhost:9090 → query: rate(http_requests_total{status_code=~"5.."}[5m])

# 3. Find the failing requests in logs
docker compose logs api --since 30m | jq '. | select(.level == 50)' | head -20

# 4. Get the request ID of a failing request, then trace it
docker compose logs api | grep '"requestId":"req_xyz"'

# 5. Check if database is the culprit
docker compose exec db psql -U app -c "SELECT * FROM pg_stat_activity WHERE state = 'active';"
```

### Scenario 2: Setting Up Uptime Monitoring

Use an external uptime monitor (UptimeRobot, Betterstack) to alert on downtime independently of your internal stack:

```
Monitor type: HTTP(S)
URL: https://myapp.com/health
Interval: 60 seconds
Alert contacts: your email/phone
Expected status: 200
```

This catches cases where your entire VPS goes down (and Prometheus along with it).

### Scenario 3: Weekly Performance Report

```bash
# Pull key metrics for the week
curl -s "http://localhost:9090/api/v1/query_range" \
  --data-urlencode 'query=avg(rate(http_request_duration_seconds_sum[1h]) / rate(http_request_duration_seconds_count[1h]))' \
  --data-urlencode "start=$(date -d '7 days ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode 'step=3600' \
  | jq '.data.result[0].values | map(.[1] | tonumber) | add / length'
```

---

## Further Reading

- [Pino Documentation](https://getpino.io/)
- [Prometheus Documentation](https://prometheus.io/docs/introduction/overview/)
- [Grafana Documentation](https://grafana.com/docs/)
- [OpenTelemetry for Node.js](https://opentelemetry.io/docs/instrumentation/js/)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [PromQL Tutorial](https://promlabs.com/promql-cheat-sheet/)

---

## Summary

| Tool / Concept | Purpose |
|---------------|---------|
| Pino | Structured, fast JSON logging |
| Log levels | debug < info < warn < error < fatal |
| Request ID | Correlate all logs for a single request |
| `/health` endpoint | Liveness check for orchestrators |
| `/ready` endpoint | Readiness check — validates dependencies |
| `/metrics` endpoint | Prometheus scrape endpoint |
| `prom-client` | Instrument Node.js with Prometheus metrics |
| Prometheus | Time-series metrics collection and alerting |
| Grafana | Visualization and dashboards |
| Alertmanager | Route alerts to Slack, email, PagerDuty |
| Node Exporter | Expose Linux OS metrics to Prometheus |
| Sentry | Error tracking and aggregation |
| Four golden signals | Latency, Traffic, Errors, Saturation |

You cannot improve what you cannot measure. Instrument from the start — adding observability to an existing system is 10x harder than building it in from day one.
