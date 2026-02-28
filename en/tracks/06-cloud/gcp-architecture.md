# GCP Architecture

## Overview

GCP's **Architecture Framework** (equivalent to AWS Well-Architected and Azure WAF) organizes production-ready design around six categories: system design, operational excellence, security, reliability, cost optimization, and performance optimization.

This chapter focuses on the architecture patterns most relevant to Node.js/TypeScript backend teams: Cloud Run–based APIs, event-driven processing with Pub/Sub, multi-region resilience, and CI/CD with Cloud Build. The patterns prioritize GCP's strengths: Application Default Credentials, Cloud SQL Auth Proxy, and the managed networking model.

---

## Prerequisites

- GCP Core Services (`gcp-core-services.md`)
- Cloud Concepts (`cloud-concepts.md`)
- Basic networking knowledge (VPCs, subnets, private service access)

---

## Core Concepts

### Regions, Zones, and Multi-Region

```
Region (us-central1)
  ├── Zone a (us-central1-a) — independent datacenter
  ├── Zone b (us-central1-b)
  └── Zone c (us-central1-c)

Multi-region (us) — automatic geo-replication within the US
Global — some services (Cloud Load Balancing, Pub/Sub) are inherently global
```

**Key design decisions:**
- Cloud SQL: deploy with `--availability-type REGIONAL` for automatic zone failover
- Cloud Run: regional (automatically distributes across zones within the region)
- Cloud Storage: `us-central1` (regional), `us` (multi-region), or `nam4` (dual-region)
- Firestore: multi-region by default

### Application Default Credentials (ADC)

The single most important GCP concept: code never contains credentials. ADC resolves credentials automatically through an ordered chain:

```
Local development:
  1. GOOGLE_APPLICATION_CREDENTIALS env var → service account JSON
  2. gcloud auth application-default login → user credentials

On GCP (Cloud Run, GKE, Compute Engine):
  3. Metadata server → attached service account

This means the same code runs identically locally and in production.
```

```typescript
// No credentials in code — ADC handles everything
import { Storage } from '@google-cloud/storage';
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';
import { Firestore } from '@google-cloud/firestore';

const storage = new Storage();                    // ADC
const secrets = new SecretManagerServiceClient(); // ADC
const db = new Firestore();                       // ADC
```

### VPC and Private Services

GCP's networking model: resources communicate over the VPC. Cloud SQL, Memorystore, and other managed services connect via **Private Service Access** (VPC peering with Google's network).

```bash
# Enable Private Service Access for your VPC
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

## Architecture Patterns

### Pattern 1: Cloud Run API + Cloud SQL

The most common production architecture for Node.js APIs on GCP. Managed containers, managed database, zero infrastructure to maintain.

```
                    ┌──────────────────┐
                    │  Cloud DNS       │ (Custom domain)
                    └──────┬───────────┘
                           │
                    ┌──────▼───────────┐
                    │  Cloud CDN       │ (Static assets, edge caching)
                    └──────┬───────────┘
                           │
                    ┌──────▼───────────┐
                    │  Cloud Load      │
                    │  Balancing       │ (Global HTTPS LB, SSL termination)
                    └──────┬───────────┘
                           │
                    ┌──────▼───────────┐
                    │  Cloud Run       │ (Serverless containers, scales 1-N)
                    │  Service         │
                    └──────┬───────────┘
                           │ Cloud SQL Auth Proxy (Unix socket)
                    ┌──────▼───────────┐
                    │  Cloud SQL       │ (PostgreSQL, Regional HA)
                    │  (Private IP)    │
                    └──────────────────┘

Secrets: Cloud Run → Secret Manager (via service account IAM)
Config:  Environment variables + Secret Manager refs
```

**Full deployment script:**

```bash
#!/bin/bash
set -euo pipefail

PROJECT="myapp-prod"
REGION="us-central1"
SERVICE="myapp-api"
IMAGE="$REGION-docker.pkg.dev/$PROJECT/myapp/api"

# 1. Build and push
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
  --no-traffic   # deploy without sending traffic

# 3. Verify new revision
NEW_REV=$(gcloud run revisions list \
  --service $SERVICE --region $REGION \
  --format "value(name)" --limit 1)

# Health check the new revision
HEALTH=$(gcloud run services describe $SERVICE \
  --region $REGION \
  --format "value(status.url)")

sleep 10
curl -f "$HEALTH/health" || { echo "Health check failed"; exit 1; }

# 4. Shift 100% traffic to new revision
gcloud run services update-traffic $SERVICE \
  --region $REGION \
  --to-latest
