# GCP Core Services

## Overview

Google Cloud Platform (GCP) is the third-largest cloud provider and the preferred platform for data-intensive workloads, machine learning, and organizations that use Google Workspace. GCP's strengths include the best-in-class managed Kubernetes (GKE), BigQuery for analytics, and a strong global network backbone.

This chapter covers the GCP services every backend developer needs: compute (Cloud Run, GKE, Cloud Functions, Compute Engine), storage (Cloud Storage, Filestore), databases (Cloud SQL, Firestore, Bigtable, Memorystore), networking (VPC, Cloud Load Balancing, Cloud CDN, Cloud DNS), and operations (IAM, Secret Manager, Cloud Monitoring, Cloud Logging).

---

## Prerequisites

- GCP account (90-day, $300 free trial; always-free tier after)
- Google Cloud CLI (`gcloud`) installed and configured
- Basic cloud concepts (see `cloud-concepts.md`)

```bash
# Install gcloud CLI (Linux/WSL)
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# Authenticate
gcloud auth login
gcloud auth application-default login   # for local SDK usage

# Set project and region defaults
gcloud config set project my-project-id
gcloud config set compute/region us-central1
gcloud config set run/region us-central1

# Verify
gcloud config list
gcloud projects list
```

---

## Core Concepts

### GCP Resource Hierarchy

```
Organization (company.com)
  └── Folders (optional — teams, departments)
        └── Projects (billing boundary, API enablement unit)
              └── Resources (VMs, buckets, databases, etc.)
```

**Projects** are the primary organizing unit. Each project has its own billing, API enablement, IAM policies, and quota. Think of a project per environment (dev, staging, prod) or per product.

```bash
# Create a project
gcloud projects create myapp-prod \
  --name="MyApp Production" \
  --organization=123456789

# Set it as default
gcloud config set project myapp-prod

# Enable required APIs (must be done per project)
gcloud services enable \
  run.googleapis.com \
  sqladmin.googleapis.com \
  redis.googleapis.com \
  secretmanager.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

### Service Categories

```
Compute:        Cloud Run, GKE, Cloud Functions, Compute Engine, Cloud Run Jobs
Storage:        Cloud Storage, Filestore, Persistent Disk
Databases:      Cloud SQL, Firestore, Bigtable, Spanner, AlloyDB
Caching:        Memorystore (Redis, Memcached)
Networking:     VPC, Cloud Load Balancing, Cloud CDN, Cloud DNS, Cloud Armor
Security:       IAM, Secret Manager, Certificate Manager, Cloud KMS
Operations:     Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting
Messaging:      Pub/Sub, Cloud Tasks, Eventarc
Developer:      Artifact Registry, Cloud Build, Cloud Deploy
Data/ML:        BigQuery, Vertex AI, Dataflow, Dataproc
```

---

## Compute Services

### Cloud Run

Fully managed serverless containers. The GCP equivalent of AWS Fargate + App Runner. Scales to zero, billed per request/CPU second, no cluster to manage.

```bash
# Deploy a container to Cloud Run
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \       # public endpoint
  --min-instances 1 \             # keep warm (avoid cold starts)
  --max-instances 10 \
  --concurrency 100 \             # requests per instance before scaling out
  --cpu 1 \
  --memory 512Mi \
  --timeout 30s \
  --set-env-vars NODE_ENV=production \
  --set-secrets DATABASE_URL=database-url:latest \   # from Secret Manager
  --service-account myapp-sa@myapp-prod.iam.gserviceaccount.com

# Get the service URL
gcloud run services describe myapp-api \
  --region us-central1 \
  --format "value(status.url)"

# Update just the image (rolling deploy)
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:v2 \
  --region us-central1

# Traffic splitting (canary)
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-revisions myapp-api-00001-abc=90,myapp-api-00002-def=10
```

**Cloud Run concurrency model:**
Cloud Run's default is 80 concurrent requests per instance. Increase it for I/O-bound Node.js apps (up to 1000). Decrease it for CPU-heavy workloads.

```bash
gcloud run services update myapp-api \
  --region us-central1 \
  --concurrency 200   # Node.js handles many concurrent I/O-bound requests
