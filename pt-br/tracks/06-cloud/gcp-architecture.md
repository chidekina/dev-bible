# Arquitetura GCP

## Visão Geral

O **Architecture Framework** do GCP (equivalente ao Well-Architected da AWS e ao WAF do Azure) organiza o design pronto para produção em torno de seis categorias: system design, operational excellence, security, reliability, cost optimization e performance optimization.

Este capítulo foca nos padrões de arquitetura mais relevantes para times backend Node.js/TypeScript: APIs baseadas em Cloud Run, processamento orientado a eventos com Pub/Sub, resiliência multi-region e CI/CD com Cloud Build. Os padrões priorizam os pontos fortes do GCP: Application Default Credentials, Cloud SQL Auth Proxy e o modelo de rede gerenciada.

---

## Pré-requisitos

- Serviços fundamentais do GCP (`gcp-core-services.md`)
- Conceitos de cloud (`cloud-concepts.md`)
- Conhecimento básico de redes (VPCs, subnets, private service access)

---

## Conceitos Fundamentais

### Regions, Zones e Multi-Region

```
Region (us-central1)
  ├── Zone a (us-central1-a) — datacenter independente
  ├── Zone b (us-central1-b)
  └── Zone c (us-central1-c)

Multi-region (us) — geo-replicação automática dentro dos EUA
Global — alguns serviços (Cloud Load Balancing, Pub/Sub) são inerentemente globais
```

**Decisões de design principais:**
- Cloud SQL: faça deploy com `--availability-type REGIONAL` para failover automático entre zones
- Cloud Run: regional (distribui automaticamente entre zones dentro da region)
- Cloud Storage: `us-central1` (regional), `us` (multi-region) ou `nam4` (dual-region)
- Firestore: multi-region por padrão

### Application Default Credentials (ADC)

O conceito mais importante do GCP: o código nunca contém credenciais. O ADC resolve credenciais automaticamente através de uma cadeia ordenada:

```
Dev local:
  1. Variável de ambiente GOOGLE_APPLICATION_CREDENTIALS → JSON de service account
  2. gcloud auth application-default login → credenciais do usuário

No GCP (Cloud Run, GKE, Compute Engine):
  3. Metadata server → service account anexada

Isso significa que o mesmo código roda de forma idêntica localmente e em produção.
```

```typescript
// Sem credenciais no código — o ADC cuida de tudo
import { Storage } from '@google-cloud/storage';
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';
import { Firestore } from '@google-cloud/firestore';

const storage = new Storage();                    // ADC
const secrets = new SecretManagerServiceClient(); // ADC
const db = new Firestore();                       // ADC
```

### VPC e Serviços Privados

O modelo de rede do GCP: recursos se comunicam pela VPC. Cloud SQL, Memorystore e outros serviços gerenciados se conectam via **Private Service Access** (VPC peering com a rede do Google).

```bash
# Habilitar Private Service Access para sua VPC
gcloud compute addresses create google-managed-services-range \
  --global \
  --purpose VPC_PEERING \
  --prefix-length 16 \
  --network myapp-vpc

gcloud services vpc-peerings connect \
  --service servicenetworking.googleapis.com \
  --ranges google-managed-services-range \
  --network myapp-vpc \
  --project myapp-prod
```

---

## Padrões de Arquitetura

### Padrão 1: Cloud Run API + Cloud SQL

A arquitetura de produção mais comum para APIs Node.js no GCP. Containers gerenciados, banco de dados gerenciado, zero infraestrutura para manter.