```

**Node.js Cloud SQL connection (production via Unix socket):**

```typescript
// src/db.ts
import { PrismaClient } from '@prisma/client';

function getDatabaseUrl(): string {
  const baseUrl = process.env.DATABASE_URL;
  if (!baseUrl) throw new Error('DATABASE_URL is required');

  // Cloud Run provides the socket at /cloudsql/project:region:instance
  // DATABASE_URL should be: postgres://user:pass@/dbname?host=/cloudsql/...
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

// Graceful shutdown
process.on('SIGTERM', async () => {
  await prisma.$disconnect();
  process.exit(0);
});
```

### Pattern 2: Pub/Sub Event-Driven Processing

GCP's Pub/Sub is a global, serverless, highly durable message queue. No provisioning required.

```
User uploads file
       ↓
Cloud Storage
       ↓ (Storage notification)
Pub/Sub Topic: file-uploads
       ↓
Cloud Run (subscriber) ← push subscription (GCP calls your endpoint)
       ↓
  Process file
  Store result in Cloud SQL
  Publish to Pub/Sub: notifications
       ↓
Cloud Run (notification service)
  Send email/push notification
```

```bash
# Create topic and push subscription
gcloud pubsub topics create file-uploads
gcloud pubsub subscriptions create file-uploads-processor \
  --topic file-uploads \
  --push-endpoint "https://myapp-processor-HASH.run.app/pubsub" \
  --push-auth-service-account pubsub-invoker@myapp-prod.iam.gserviceaccount.com \
  --ack-deadline 300 \
  --max-delivery-attempts 5 \
  --dead-letter-topic projects/myapp-prod/topics/file-uploads-dlq \
  --dead-letter-topic-max-delivery-attempts 5

# Connect Cloud Storage to Pub/Sub
gcloud storage buckets notifications create gs://myapp-uploads-prod \
  --topic file-uploads \
  --event-types OBJECT_FINALIZE \
  --payload-format json
```

```typescript
// Cloud Run service receiving Pub/Sub push messages
import Fastify from 'fastify';

const app = Fastify({ logger: true });

interface PubSubMessage {
  message: {
    data: string;        // base64-encoded JSON
    messageId: string;
    attributes: Record<string, string>;
  };
  subscription: string;
}

app.post<{ Body: PubSubMessage }>('/pubsub', async (request, reply) => {
  const { message } = request.body;

  // Decode the message
  const data = JSON.parse(Buffer.from(message.data, 'base64').toString());

  try {
    await processFileUpload(data);
    // Return 2xx to acknowledge the message
    reply.status(204).send();
  } catch (err) {
    request.log.error({ err, messageId: message.messageId }, 'Processing failed');
    // Return 5xx to trigger retry (up to max-delivery-attempts)
    reply.status(500).send({ error: 'Processing failed' });
  }
});

async function processFileUpload(data: { bucket: string; name: string; size: string }) {
  // Download, process, store result...
  app.log.info({ file: data.name }, 'Processing file');
}
```

### Pattern 3: Multi-Region with Global Load Balancing

GCP's Global Load Balancer routes requests to the nearest healthy Cloud Run service across regions.

```
Cloud DNS
    ↓
Cloud Load Balancing (Global — anycast IP)
    ├── Backend: us-central1 Cloud Run (60% weight)
    └── Backend: europe-west1 Cloud Run (40% weight)
         (automatically routes to nearest healthy backend)

Cloud SQL:
  Primary: us-central1
  Cross-region replica: europe-west1 (read-only; promote on failover)
```

```bash
# Deploy to multiple regions
for REGION in us-central1 europe-west1; do
  gcloud run deploy myapp-api \
    --region $REGION \
    --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
    --allow-unauthenticated \
    --min-instances 1
done

# Create Network Endpoint Groups (NEGs) for Cloud Load Balancing
for REGION in us-central1 europe-west1; do
  gcloud compute network-endpoint-groups create myapp-neg-$REGION \
    --region $REGION \
    --network-endpoint-type serverless \
    --cloud-run-service myapp-api
done

# Create backend service
gcloud compute backend-services create myapp-backend \
  --global \
  --load-balancing-scheme EXTERNAL_MANAGED

# Add backends
gcloud compute backend-services add-backend myapp-backend \
  --global \
  --network-endpoint-group myapp-neg-us-central1 \
  --network-endpoint-group-region us-central1

gcloud compute backend-services add-backend myapp-backend \
  --global \
  --network-endpoint-group myapp-neg-europe-west1 \
  --network-endpoint-group-region europe-west1

# Create URL map and proxy
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

### Pattern 4: Cloud Build CI/CD Pipeline

```yaml
# cloudbuild.yaml
steps:
  # Run tests
  - name: node:22
    entrypoint: npm
    args: [ci]

  - name: node:22
    entrypoint: npm
    args: [test]

  - name: node:22
    entrypoint: npm
    args: [run, build]

  # Build and push image
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

  # Deploy to Cloud Run (staging)
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

  # Health check staging
  - name: curlimages/curl
    script: |
      sleep 10
      curl -f https://myapp-api-staging-HASH.run.app/health

  # Deploy to production
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
# Create Cloud Build trigger (on push to main)
gcloud builds triggers create github \
  --repo-name myapp \
  --repo-owner myorg \
  --branch-pattern "^main$" \
  --build-config cloudbuild.yaml \
  --substitutions _DEPLOY_REGION=us-central1
```

---

## Operational Patterns

### Canary Releases with Cloud Run Traffic Splitting

```bash
# Deploy new revision, hold traffic at 0%
gcloud run deploy myapp-api \
  --image myregistry/myapp:v2 \
  --region us-central1 \
  --no-traffic

# Get revision name
NEW_REV=$(gcloud run revisions list \
  --service myapp-api --region us-central1 \
  --format "value(name)" --limit 1)

# Send 5% canary traffic
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-revisions $NEW_REV=5

# Monitor error rate in Cloud Monitoring...

# Promote to 100%
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-latest

# Rollback: send 100% to previous revision
PREV_REV=$(gcloud run revisions list \
  --service myapp-api --region us-central1 \
  --format "value(name)" --limit 1 --filter "name!=$NEW_REV")
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-revisions $PREV_REV=100
```

### Cloud Monitoring Alerts

```bash
# Create uptime check
gcloud monitoring uptime-check-configs create \
  --display-name "myapp-api health" \
  --resource-type url \
  --monitored-resource "type:uptime_url,labels:{host:myapp-api-HASH.run.app,path:/health}"

# Alert policy via JSON config
cat > alert-policy.json << 'EOF'
{
  "displayName": "Cloud Run 5xx errors",
  "conditions": [{
    "displayName": "5xx error rate > 5%",
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

## Common Patterns & Best Practices

- **One project per environment.** `myapp-dev`, `myapp-staging`, `myapp-prod`. This provides clean billing, IAM isolation, and quota separation.
- **Label everything.** GCP labels enable cost attribution: `env=prod`, `team=backend`, `service=api`.
- **Use Artifact Registry, not Container Registry.** Container Registry (`gcr.io`) is deprecated. Use Artifact Registry (`REGION-docker.pkg.dev`).
- **Prefer Cloud Build over external CI for GCP deployments.** It runs inside GCP with native IAM — no service account keys in CI secrets.
- **Cloud Run min-instances=1 for production APIs.** Cold starts are 1-3 seconds. Keeping one warm instance costs ~$5/month and eliminates the latency spike.

---

## Anti-Patterns to Avoid

### Service Account JSON Keys in CI/CD

```bash
# BAD — downloadable key that can be leaked
gcloud iam service-accounts keys create key.json \
  --iam-account deployer@myapp-prod.iam.gserviceaccount.com
# Then storing key.json in GitHub Secrets

# GOOD — Workload Identity Federation: no keys
# GitHub Actions can impersonate a GCP service account via OIDC
# https://cloud.google.com/blog/products/identity-security/enabling-keyless-authentication-from-github-actions
```

### Broad IAM Roles on the Project

```bash
# BAD — Editor or Owner on the whole project
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@..." \
  --role "roles/editor"

# GOOD — minimal roles on specific resources
gcloud run services add-iam-policy-binding myapp-api \
  --member "serviceAccount:myapp-sa@..." \
  --role "roles/run.invoker"
  --region us-central1
```

### Public Cloud SQL IP with Firewall Rules

Cloud SQL with a public IP + firewall rules is weaker than private IP + Cloud SQL Auth Proxy. Use private IP.

### Blocking Pub/Sub Retries

```typescript
// BAD — swallowing errors causes message loss
app.post('/pubsub', async (req, res) => {
  try {
    await processMessage(req.body);
  } catch (err) {
    console.error(err);
  }
  res.status(204).send();  // always 200 → message is acked even on failure
});

// GOOD — return 5xx on failure to trigger retry
app.post('/pubsub', async (req, res) => {
  try {
    await processMessage(req.body);
    res.status(204).send();
  } catch (err) {
    console.error(err);
    res.status(500).send();   // Pub/Sub will retry up to max-delivery-attempts
  }
});
```

---

## Debugging & Troubleshooting

### Cloud Run Service Returns 502

```bash
# 1. Check if the container started successfully
gcloud run revisions describe REVISION_NAME \
  --region us-central1 \
  --format "value(status.conditions)"

# 2. Check container logs for startup errors
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="myapp-api" AND severity>=ERROR' \
  --project myapp-prod --limit 50 --format json

# 3. Common causes:
# - Container listens on wrong port (must match --port or default 8080)
# - Container crashes at startup (missing env var, failed DB connection)
# - Health check endpoint returns non-2xx
```

### Cloud SQL Connection Pool Exhausted

```typescript
// Cloud Run: each instance connects to Cloud SQL via the proxy
// Default Prisma pool size is too large for Cloud Run (which may have many instances)

// Set pool size based on Cloud SQL max_connections / expected max instances
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '?connection_limit=5&pool_timeout=20',
    },
  },
});
```

### Pub/Sub Messages Not Being Delivered

```bash
# Check subscription backlog
gcloud pubsub subscriptions describe file-uploads-processor