```

### Cloud Run Jobs

For batch workloads (database migrations, report generation, data imports) — not long-running services.

```bash
# Create a job
gcloud run jobs create db-migrate \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
  --region us-central1 \
  --command "node" \
  --args "dist/migrate.js" \
  --set-secrets DATABASE_URL=database-url:latest \
  --max-retries 3 \
  --task-timeout 600s

# Run it
gcloud run jobs execute db-migrate --region us-central1

# Schedule it (Cloud Scheduler)
gcloud scheduler jobs create http weekly-report \
  --schedule "0 9 * * 1" \
  --uri "https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/myapp-prod/jobs/generate-report:run" \
  --oauth-service-account-email myapp-sa@myapp-prod.iam.gserviceaccount.com \
  --location us-central1
```

### GKE (Google Kubernetes Engine)

Managed Kubernetes. The most mature managed K8s offering. Use when you need the full Kubernetes ecosystem (custom controllers, service meshes, complex multi-service deployments).

```bash
# Create an Autopilot cluster (GKE manages nodes — recommended)
gcloud container clusters create-auto myapp-cluster \
  --region us-central1 \
  --release-channel regular

# Get credentials for kubectl
gcloud container clusters get-credentials myapp-cluster \
  --region us-central1

# Deploy
kubectl apply -f k8s/

# Check status
kubectl get pods -n myapp
kubectl get services -n myapp
```

### Cloud Functions (2nd Gen)

Event-driven serverless functions. 2nd gen is built on Cloud Run internally — longer timeouts, more memory.

```typescript
// src/index.ts
import { http, HttpFunction } from '@google-cloud/functions-framework';

const handleUsers: HttpFunction = async (req, res) => {
  if (req.method === 'GET') {
    const users = await db.user.findMany();
    res.json(users);
    return;
  }

  if (req.method === 'POST') {
    const user = await db.user.create({ data: req.body });
    res.status(201).json(user);
    return;
  }

  res.status(405).json({ error: 'Method not allowed' });
};

http('users', handleUsers);
```

```bash
# Deploy a Cloud Function (2nd gen)
gcloud functions deploy users \
  --gen2 \
  --runtime nodejs22 \
  --region us-central1 \
  --source . \
  --entry-point users \
  --trigger-http \
  --allow-unauthenticated \
  --set-secrets DATABASE_URL=database-url:latest \
  --memory 256MiB \
  --timeout 30s

# Pub/Sub trigger
gcloud functions deploy processMessage \
  --gen2 \
  --runtime nodejs22 \
  --region us-central1 \
  --trigger-topic my-topic \
  --entry-point processMessage
```

---

## Storage Services

### Cloud Storage

Object storage equivalent to S3. Globally replicated, 11 nines of durability.

```bash
# Create a bucket
gcloud storage buckets create gs://myapp-uploads-prod \
  --location us-central1 \
  --uniform-bucket-level-access   # recommended: disable per-object ACLs

# Upload
gcloud storage cp ./logo.png gs://myapp-uploads-prod/images/logo.png

# Sync directory
gcloud storage rsync ./public/ gs://myapp-static-prod/ --delete-unmatched-destination-objects

# Generate signed URL
gcloud storage sign-url gs://myapp-uploads-prod/documents/report.pdf \
  --duration 1h \
  --private-key-file=/path/to/key.json
```

Cloud Storage from Node.js (`@google-cloud/storage`):

```typescript
import { Storage } from '@google-cloud/storage';

// When running on GCP (Cloud Run, GKE), ADC (Application Default Credentials)
// automatically uses the service account. No credentials in code.
const storage = new Storage({ projectId: process.env.GOOGLE_CLOUD_PROJECT });
const bucket = storage.bucket(process.env.GCS_BUCKET!);

// Upload
async function uploadFile(destination: string, data: Buffer, contentType: string): Promise<string> {
  const file = bucket.file(destination);
  await file.save(data, { contentType, resumable: false });
  return `https://storage.googleapis.com/${process.env.GCS_BUCKET}/${destination}`;
}

// Generate signed URL for private files
async function getSignedUrl(filePath: string, expiresInSeconds = 3600): Promise<string> {
  const [url] = await bucket.file(filePath).getSignedUrl({
    action: 'read',
    expires: Date.now() + expiresInSeconds * 1000,
  });
  return url;
}