```
                    ┌──────────────────┐
                    │  Cloud DNS       │ (Domínio customizado)
                    └──────┬───────────┘
                           │
                    ┌──────▼───────────┐
                    │  Cloud CDN       │ (Assets estáticos, cache no edge)
                    └──────┬───────────┘
                           │
                    ┌──────▼───────────┐
                    │  Cloud Load      │
                    │  Balancing       │ (LB HTTPS global, SSL termination)
                    └──────┬───────────┘
                           │
                    ┌──────▼───────────┐
                    │  Cloud Run       │ (Containers serverless, escala 1-N)
                    │  Service         │
                    └──────┬───────────┘
                           │ Cloud SQL Auth Proxy (socket Unix)
                    ┌──────▼───────────┐
                    │  Cloud SQL       │ (PostgreSQL, HA Regional)
                    │  (IP Privado)    │
                    └──────────────────┘

Secrets: Cloud Run → Secret Manager (via IAM da service account)
Config:  Variáveis de ambiente + refs do Secret Manager
```

**Script de deploy completo:**

```bash
#!/bin/bash
set -euo pipefail

PROJECT="myapp-prod"
REGION="us-central1"
SERVICE="myapp-api"
IMAGE="$REGION-docker.pkg.dev/$PROJECT/myapp/api"

# 1. Build e push
gcloud builds submit \
  --project $PROJECT \
  --tag "$IMAGE:$GITHUB_SHA" \
  --timeout 600s .

# 2. Deploy
gcloud run deploy $SERVICE \
  --project $PROJECT \
  --region $REGION \
  --image "$IMAGE:$GITHUB_SHA" \
  --platform managed \
  --allow-unauthenticated \
  --min-instances 1 \
  --max-instances 20 \
  --concurrency 200 \
  --cpu 1 \
  --memory 512Mi \
  --timeout 30s \
  --service-account "myapp-sa@$PROJECT.iam.gserviceaccount.com" \
  --add-cloudsql-instances "$PROJECT:$REGION:myapp-db" \
  --set-secrets "DATABASE_URL=database-url:latest,JWT_SECRET=jwt-secret:latest" \
  --set-env-vars "NODE_ENV=production,GOOGLE_CLOUD_PROJECT=$PROJECT" \
  --no-traffic   # deploy sem enviar tráfego

# 3. Verificar nova revisão
NEW_REV=$(gcloud run revisions list \
  --service $SERVICE --region $REGION \
  --format "value(name)" --limit 1)

# Health check na nova revisão
HEALTH=$(gcloud run services describe $SERVICE \
  --region $REGION \
  --format "value(status.url)")

sleep 10
curl -f "$HEALTH/health" || { echo "Health check falhou"; exit 1; }

# 4. Migrar 100% do tráfego para nova revisão
gcloud run services update-traffic $SERVICE \
  --region $REGION \
  --to-latest
```

**Conexão Node.js ao Cloud SQL (produção via socket Unix):**

```typescript
// src/db.ts
import { PrismaClient } from '@prisma/client';

function getDatabaseUrl(): string {
  const baseUrl = process.env.DATABASE_URL;
  if (!baseUrl) throw new Error('DATABASE_URL é obrigatório');

  // Cloud Run fornece o socket em /cloudsql/project:region:instance
  // DATABASE_URL deve ser: postgres://user:pass@/dbname?host=/cloudsql/...
  return baseUrl;
}

export const prisma = new PrismaClient({
  datasources: {
    db: { url: getDatabaseUrl() },
  },
  log: process.env.NODE_ENV === 'development'
    ? ['query', 'warn', 'error']
    : ['warn', 'error'],
});

// Desligamento gracioso
process.on('SIGTERM', async () => {
  await prisma.$disconnect();
  process.exit(0);
});
```

### Padrão 2: Processamento Orientado a Eventos com Pub/Sub

O Pub/Sub do GCP é uma fila de mensagens global, serverless e altamente durável. Sem provisionamento necessário.

```
Usuário faz upload de arquivo
       ↓
Cloud Storage
       ↓ (notificação de Storage)
Pub/Sub Topic: file-uploads
       ↓
Cloud Run (subscriber) ← push subscription (GCP chama seu endpoint)
       ↓
  Processa arquivo
  Armazena resultado no Cloud SQL
  Publica no Pub/Sub: notifications
       ↓
Cloud Run (serviço de notificação)
  Envia e-mail/notificação push
```

