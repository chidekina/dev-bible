# Monitoramento & Observabilidade

## Visão Geral

Observabilidade é a capacidade de entender o estado interno de um sistema a partir de suas saídas externas. Monitoramento diz **quando** algo está errado. Observabilidade diz **por quê**.

Os três pilares da observabilidade são:
- **Logs** — registros com timestamp de eventos discretos
- **Métricas** — medições numéricas agregadas ao longo do tempo
- **Traces** — registros end-to-end de uma requisição ao longo dos services

Sem observabilidade, você está voando às cegas. Você descobre problemas quando os usuários os relatam, depura por intuição e não tem baseline para comparar após um deploy. Com ela, você vê problemas antes dos usuários, depura com evidências e cada deploy é validado por dados.

Este capítulo cobre observabilidade prática para aplicações Node.js/TypeScript em VPS: logging estruturado com Pino, métricas com Prometheus e Grafana, health checks, alertas e tracing distribuído com OpenTelemetry.

---

## Pré-requisitos

- Uma aplicação rodando (Node.js/TypeScript de preferência)
- Docker e Docker Compose
- Entendimento básico de HTTP e JSON

---

## Conceitos Fundamentais

### Logs

Logs são o fluxo bruto de eventos que sua aplicação produz. A mudança principal do `console.log` para logging de nível produção:

| `console.log` | Logging estruturado (Pino) |
|--------------|--------------------------|
| Texto não estruturado | JSON com campos consistentes |
| Sem níveis de severidade | `info`, `warn`, `error`, `debug` |
| Sem contexto | `requestId`, `userId`, `traceId` |
| Difícil de pesquisar | Facilmente filtrado e agregado |
| Síncrono | Assíncrono, non-blocking |

### Métricas

Métricas são medições numéricas ao longo do tempo. Quatro sinais dourados:

| Sinal | Pergunta | Exemplo de métrica |
|-------|---------|-------------------|
| **Latência** | Quanto tempo as requisições levam? | Tempo de resposta `p50`, `p95`, `p99` |
| **Tráfego** | Quantas requisições por segundo? | `http_requests_total` |
| **Erros** | Que fração das requisições falha? | `http_errors_total / http_requests_total` |
| **Saturação** | Quão cheio está o sistema? | CPU %, memória %, profundidade de fila |

### Traces

Um trace registra a jornada de uma única requisição pelo seu sistema — desde a chamada HTTP inicial, passando por chamadas de service, queries de banco e chamadas de API externas. Traces são compostos de **spans**, cada um representando uma unidade de trabalho.

### Health Checks

Endpoints de health permitem que orquestradores (Docker, Kubernetes, load balancers) saibam se seu service está pronto para receber tráfego:

```
GET /health → 200 OK (service está saudável)
GET /ready  → 200 OK (service está pronto para tráfego)
GET /metrics → 200 (endpoint de scrape do Prometheus)
```

---

## Exemplos Práticos

### Exemplo 1: Logging Estruturado com Pino

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

  // Em desenvolvimento, usar pino-pretty para saída legível por humanos
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

  // Campos padrão adicionados a cada entrada de log
  base: {
    service: process.env.SERVICE_NAME ?? 'myapp',
    version: process.env.APP_VERSION ?? 'dev',
    env: process.env.NODE_ENV ?? 'development',
  },

  // Redigir campos sensíveis (previne secrets nos logs)
  redact: {
    paths: ['*.password', '*.token', '*.secret', '*.authorization', 'req.headers.cookie'],
    censor: '[REDACTED]',
  },
});
```

Uso ao longo da aplicação:
```typescript
import { logger } from './lib/logger.js';

// Campos estruturados primeiro, mensagem depois
logger.info({ userId: '123', action: 'login' }, 'Usuário logou');
logger.error({ err, requestId: req.id }, 'Falha ao processar pagamento');
logger.warn({ endpoint: '/api/v1/old', replacement: '/api/v2/new' }, 'Endpoint depreciado chamado');

// Child loggers herdam o contexto do pai
const requestLogger = logger.child({ requestId: 'req_abc123' });
requestLogger.info('Processando requisição');    // inclui requestId automaticamente
requestLogger.info({ userId: '456' }, 'Usuário encontrado');  // inclui ambos
```

### Exemplo 2: Middleware de Log de Requisições (Fastify)

```typescript
import Fastify from 'fastify';
import { logger } from './lib/logger.js';
import { randomUUID } from 'crypto';

const app = Fastify({
  loggerInstance: logger,
  genReqId: () => randomUUID(),
});