// Stream upload (for large files)
async function streamUpload(destination: string, readableStream: NodeJS.ReadableStream, contentType: string): Promise<void> {
  const file = bucket.file(destination);
  const writeStream = file.createWriteStream({ contentType });
  await new Promise<void>((resolve, reject) => {
    readableStream.pipe(writeStream)
      .on('finish', resolve)
      .on('error', reject);
  });
}
```

**Storage classes:**
| Class | Use case | Minimum duration |
|-------|----------|-----------------|
| Standard | Frequently accessed | None |
| Nearline | Accessed ~once/month | 30 days |
| Coldline | Accessed ~once/quarter | 90 days |
| Archive | Accessed ~once/year | 365 days |

```bash
# Set lifecycle policy: move to Nearline after 30 days, Archive after 365
gcloud storage buckets update gs://myapp-uploads-prod \
  --lifecycle-file=lifecycle.json
```

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
        "condition": { "age": 30, "matchesStorageClass": ["STANDARD"] }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "ARCHIVE" },
        "condition": { "age": 365, "matchesStorageClass": ["NEARLINE"] }
      }
    ]
  }
}
```

---

## Database Services

### Cloud SQL

Managed relational databases: PostgreSQL, MySQL, SQL Server.

```bash
# Create a PostgreSQL Cloud SQL instance
gcloud sql instances create myapp-db \
  --database-version POSTGRES_16 \
  --tier db-g1-small \           # or db-n1-standard-2 for production
  --region us-central1 \
  --availability-type REGIONAL \ # multi-zone HA
  --storage-size 20GB \
  --storage-type SSD \
  --storage-auto-increase \
  --backup-start-time 03:00 \
  --retained-backups-count 7 \
  --deletion-protection \
  --no-assign-ip \               # private IP only
  --network projects/myapp-prod/global/networks/myapp-vpc

# Create database and user
gcloud sql databases create app --instance myapp-db
gcloud sql users create appuser \
  --instance myapp-db \
  --password "$(openssl rand -base64 32)"

# Connect via Cloud SQL Auth Proxy (local dev)
cloud-sql-proxy myapp-prod:us-central1:myapp-db &
# Then connect to localhost:5432 normally
```

**Cloud SQL Auth Proxy pattern** — the recommended connection method. The proxy handles IAM authentication and TLS automatically:

```bash
# Install Cloud SQL Auth Proxy
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Run (in background for local dev)
./cloud-sql-proxy myapp-prod:us-central1:myapp-db --port 5432 &

# Connect normally
DATABASE_URL=postgres://appuser:password@127.0.0.1:5432/app
```

In production (Cloud Run), use the Unix socket:

```typescript
// DATABASE_URL for Cloud Run (Cloud SQL socket)
// postgres://user:password@/dbname?host=/cloudsql/project:region:instance
const databaseUrl = process.env.DATABASE_URL;
// Cloud Run automatically mounts the Cloud SQL socket when you add
// the Cloud SQL connection in the service config:
// gcloud run deploy ... --add-cloudsql-instances myapp-prod:us-central1:myapp-db
```

### Firestore

Serverless NoSQL document database. No server management, automatic scaling, real-time sync.

```typescript
import { Firestore, FieldValue } from '@google-cloud/firestore';

// ADC automatically picks up service account credentials on GCP
const db = new Firestore({ projectId: process.env.GOOGLE_CLOUD_PROJECT });

// Write a document
async function createUser(user: { name: string; email: string }) {
  const ref = db.collection('users').doc();
  await ref.set({
    ...user,
    createdAt: FieldValue.serverTimestamp(),
  });
  return ref.id;
}

// Read a document
async function getUser(userId: string) {
  const doc = await db.collection('users').doc(userId).get();
  if (!doc.exists) return null;
  return { id: doc.id, ...doc.data() };
}

// Query with filter
async function getActiveUsers() {
  const snapshot = await db.collection('users')
    .where('status', '==', 'active')
    .orderBy('createdAt', 'desc')
    .limit(100)
    .get();

  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}

// Transactions
async function transferCredits(fromId: string, toId: string, amount: number) {
  await db.runTransaction(async (transaction) => {
    const fromRef = db.collection('wallets').doc(fromId);
    const toRef = db.collection('wallets').doc(toId);

    const [from, to] = await transaction.getAll(fromRef, toRef);
    if (!from.exists || !to.exists) throw new Error('Wallet not found');
    if (from.data()!.credits < amount) throw new Error('Insufficient credits');

    transaction.update(fromRef, { credits: FieldValue.increment(-amount) });
    transaction.update(toRef, { credits: FieldValue.increment(amount) });
  });
}
```