# Look for dead-letter queue messages
gcloud pubsub subscriptions pull file-uploads-dlq --limit 5

# Check push subscription auth
gcloud pubsub subscriptions describe file-uploads-processor \
  --format "value(pushConfig.oidcToken.serviceAccountEmail)"
# Verify that service account has roles/run.invoker on the Cloud Run service
```

---

## Real-World Scenarios

### Scenario 1: SaaS MVP on GCP

```
Budget: $50-100/month
Stack:  Node.js API + React SPA + PostgreSQL

Resources:
  Cloud Run (API, min=1, 1 CPU, 512MiB)    — ~$15/mo (mostly compute)
  Cloud SQL db-g1-small PostgreSQL          — ~$25/mo
  Cloud Storage (static hosting)            — ~$1/mo
  Cloud Load Balancing                      — ~$18/mo
  Cloud Build (120 min/day free)            — $0
  Secret Manager                            — ~$0.30/mo
  Cloud Monitoring (free tier)              — $0
  Total:                                      ~$59/mo
```

### Scenario 2: Database Migration as a Cloud Run Job

```typescript
// migrate.ts — runs as Cloud Run Job, not a long-running service
import { PrismaClient } from '@prisma/client';

async function main() {
  console.log('Starting migration...');
  const prisma = new PrismaClient();

  try {
    await prisma.$executeRaw`SELECT 1`; // connectivity check
    console.log('Database connected');

    // Run Prisma migrations
    // In practice: run `prisma migrate deploy` as the container CMD
    console.log('Migrations complete');
  } finally {
    await prisma.$disconnect();
  }
}