```bash
# Criar topic e push subscription
gcloud pubsub topics create file-uploads
gcloud pubsub subscriptions create file-uploads-processor \
  --topic file-uploads \
  --push-endpoint "https://myapp-processor-HASH.run.app/pubsub" \
  --push-auth-service-account pubsub-invoker@myapp-prod.iam.gserviceaccount.com \
  --ack-deadline 300 \
  --max-delivery-attempts 5 \
  --dead-letter-topic projects/myapp-prod/topics/file-uploads-dlq \
  --dead-letter-topic-max-delivery-attempts 5

# Conectar Cloud Storage ao Pub/Sub
gcloud storage buckets notifications create gs://myapp-uploads-prod \
  --topic file-uploads \
  --event-types OBJECT_FINALIZE \
  --payload-format json
```

```typescript
// Serviço Cloud Run recebendo mensagens push do Pub/Sub
import Fastify from 'fastify';

const app = Fastify({ logger: true });

interface PubSubMessage {
  message: {
    data: string;        // JSON codificado em base64
    messageId: string;
    attributes: Record<string, string>;
  };
  subscription: string;
}

app.post<{ Body: PubSubMessage }>('/pubsub', async (request, reply) => {
  const { message } = request.body;

  // Decodificar a mensagem
  const data = JSON.parse(Buffer.from(message.data, 'base64').toString());

  try {
    await processFileUpload(data);
    // Retornar 2xx para confirmar o recebimento da mensagem
    reply.status(204).send();
  } catch (err) {
    request.log.error({ err, messageId: message.messageId }, 'Processamento falhou');
    // Retornar 5xx para acionar retry (até max-delivery-attempts)
    reply.status(500).send({ error: 'Processamento falhou' });
  }
});

async function processFileUpload(data: { bucket: string; name: string; size: string }) {
  // Baixar, processar, armazenar resultado...
  app.log.info({ file: data.name }, 'Processando arquivo');
}
```

### Padrão 3: Multi-Region com Global Load Balancing

O Global Load Balancer do GCP roteia requisições para o serviço Cloud Run saudável mais próximo entre regions.

```
Cloud DNS
    ↓
Cloud Load Balancing (Global — IP anycast)
    ├── Backend: Cloud Run us-central1 (peso 60%)
    └── Backend: Cloud Run europe-west1 (peso 40%)
         (roteia automaticamente para o backend saudável mais próximo)

Cloud SQL:
  Primário: us-central1
  Réplica cross-region: europe-west1 (somente leitura; promova no failover)
```

```bash
# Fazer deploy em múltiplas regions
for REGION in us-central1 europe-west1; do
  gcloud run deploy myapp-api \
    --region $REGION \
    --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
    --allow-unauthenticated \
    --min-instances 1
done

# Criar Network Endpoint Groups (NEGs) para o Cloud Load Balancing
for REGION in us-central1 europe-west1; do
  gcloud compute network-endpoint-groups create myapp-neg-$REGION \
    --region $REGION \
    --network-endpoint-type serverless \
    --cloud-run-service myapp-api
done

# Criar backend service
gcloud compute backend-services create myapp-backend \
  --global \
  --load-balancing-scheme EXTERNAL_MANAGED

# Adicionar backends
gcloud compute backend-services add-backend myapp-backend \
  --global \
  --network-endpoint-group myapp-neg-us-central1 \
  --network-endpoint-group-region us-central1

gcloud compute backend-services add-backend myapp-backend \
  --global \
  --network-endpoint-group myapp-neg-europe-west1 \
  --network-endpoint-group-region europe-west1

# Criar URL map e proxy
gcloud compute url-maps create myapp-urlmap \
  --default-service myapp-backend

gcloud compute target-https-proxies create myapp-https-proxy \
  --url-map myapp-urlmap \
  --ssl-certificates myapp-cert

gcloud compute forwarding-rules create myapp-https \
  --global \
  --target-https-proxy myapp-https-proxy \
  --ports 443
```