// Serializers customizados para logs mais limpos
app.addHook('onRequest', async (req) => {
  req.log.info({
    method: req.method,
    url: req.url,
    userAgent: req.headers['user-agent'],
    ip: req.ip,
  }, 'Requisição recebida');
});

app.addHook('onResponse', async (req, reply) => {
  req.log.info({
    statusCode: reply.statusCode,
    responseTime: reply.elapsedTime,
  }, 'Requisição concluída');
});

app.addHook('onError', async (req, reply, error) => {
  req.log.error({
    err: error,
    statusCode: reply.statusCode,
  }, 'Erro na requisição');
});
```

### Exemplo 3: Endpoint de Health Check

```typescript
import type { FastifyInstance } from 'fastify';
import { prisma } from './lib/db.js';

export async function healthRoutes(app: FastifyInstance) {
  // Liveness — o processo está rodando?
  app.get('/health', async () => {
    return {
      status: 'ok',
      uptime: process.uptime(),
      timestamp: new Date().toISOString(),
    };
  });

  // Readiness — o app consegue servir tráfego? (verifica dependências)
  app.get('/ready', async (req, reply) => {
    const checks: Record<string, { status: string; latencyMs?: number }> = {};

    // Verificar banco de dados
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

### Exemplo 4: Métricas Prometheus

```bash
npm install prom-client
```

`src/lib/metrics.ts`:
```typescript
import { Registry, Counter, Histogram, Gauge, collectDefaultMetrics } from 'prom-client';

export const registry = new Registry();

// Coletar métricas padrão do Node.js (memória, CPU, event loop, etc.)
collectDefaultMetrics({ register: registry });

// Métricas customizadas da aplicação
export const httpRequestsTotal = new Counter({
  name: 'http_requests_total',
  help: 'Total de requisições HTTP',
  labelNames: ['method', 'route', 'status_code'],
  registers: [registry],
});

export const httpRequestDuration = new Histogram({
  name: 'http_request_duration_seconds',
  help: 'Duração das requisições HTTP em segundos',
  labelNames: ['method', 'route', 'status_code'],
  buckets: [0.005, 0.01, 0.025, 0.05, 0.1, 0.25, 0.5, 1, 2.5, 5, 10],
  registers: [registry],
});

export const activeConnections = new Gauge({
  name: 'active_connections',
  help: 'Número de conexões ativas',
  registers: [registry],
});

export const dbQueryDuration = new Histogram({
  name: 'db_query_duration_seconds',
  help: 'Duração de queries do banco em segundos',
  labelNames: ['operation', 'table'],
  buckets: [0.001, 0.005, 0.01, 0.05, 0.1, 0.5, 1],
  registers: [registry],
});
```

Plugin de métricas para Fastify:
```typescript
// src/plugins/metrics.ts
import type { FastifyInstance } from 'fastify';
import fp from 'fastify-plugin';
import { registry, httpRequestsTotal, httpRequestDuration } from '../lib/metrics.js';

export const metricsPlugin = fp(async (app: FastifyInstance) => {
  // Registrar métricas em cada requisição
  app.addHook('onResponse', async (req, reply) => {
    const route = req.routeOptions?.url ?? 'unknown';

    // Ignorar o próprio endpoint de métricas
    if (route === '/metrics') return;

    const labels = {
      method: req.method,
      route,
      status_code: String(reply.statusCode),
    };

    httpRequestsTotal.inc(labels);
    httpRequestDuration.observe(labels, reply.elapsedTime / 1000);
  });

  // Expor endpoint de métricas para o Prometheus fazer scrape
  app.get('/metrics', async (req, reply) => {
    reply.header('Content-Type', registry.contentType);
    return registry.metrics();
  });
});
```

### Exemplo 5: Stack Prometheus + Grafana

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
      - targets: ['host.docker.internal:3000']  # ou nome do container
    metrics_path: '/metrics'
    scrape_interval: 10s

  - job_name: 'node-exporter'
    static_configs:
      - targets: ['node-exporter:9100']
```

### Exemplo 6: Configuração de Alertas

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
          summary: "Alta taxa de erros detectada"
          description: "Taxa de erros é {{ humanizePercentage $value }} nos últimos 5 minutos"

      - alert: SlowResponses
        expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m])) > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "P95 de tempo de resposta acima de 1 segundo"
          description: "Tempo de resposta no percentil 95 é {{ humanizeDuration $value }}"

      - alert: ServiceDown
        expr: up{job="myapp"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Aplicação está fora do ar"
          description: "myapp está indisponível há mais de 1 minuto"

  - name: infrastructure
    rules:
      - alert: HighMemoryUsage
        expr: (1 - (node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)) > 0.9
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Alto uso de memória"
          description: "Uso de memória está acima de 90%: {{ humanizePercentage $value }}"

      - alert: DiskSpaceLow
        expr: (1 - (node_filesystem_avail_bytes / node_filesystem_size_bytes)) > 0.85
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Espaço em disco baixo"
          description: "Disco {{ $labels.device }} está {{ humanizePercentage $value }} cheio"
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

## Padrões e Boas Práticas

### IDs de Correlação

Propague um request ID por todos os logs para rastrear uma requisição pelo seu sistema:

```typescript
import { randomUUID } from 'crypto';
import { AsyncLocalStorage } from 'async_hooks';

const requestContext = new AsyncLocalStorage<{ requestId: string }>();

// Middleware: anexar requestId a cada requisição
app.addHook('onRequest', async (req) => {
  const requestId = req.headers['x-request-id'] as string ?? randomUUID();
  req.id = requestId;
  requestContext.run({ requestId }, () => {});
});

// Utilitário: obter requestId em qualquer parte da call stack
export function getRequestId(): string | undefined {
  return requestContext.getStore()?.requestId;
}
```

### Estratégia de Níveis de Log

```typescript
// Use os níveis de forma consistente na sua equipe
logger.debug({ query, params }, 'Query SQL');                    // depuração detalhada
logger.info({ userId, action }, 'Evento de negócio');           // operações normais
logger.warn({ rateLimitRemaining }, 'Rate limit baixo');        // algo errado mas não quebrado
logger.error({ err, context }, 'Operação falhou');              // requer atenção
logger.fatal({ err }, 'Erro irrecuperável');                    // processo prestes a encerrar
```

Defina `LOG_LEVEL=debug` em desenvolvimento, `LOG_LEVEL=info` em produção. Nunca envie logs de debug para produção.

### Sentry para Rastreamento de Erros

```bash
npm install @sentry/node
```

```typescript
import * as Sentry from '@sentry/node';

Sentry.init({
  dsn: process.env.SENTRY_DSN,
  environment: process.env.NODE_ENV,
  tracesSampleRate: 0.1,   // amostrar 10% das requisições para monitoramento de performance
});

// No error handler do Fastify
app.setErrorHandler((error, req, reply) => {
  Sentry.captureException(error, {
    extra: {
      requestId: req.id,
      url: req.url,
      method: req.method,
    },
  });
  logger.error({ err: error, requestId: req.id }, 'Erro não tratado');
  reply.code(500).send({ error: 'Internal server error' });
});
```

### Dashboard do Grafana como Código

Provisione dashboards automaticamente para que sobrevivam a reinicializações de container:

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

## Anti-padrões a Evitar

### Logar Dados Sensíveis

```typescript
// RUIM — senha e token nos logs
logger.info({ user: { email, password, authToken } }, 'Usuário autenticado');

// BOM — redigir antes de logar, ou logar apenas o necessário
logger.info({ userId: user.id, email: user.email }, 'Usuário autenticado');
```

### Usar console.log em Produção

`console.log` é síncrono, não estruturado e impossível de medir. Substitua por Pino no início de cada projeto.

### Alertar sobre Sintomas, Não Sinais

```yaml
# RUIM — muito barulhento, dispara em picos breves
expr: http_errors_total > 0

# BOM — dispara apenas em taxa de erro sustentada
expr: rate(http_requests_total{status_code=~"5.."}[5m]) / rate(http_requests_total[5m]) > 0.05
for: 2m   # deve ser verdadeiro por 2 minutos contínuos
```

### Ignorar Cardinalidade

Cada combinação única de valores de label cria uma nova série temporal. Labels de alta cardinalidade explodem o armazenamento de métricas:

```typescript
// RUIM — userId como label cria milhões de séries temporais
httpRequests.inc({ userId: req.user.id, route: '/api/users' });

// BOM — apenas dimensões de baixa cardinalidade como labels
httpRequests.inc({ route: '/api/users', status: '200' });
```

### Sem Limites de Retenção de Logs

Sem limites, os logs enchem seu disco. Configure rotação de logs do Docker e/ou envie logs para um service externo.

---

## Debug & Troubleshooting

### Encontrar Erros nos Logs

```bash
# Logs do container Docker
docker compose logs api | jq '. | select(.level >= 50)'   # apenas erros (pino nível 50=error)

# Filtrar por tempo
docker compose logs api --since 1h | grep '"level":50'

# Pesquisar por request ID
docker compose logs api | grep 'req_abc123'

# Monitoramento ao vivo de erros
docker compose logs -f api | jq '. | select(.level >= 40)'  # warn + error
```

### Exemplos de Queries Prometheus

```promql
# Taxa de requisições (requisições por segundo)
sum(rate(http_requests_total[5m]))

# Taxa de erros como porcentagem
100 * sum(rate(http_requests_total{status_code=~"5.."}[5m])) / sum(rate(http_requests_total[5m]))

# P95 de latência por rota
histogram_quantile(0.95, sum(rate(http_request_duration_seconds_bucket[5m])) by (le, route))

# Porcentagem de uso de memória
100 * (1 - node_memory_MemAvailable_bytes / node_memory_MemTotal_bytes)

# Uso de CPU
100 - avg(rate(node_cpu_seconds_total{mode="idle"}[5m])) * 100

# Uso de disco
100 * (1 - node_filesystem_avail_bytes{mountpoint="/"} / node_filesystem_size_bytes{mountpoint="/"})
```

### Verificar se o Prometheus Está Fazendo Scrape

```bash
# Verificar targets
curl http://localhost:9090/api/v1/targets | jq '.data.activeTargets[] | {job, health, lastScrape}'

# Verificar se métrica específica existe
curl -s 'http://localhost:9090/api/v1/query?query=http_requests_total' | jq '.data.result'
```

---

## Cenários do Mundo Real

### Cenário 1: Depurando um Incidente em Produção

```bash
# 1. Verificar se o service está no ar
curl https://myapp.com/health

# 2. Verificar taxa de erros no Prometheus
# http://localhost:9090 → query: rate(http_requests_total{status_code=~"5.."}[5m])

# 3. Encontrar as requisições com falha nos logs
docker compose logs api --since 30m | jq '. | select(.level == 50)' | head -20

# 4. Pegar o request ID de uma requisição com falha, então rastreá-la
docker compose logs api | grep '"requestId":"req_xyz"'

# 5. Verificar se o banco é o culpado
docker compose exec db psql -U app -c "SELECT * FROM pg_stat_activity WHERE state = 'active';"
```

### Cenário 2: Configurando Monitoramento de Uptime

Use um monitor externo de uptime (UptimeRobot, Betterstack) para alertar sobre downtime independentemente do seu stack interno:

```
Tipo do monitor: HTTP(S)
URL: https://myapp.com/health
Intervalo: 60 segundos
Contatos de alerta: seu e-mail/telefone
Status esperado: 200
```

Isso detecta casos onde seu VPS inteiro cai (e o Prometheus junto com ele).

### Cenário 3: Relatório Semanal de Performance

```bash
# Extrair métricas-chave da semana
curl -s "http://localhost:9090/api/v1/query_range" \
  --data-urlencode 'query=avg(rate(http_request_duration_seconds_sum[1h]) / rate(http_request_duration_seconds_count[1h]))' \
  --data-urlencode "start=$(date -d '7 days ago' +%s)" \
  --data-urlencode "end=$(date +%s)" \
  --data-urlencode 'step=3600' \
  | jq '.data.result[0].values | map(.[1] | tonumber) | add / length'
```

---

## Leitura Complementar

- [Documentação do Pino](https://getpino.io/)
- [Documentação do Prometheus](https://prometheus.io/docs/introduction/overview/)
- [Documentação do Grafana](https://grafana.com/docs/)
- [OpenTelemetry para Node.js](https://opentelemetry.io/docs/instrumentation/js/)
- [Google SRE Book — Monitoring Distributed Systems](https://sre.google/sre-book/monitoring-distributed-systems/)
- [Tutorial de PromQL](https://promlabs.com/promql-cheat-sheet/)

---

## Resumo

| Ferramenta / Conceito | Propósito |
|----------------------|-----------|
| Pino | Logging JSON estruturado e rápido |
| Níveis de log | debug < info < warn < error < fatal |
| Request ID | Correlacionar todos os logs de uma única requisição |
| Endpoint `/health` | Verificação de liveness para orquestradores |
| Endpoint `/ready` | Verificação de readiness — valida dependências |
| Endpoint `/metrics` | Endpoint de scrape do Prometheus |
| `prom-client` | Instrumentar Node.js com métricas Prometheus |
| Prometheus | Coleta de métricas em série temporal e alertas |
| Grafana | Visualização e dashboards |
| Alertmanager | Rotear alertas para Slack, e-mail, PagerDuty |
| Node Exporter | Expor métricas do SO Linux para o Prometheus |
| Sentry | Rastreamento e agregação de erros |
| Quatro sinais dourados | Latência, Tráfego, Erros, Saturação |

Você não pode melhorar o que não mede. Instrumente desde o início — adicionar observabilidade a um sistema existente é 10 vezes mais difícil do que construí-la desde o dia um.