main().catch((err) => {
  console.error('Migration failed:', err);
  process.exit(1);
});
```

```bash
# Run migration before new deployment
gcloud run jobs execute db-migrate \
  --region us-central1 \
  --wait  # waits for completion before returning

# Check result
gcloud run jobs executions describe $(gcloud run jobs executions list \
  --job db-migrate --region us-central1 \
  --format "value(name)" --limit 1) \
  --region us-central1
```

---

## Further Reading

- [GCP Architecture Framework](https://cloud.google.com/architecture/framework)
- [Cloud Run patterns](https://cloud.google.com/run/docs/tutorials)
- [Workload Identity Federation](https://cloud.google.com/iam/docs/workload-identity-federation)
- [Cloud SQL Auth Proxy](https://cloud.google.com/sql/docs/postgres/sql-proxy)
- [GCP reference architectures](https://cloud.google.com/architecture)
- [Cloud Build CI/CD](https://cloud.google.com/build/docs/deploying-builds/deploy-cloud-run)

---

## Summary

| Pattern | When to Use |
|---------|-------------|
| Cloud Run + Cloud SQL | Most Node.js APIs — simple, managed, scalable |
| Pub/Sub event-driven | Async processing, decoupled services |
| Global Load Balancing | Multi-region active-active deployments |
| Cloud Run Jobs | Database migrations, batch processing, cron tasks |
| Cloud Build | CI/CD with native GCP IAM — no external secrets |
| Canary releases | Gradual traffic shifts with automatic rollback |
| Workload Identity | CI/CD without service account JSON keys |

GCP's architecture simplicity for Node.js comes from three features working together: Cloud Run handles compute, Cloud SQL Auth Proxy handles database auth, and ADC handles all SDK authentication. The result is a codebase with zero credential management — no JSON files, no rotation scripts, no IAM key sprawl.