### Padrão 4: Pipeline CI/CD com Cloud Build

```yaml
# cloudbuild.yaml
steps:
  # Executar testes
  - name: node:22
    entrypoint: npm
    args: [ci]

  - name: node:22
    entrypoint: npm
    args: [test]

  - name: node:22
    entrypoint: npm
    args: [run, build]

  # Build e push da imagem
  - name: gcr.io/cloud-builders/docker
    args:
      - build
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/myapp/api:$COMMIT_SHA
      - -t
      - us-central1-docker.pkg.dev/$PROJECT_ID/myapp/api:latest
      - .

  - name: gcr.io/cloud-builders/docker
    args: [push, --all-tags, us-central1-docker.pkg.dev/$PROJECT_ID/myapp/api]

  # Deploy para Cloud Run (staging)
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - run
      - deploy
      - myapp-api-staging
      - --image
      - us-central1-docker.pkg.dev/$PROJECT_ID/myapp/api:$COMMIT_SHA
      - --region
      - us-central1
      - --platform
      - managed

  # Health check no staging
  - name: curlimages/curl
    script: |
      sleep 10
      curl -f https://myapp-api-staging-HASH.run.app/health

  # Deploy para produção
  - name: gcr.io/google.com/cloudsdktool/cloud-sdk
    entrypoint: gcloud
    args:
      - run
      - deploy
      - myapp-api
      - --image
      - us-central1-docker.pkg.dev/$PROJECT_ID/myapp/api:$COMMIT_SHA
      - --region
      - us-central1

options:
  logging: CLOUD_LOGGING_ONLY

timeout: 1200s
```

```bash
# Criar trigger do Cloud Build (no push para main)
gcloud builds triggers create github \
  --repo-name myapp \
  --repo-owner myorg \
  --branch-pattern "^main$" \
  --build-config cloudbuild.yaml \
  --substitutions _DEPLOY_REGION=us-central1
```

---

## Padrões Operacionais

### Releases Canary com Traffic Splitting do Cloud Run

```bash
# Deploy da nova revisão, tráfego retido em 0%
gcloud run deploy myapp-api \
  --image myregistry/myapp:v2 \
  --region us-central1 \
  --no-traffic

# Obter nome da revisão
NEW_REV=$(gcloud run revisions list \
  --service myapp-api --region us-central1 \
  --format "value(name)" --limit 1)

# Enviar 5% de tráfego canary
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-revisions $NEW_REV=5

# Monitorar taxa de erros no Cloud Monitoring...

# Promover para 100%
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-latest

# Rollback: enviar 100% para revisão anterior
PREV_REV=$(gcloud run revisions list \
  --service myapp-api --region us-central1 \
  --format "value(name)" --limit 1 --filter "name!=$NEW_REV")
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-revisions $PREV_REV=100
```

### Alertas com Cloud Monitoring

```bash
# Criar verificação de uptime
gcloud monitoring uptime-check-configs create \
  --display-name "myapp-api health" \
  --resource-type url \
  --monitored-resource "type:uptime_url,labels:{host:myapp-api-HASH.run.app,path:/health}"

# Política de alerta via config JSON
cat > alert-policy.json << 'EOF'
{
  "displayName": "Erros 5xx no Cloud Run",
  "conditions": [{
    "displayName": "Taxa de erros 5xx > 5%",
    "conditionThreshold": {
      "filter": "resource.type=\"cloud_run_revision\" AND metric.type=\"run.googleapis.com/request_count\"",
      "aggregations": [{
        "alignmentPeriod": "60s",
        "crossSeriesReducer": "REDUCE_SUM",
        "groupByFields": ["resource.labels.service_name"],
        "perSeriesAligner": "ALIGN_RATE"
      }],
      "comparison": "COMPARISON_GT",
      "thresholdValue": 0.05,
      "duration": "60s"
    }
  }],
  "alertStrategy": { "autoClose": "604800s" },
  "notificationChannels": ["projects/myapp-prod/notificationChannels/CHANNEL_ID"]
}
EOF

gcloud alpha monitoring policies create --policy-from-file=alert-policy.json
```