**When to use Firestore vs Cloud SQL:**
- Firestore: mobile/real-time apps, flexible schema, multi-region active-active, auto-scaling
- Cloud SQL: relational data, complex queries, existing SQL expertise, ACID transactions — most server-side apps

### Memorystore for Redis

Managed Redis on GCP.

```bash
# Create a Redis instance
gcloud redis instances create myapp-cache \
  --size 1 \
  --region us-central1 \
  --redis-version redis_7_0 \
  --tier standard            # standard = HA with replica

# Get connection details
gcloud redis instances describe myapp-cache \
  --region us-central1 \
  --format "value(host,port)"
```

```typescript
import { createClient } from 'redis';

const redis = createClient({
  socket: {
    host: process.env.REDIS_HOST,
    port: Number(process.env.REDIS_PORT ?? 6379),
  },
});
await redis.connect();
```

---

## Security Services

### IAM (Identity and Access Management)

GCP IAM uses a **principal + role + resource** model. Principals can be user accounts, service accounts, or groups.

```bash
# Create a service account for your application
gcloud iam service-accounts create myapp-sa \
  --display-name "MyApp Service Account"

# Grant roles to the service account
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/cloudsql.client"

gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/storage.objectAdmin"

# Attach service account to Cloud Run service
gcloud run deploy myapp-api \
  --service-account myapp-sa@myapp-prod.iam.gserviceaccount.com \
  ...
```

**Workload Identity (GKE):** allows pods to use a Kubernetes service account mapped to a GCP service account — no JSON key files.

### Secret Manager

```bash
# Create a secret
echo -n "postgres://appuser:password@/app?host=/cloudsql/..." | \
  gcloud secrets create database-url --data-file=-

# Add a new version (rotation)
echo -n "new-connection-string" | \
  gcloud secrets versions add database-url --data-file=-

# Access the latest version
gcloud secrets versions access latest --secret database-url
```

From Node.js:

```typescript
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';

const client = new SecretManagerServiceClient();

async function getSecret(secretId: string): Promise<string> {
  const [version] = await client.accessSecretVersion({
    name: `projects/${process.env.GOOGLE_CLOUD_PROJECT}/secrets/${secretId}/versions/latest`,
  });
  return version.payload!.data!.toString();
}

const DATABASE_URL = await getSecret('database-url');
```

### Cloud Monitoring & Cloud Logging

```bash
# Create an alerting policy (error rate)
gcloud alpha monitoring policies create \
  --policy-from-file=alert-policy.json

# Query logs with gcloud
gcloud logging read \
  'resource.type="cloud_run_revision" AND severity>=ERROR' \
  --project myapp-prod \
  --limit 50 \
  --format json
```

---

## Anti-Patterns to Avoid

### Using Service Account JSON Keys

```bash
# BAD — downloading a JSON key creates a long-lived credential that can be leaked
gcloud iam service-accounts keys create key.json \
  --iam-account myapp-sa@myapp-prod.iam.gserviceaccount.com

# GOOD — use Workload Identity (GKE), Metadata Server (Compute Engine),
# or attach a service account to Cloud Run. ADC handles everything.
```

### Leaving Default Compute Service Account with Editor Role

The default Compute service account has broad project-wide permissions. Create dedicated service accounts with minimal roles for each service.

### Public Cloud SQL Instances

```bash
# Always use --no-assign-ip (private IP only)
# Connect via Cloud SQL Auth Proxy or direct private IP within VPC
gcloud sql instances patch myapp-db \
  --no-assign-ip   # remove public IP if it was set
```

### Not Enabling Object Versioning on Critical Buckets

```bash
gcloud storage buckets update gs://myapp-critical-data \
  --versioning
```

---

## Debugging & Troubleshooting

### Cloud Run Service Won't Start

```bash
# Check deployment status
gcloud run services describe myapp-api --region us-central1

# List revisions and their status
gcloud run revisions list --service myapp-api --region us-central1

# Stream logs
gcloud run services logs tail myapp-api --region us-central1

# Specific revision logs
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="myapp-api"' \
  --project myapp-prod --limit 100
```

### Secret Manager Permission Denied

```bash
# Check if service account has secretAccessor role
gcloud projects get-iam-policy myapp-prod \
  --flatten="bindings[].members" \
  --filter="bindings.members:myapp-sa@" \
  --format="table(bindings.role)"
```

### Cloud SQL Connection Timeout

```bash
# Verify Cloud SQL Auth Proxy is configured for Cloud Run
gcloud run services describe myapp-api \
  --region us-central1 \
  --format "value(spec.template.metadata.annotations)"
# Look for: run.googleapis.com/cloudsql-instances

# Add it if missing
gcloud run services update myapp-api \
  --region us-central1 \
  --add-cloudsql-instances myapp-prod:us-central1:myapp-db
```

---

## Real-World Scenarios

### Scenario 1: Deploy a Node.js API to Cloud Run

```bash
# 1. Build and push image to Artifact Registry
gcloud artifacts repositories create myapp \
  --repository-format docker \
  --location us-central1

gcloud builds submit \
  --tag us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest .

# 2. Store database URL in Secret Manager
echo -n "postgres://..." | gcloud secrets create database-url --data-file=-

# 3. Create service account
gcloud iam service-accounts create myapp-sa
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"

# 4. Deploy
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \
  --set-secrets DATABASE_URL=database-url:latest \
  --service-account myapp-sa@myapp-prod.iam.gserviceaccount.com \
  --add-cloudsql-instances myapp-prod:us-central1:myapp-db
```

### Scenario 2: Pub/Sub Fan-Out

```typescript
// Publisher
import { PubSub } from '@google-cloud/pubsub';
const pubsub = new PubSub({ projectId: process.env.GOOGLE_CLOUD_PROJECT });

async function publishEvent(event: { type: string; payload: unknown }) {
  const topic = pubsub.topic('user-events');
  await topic.publishMessage({
    json: event,
    attributes: { eventType: event.type },
  });
}

// Subscriber (Cloud Function or Cloud Run)
app.post('/pubsub-handler', async (req, res) => {
  const message = Buffer.from(req.body.message.data, 'base64').toString();
  const event = JSON.parse(message);
  await handleEvent(event);
  res.status(204).send();
});
```

---

## Further Reading

- [GCP Documentation](https://cloud.google.com/docs)
- [Cloud Run documentation](https://cloud.google.com/run/docs)
- [Google Cloud Node.js client libraries](https://cloud.google.com/nodejs/docs/reference)
- [gcloud CLI reference](https://cloud.google.com/sdk/gcloud/reference)
- [GCP Architecture Framework](https://cloud.google.com/architecture/framework)
- [Cloud SQL best practices](https://cloud.google.com/sql/docs/postgres/best-practices)

---

## Summary

| Service | Use Case | AWS Equivalent | Azure Equivalent |
|---------|----------|----------------|-----------------|
| Cloud Run | Serverless containers | ECS Fargate / App Runner | Container Apps |
| Cloud Run Jobs | Batch workloads | ECS Fargate tasks (one-shot) | Container Apps Jobs |
| GKE Autopilot | Managed Kubernetes | EKS | AKS |
| Cloud Functions | Event-driven serverless | Lambda | Azure Functions |
| Cloud Storage | Object storage | S3 | Blob Storage |
| Cloud SQL | Managed PostgreSQL/MySQL | RDS | Azure Database for PostgreSQL |
| Firestore | Serverless NoSQL | DynamoDB | Cosmos DB |
| Memorystore | Managed Redis | ElastiCache | Azure Cache for Redis |
| Cloud Load Balancing | Global HTTP(S) LB | ALB | Application Gateway |
| Cloud CDN | Content delivery network | CloudFront | Azure Front Door |
| IAM + Service Accounts | Access control | IAM roles | Azure AD + Managed Identity |
| Secret Manager | Secret storage | Secrets Manager | Key Vault |
| Cloud Monitoring | Metrics + alerting | CloudWatch | Azure Monitor |

GCP's killer features for backend developers: Cloud Run's simplicity, Cloud SQL Auth Proxy for secure DB connections without keys, and Application Default Credentials making auth seamless across local dev and production.