---

## Padrões e Boas Práticas

- **Um projeto por ambiente.** `myapp-dev`, `myapp-staging`, `myapp-prod`. Isso fornece billing, isolamento IAM e separação de quota limpos.
- **Labele tudo.** Labels do GCP habilitam atribuição de custos: `env=prod`, `team=backend`, `service=api`.
- **Use Artifact Registry, não Container Registry.** Container Registry (`gcr.io`) está depreciado. Use Artifact Registry (`REGION-docker.pkg.dev`).
- **Prefira Cloud Build a CI externo para deploys GCP.** Ele roda dentro do GCP com IAM nativo — sem chaves de service account em secrets de CI.
- **Cloud Run min-instances=1 para APIs de produção.** Cold starts demoram 1-3 segundos. Manter uma instance aquecida custa ~$5/mês e elimina o pico de latência.

---

## Anti-Padrões a Evitar

### Chaves JSON de Service Account em CI/CD

```bash
# RUIM — chave para download que pode ser vazada
gcloud iam service-accounts keys create key.json \
  --iam-account deployer@myapp-prod.iam.gserviceaccount.com
# Depois armazenar key.json no GitHub Secrets

# BOM — Workload Identity Federation: sem chaves
# GitHub Actions pode impersonar uma GCP service account via OIDC
# https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions
```

### Roles IAM Amplas no Projeto

```bash
# RUIM — Editor ou Owner em todo o projeto
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@..." \
  --role "roles/editor"

# BOM — roles mínimas em recursos específicos
gcloud run services add-iam-policy-binding myapp-api \
  --member "serviceAccount:myapp-sa@..." \
  --role "roles/run.invoker"
  --region us-central1
```

### Cloud SQL com IP Público + Regras de Firewall

Cloud SQL com IP público + regras de firewall é mais fraco que IP privado + Cloud SQL Auth Proxy. Use IP privado.

### Engolir Erros e Bloquear Retries do Pub/Sub

```typescript
// RUIM — engolir erros causa perda de mensagens
app.post('/pubsub', async (req, res) => {
  try {
    await processMessage(req.body);
  } catch (err) {
    console.error(err);
  }
  res.status(204).send();  // sempre 200 → mensagem é confirmada mesmo em falha
});

// BOM — retornar 5xx em falha para acionar retry
app.post('/pubsub', async (req, res) => {
  try {
    await processMessage(req.body);
    res.status(204).send();
  } catch (err) {
    console.error(err);
    res.status(500).send();   // Pub/Sub fará retry até max-delivery-attempts
  }
});
```

---

## Debugging e Solução de Problemas

### Serviço Cloud Run Retorna 502

```bash
# 1. Verificar se o container iniciou corretamente
gcloud run revisions describe REVISION_NAME \
  --region us-central1 \
  --format "value(status.conditions)"

# 2. Verificar logs do container por erros de inicialização
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="myapp-api" AND severity>=ERROR' \
  --project myapp-prod --limit 50 --format json

# 3. Causas comuns:
# - Container ouve na porta errada (deve corresponder a --port ou padrão 8080)
# - Container crasha na inicialização (variável de env faltando, conexão DB falhando)
# - Endpoint de health check retorna não-2xx
```

### Pool de Conexões Cloud SQL Esgotado

```typescript
// Cloud Run: cada instance conecta ao Cloud SQL via proxy
// O tamanho padrão do pool do Prisma é muito grande para Cloud Run
// (que pode ter muitas instances)

// Defina o tamanho do pool baseado em max_connections do Cloud SQL / máximo de instances esperadas
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '?connection_limit=5&pool_timeout=20',
    },
  },
});
```

### Mensagens Pub/Sub Não Sendo Entregues

```bash
# Verificar backlog da subscription
gcloud pubsub subscriptions describe file-uploads-processor

# Verificar mensagens na dead-letter queue
gcloud pubsub subscriptions pull file-uploads-dlq --limit 5

# Verificar autenticação da push subscription
gcloud pubsub subscriptions describe file-uploads-processor \
  --format "value(pushConfig.oidcToken.serviceAccountEmail)"
# Verificar que a service account tem roles/run.invoker no serviço Cloud Run
```

---

## Cenários do Mundo Real

### Cenário 1: MVP SaaS no GCP

```
Orçamento: $50-100/mês
Stack:  API Node.js + React SPA + PostgreSQL

Recursos:
  Cloud Run (API, min=1, 1 CPU, 512MiB)    — ~$15/mês (principalmente compute)
  Cloud SQL db-g1-small PostgreSQL          — ~$25/mês
  Cloud Storage (hosting estático)          — ~$1/mês
  Cloud Load Balancing                      — ~$18/mês
  Cloud Build (120 min/dia gratuitos)       — $0
  Secret Manager                            — ~$0,30/mês
  Cloud Monitoring (nível gratuito)         — $0
  Total:                                      ~$59/mês
```

### Cenário 2: Migração de Banco de Dados como Cloud Run Job

```typescript
// migrate.ts — roda como Cloud Run Job, não como serviço de longa duração
import { PrismaClient } from '@prisma/client';

async function main() {
  console.log('Iniciando migração...');
  const prisma = new PrismaClient();

  try {
    await prisma.$executeRaw`SELECT 1`; // verificação de conectividade
    console.log('Banco de dados conectado');

    // Executar migrações do Prisma
    // Na prática: executar `prisma migrate deploy` como CMD do container
    console.log('Migrações concluídas');
  } finally {
    await prisma.$disconnect();
  }
}

main().catch((err) => {
  console.error('Migração falhou:', err);
  process.exit(1);
});
```

```bash
# Executar migração antes do novo deploy
gcloud run jobs execute db-migrate \
  --region us-central1 \
  --wait  # aguarda conclusão antes de retornar

# Verificar resultado
gcloud run jobs executions describe $(gcloud run jobs executions list \
  --job db-migrate --region us-central1 \
  --format "value(name)" --limit 1) \
  --region us-central1
```

---

## Leitura Adicional

- [GCP Architecture Framework](https://cloud.google.com/architecture/framework)
- [Padrões do Cloud Run](https://cloud.google.com/run/docs/tutorials)
- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy)
- [Arquiteturas de referência do GCP](https://cloud.google.com/architecture)
- [CI/CD com Cloud Build](https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run)

---

## Resumo

| Padrão | Quando Usar |
|--------|-------------|
| Cloud Run + Cloud SQL | Maioria das APIs Node.js — simples, gerenciado, escalável |
| Pub/Sub orientado a eventos | Processamento assíncrono, serviços desacoplados |
| Global Load Balancing | Deploys active-active multi-region |
| Cloud Run Jobs | Migrações de banco de dados, processamento batch, tarefas cron |
| Cloud Build | CI/CD com IAM nativo do GCP — sem secrets externos |
| Releases canary | Migração gradual de tráfego com rollback automático |
| Workload Identity | CI/CD sem chaves JSON de service account |

A simplicidade de arquitetura do GCP para Node.js vem de três recursos funcionando juntos: Cloud Run gerencia o compute, Cloud SQL Auth Proxy gerencia a autenticação do banco sem chaves, e o ADC gerencia toda a autenticação do SDK. O resultado é uma codebase com zero gerenciamento de credenciais — sem arquivos JSON, sem scripts de rotação, sem proliferação de chaves IAM.
