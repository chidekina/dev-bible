# Azure Core Services

## Overview

Microsoft Azure is the second-largest cloud provider, with particularly strong adoption in enterprise environments, Windows/.NET shops, and organizations already invested in the Microsoft ecosystem. Azure's strength lies in its hybrid cloud story (Azure Arc), first-class Active Directory integration, and deep enterprise compliance coverage.

This chapter covers the Azure services every backend developer needs: compute (App Service, Container Apps, Azure Functions, AKS), storage (Blob Storage, Azure Files), databases (Azure Database for PostgreSQL, Cosmos DB, Azure Cache for Redis), networking (Virtual Network, Application Gateway, Azure Front Door), and operations (Azure AD, Key Vault, Monitor, Log Analytics).

We focus on practical Node.js/TypeScript usage: CLI commands, SDK examples, and configuration patterns that work in real production environments.

---

## Prerequisites

- Azure account (free tier: $200 credit for 30 days, then always-free services)
- Azure CLI installed and configured
- Basic cloud concepts (see `cloud-concepts.md`)

```bash
# Install Azure CLI (Linux/WSL)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Log in
az login

# Set default subscription
az account set --subscription "My Subscription"

# Verify
az account show
```

---

## Core Concepts

### Resource Hierarchy

Azure organizes resources in a nested hierarchy:

```
Management Groups          (governance across subscriptions)
  └── Subscriptions        (billing boundary)
        └── Resource Groups (logical containers for related resources)
              └── Resources  (VMs, databases, storage accounts, etc.)
```

**Resource Groups** are the key organizational unit. Think of them as a folder for a project or environment. You can delete a resource group and all its resources at once — very useful for ephemeral environments.

```bash
# Create a resource group
az group create \
  --name myapp-prod-rg \
  --location eastus \
  --tags Project=myapp Environment=production Team=backend

# List resource groups
az group list --output table
```

### Service Categories

```
Compute:        App Service, Container Apps, Functions, AKS, VMs
Storage:        Blob Storage, Azure Files, Queue Storage, Table Storage
Databases:      Azure Database for PostgreSQL, Cosmos DB, SQL Database
Caching:        Azure Cache for Redis
Networking:     Virtual Network, Application Gateway, Front Door, DNS
Security:       Azure AD, Key Vault, Managed Identity, Defender for Cloud
Operations:     Monitor, Log Analytics, Application Insights, Cost Management
Messaging:      Service Bus, Event Grid, Event Hubs
Developer:      Container Registry, DevOps, GitHub Actions integration
```

---

## Compute Services

### Azure App Service

Managed platform for web applications. Supports Node.js, Python, .NET, Java, PHP, and Docker containers. No infrastructure to manage.

```bash
# Create an App Service Plan (the underlying compute)
az appservice plan create \
  --name myapp-plan \
  --resource-group myapp-prod-rg \
  --sku B2 \           # B1/B2/B3 = Basic, P1v3/P2v3 = Premium
  --is-linux

# Create the web app (Node.js)
az webapp create \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --plan myapp-plan \
  --runtime "NODE:22-lts"

# Set environment variables
az webapp config appsettings set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --settings \
    NODE_ENV=production \
    DATABASE_URL="@Microsoft.KeyVault(SecretUri=https://myapp-kv.vault.azure.net/secrets/database-url)"

# Deploy from a Docker image
az webapp config container set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --docker-custom-image-name myregistry.azurecr.io/myapp:latest \
  --docker-registry-server-url https://myregistry.azurecr.io

# Enable auto-scaling
az monitor autoscale create \
  --resource-group myapp-prod-rg \
  --resource myapp-plan \
  --resource-type Microsoft.Web/serverfarms \
  --name myapp-autoscale \
  --min-count 2 \
  --max-count 10 \
  --count 2

az monitor autoscale rule create \
  --resource-group myapp-prod-rg \
  --autoscale-name myapp-autoscale \
  --condition "CpuPercentage > 70 avg 5m" \
  --scale out 2
```

**App Service tiers:**
| Tier | vCPU | RAM | Use case |
|------|------|-----|----------|
| B1 | 1 | 1.75 GB | Dev/test |
| B2 | 2 | 3.5 GB | Small prod |
| P1v3 | 2 | 8 GB | Production |
| P2v3 | 4 | 16 GB | High traffic |

### Azure Container Apps

Fully managed serverless container platform built on Kubernetes. No cluster to manage — you get scaling (including scale-to-zero), service discovery, and ingress out of the box.

```bash
# Create a Container Apps environment
az containerapp env create \
  --name myapp-env \
  --resource-group myapp-prod-rg \
  --location eastus

# Deploy a container app
az containerapp create \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --environment myapp-env \
  --image myregistry.azurecr.io/myapp:latest \
  --target-port 3000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 10 \
  --cpu 0.5 \
  --memory 1.0Gi \
  --registry-server myregistry.azurecr.io \
  --env-vars \
    NODE_ENV=production \
    "DATABASE_URL=secretref:database-url"

# Add a secret
az containerapp secret set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --secrets "database-url=postgres://..."

# Scale rules (HTTP-based)
az containerapp update \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --scale-rule-name http-scaling \
  --scale-rule-type http \
  --scale-rule-metadata concurrentRequests=100
```

**Container Apps vs App Service:**
- Container Apps: better for microservices, scale-to-zero, event-driven scaling, Dapr integration
- App Service: simpler for single-container apps, better deployment slots, more familiar for web apps

### Azure Functions

Serverless functions. Billed per execution (first 1M executions/month free).

```typescript
// src/functions/httpTrigger.ts
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';

app.http('users', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  route: 'users/{id?}',
  handler: async (request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
    context.log('Processing users request');

    const id = request.params.id;

    if (id) {
      const user = await db.user.findUnique({ where: { id } });
      if (!user) return { status: 404, jsonBody: { error: 'Not found' } };
      return { jsonBody: user };
    }

    const users = await db.user.findMany();
    return { jsonBody: users };
  },
});
```

```bash
# Install Azure Functions Core Tools
npm install -g azure-functions-core-tools@4

# Initialize a new Functions project
func init myapp-functions --typescript

# Start local development
func start

# Deploy to Azure
func azure functionapp publish myapp-functions
```

**Function trigger types:**
- `http` — HTTP/HTTPS requests
- `timer` — CRON schedules
- `serviceBusTrigger` — Azure Service Bus messages
- `blobTrigger` — Blob Storage uploads
- `eventGridTrigger` — Event Grid events
- `cosmosDBTrigger` — Cosmos DB change feed

---

## Storage Services

### Azure Blob Storage

Object storage equivalent to S3. Three tiers of access:

```bash
# Create a storage account
az storage account create \
  --name myappstorageprod \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2

# Create a container (like an S3 bucket "folder")
az storage container create \
  --name uploads \
  --account-name myappstorageprod \
  --auth-mode login

# Upload a file
az storage blob upload \
  --account-name myappstorageprod \
  --container-name uploads \
  --name documents/report.pdf \
  --file ./report.pdf

# Generate SAS URL (time-limited access)
az storage blob generate-sas \
  --account-name myappstorageprod \
  --container-name uploads \
  --name documents/report.pdf \
  --permissions r \
  --expiry "$(date -u -d '1 hour' '+%Y-%m-%dT%H:%MZ')" \
  --auth-mode login \
  --as-user \
  --full-uri
```

Azure Blob Storage from Node.js (`@azure/storage-blob`):

```typescript
import {
  BlobServiceClient,
  StorageSharedKeyCredential,
  generateBlobSASQueryParameters,
  BlobSASPermissions,
} from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

// Production: use Managed Identity (no credentials in code)
const blobClient = new BlobServiceClient(
  `https://${process.env.STORAGE_ACCOUNT}.blob.core.windows.net`,
  new DefaultAzureCredential()
);

const containerClient = blobClient.getContainerClient('uploads');

// Upload a file
async function uploadBlob(blobName: string, data: Buffer, contentType: string): Promise<string> {
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  await blockBlobClient.upload(data, data.length, {
    blobHTTPHeaders: { blobContentType: contentType },
  });
  return blockBlobClient.url;
}

// List blobs
async function listBlobs(prefix: string): Promise<string[]> {
  const blobs: string[] = [];
  for await (const blob of containerClient.listBlobsFlat({ prefix })) {
    blobs.push(blob.name);
  }
  return blobs;
}

// Download a blob
async function downloadBlob(blobName: string): Promise<Buffer> {
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  const response = await blockBlobClient.download(0);
  const chunks: Buffer[] = [];
  for await (const chunk of response.readableStreamBody!) {
    chunks.push(Buffer.from(chunk));
  }
  return Buffer.concat(chunks);
}
```

**Blob access tiers:**
| Tier | Use case | Retrieval cost | Storage cost |
|------|----------|----------------|--------------|
| Hot | Frequently accessed | Low | High |
| Cool | Infrequently accessed (30+ day minimum) | Medium | Medium |
| Cold | Rarely accessed (90+ day minimum) | High | Low |
| Archive | Rarely accessed (180+ day minimum) | Very high | Very low |

---

## Database Services

### Azure Database for PostgreSQL — Flexible Server

Fully managed PostgreSQL. Flexible Server is the recommended tier (replaces Single Server).

```bash
# Create a Flexible Server
az postgres flexible-server create \
  --name myapp-db \
  --resource-group myapp-prod-rg \
  --location eastus \
  --admin-user appuser \
  --admin-password "$(openssl rand -base64 32)" \
  --sku-name Standard_D2s_v3 \
  --tier GeneralPurpose \
  --version 16 \
  --storage-size 32 \
  --high-availability ZoneRedundant \
  --backup-retention 7 \
  --geo-redundant-backup Enabled \
  --public-access None    # private access only

# Create a database
az postgres flexible-server db create \
  --resource-group myapp-prod-rg \
  --server-name myapp-db \
  --database-name app

# Connect via CLI
az postgres flexible-server connect \
  --name myapp-db \
  --admin-user appuser \
  --admin-password mysecretpassword \
  --database-name app
```

**SKU naming:**
- `Burstable_B1ms` — dev/test, variable load
- `GeneralPurpose_Standard_D2s_v3` — most production workloads
- `MemoryOptimized_Standard_E2s_v3` — memory-intensive workloads (large working sets)

### Azure Cosmos DB

Multi-model, globally distributed NoSQL database. Supports multiple APIs: NoSQL (formerly Core SQL), MongoDB, Cassandra, Gremlin, Table.

```bash
# Create a Cosmos DB account (NoSQL API)
az cosmosdb create \
  --name myapp-cosmos \
  --resource-group myapp-prod-rg \
  --kind GlobalDocumentDB \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --enable-automatic-failover true

# Create database and container
az cosmosdb sql database create \
  --account-name myapp-cosmos \
  --resource-group myapp-prod-rg \
  --name app

az cosmosdb sql container create \
  --account-name myapp-cosmos \
  --resource-group myapp-prod-rg \
  --database-name app \
  --name events \
  --partition-key-path /userId \
  --throughput 400
```

Cosmos DB NoSQL API from Node.js:

```typescript
import { CosmosClient } from '@azure/cosmos';
import { DefaultAzureCredential } from '@azure/identity';

const client = new CosmosClient({
  endpoint: process.env.COSMOS_ENDPOINT!,
  aadCredentials: new DefaultAzureCredential(),
});

const container = client.database('app').container('events');

// Create a document
async function createEvent(event: { userId: string; type: string; data: unknown }) {
  const { resource } = await container.items.create({
    ...event,
    id: crypto.randomUUID(),
    createdAt: new Date().toISOString(),
  });
  return resource;
}

// Query documents
async function getUserEvents(userId: string) {
  const { resources } = await container.items
    .query({
      query: 'SELECT * FROM c WHERE c.userId = @userId ORDER BY c.createdAt DESC',
      parameters: [{ name: '@userId', value: userId }],
    })
    .fetchAll();
  return resources;
}
```

**When to use Cosmos DB vs PostgreSQL:**
- Cosmos DB: globally distributed writes, variable schema, extreme scale (millions of ops/sec), multi-region active-active
- PostgreSQL: relational data, complex queries, ACID transactions, existing SQL expertise — covers 95% of use cases

### Azure Cache for Redis

Managed Redis service. Same `redis` npm client as any Redis.

```bash
# Create a Redis cache
az redis create \
  --name myapp-cache \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku Standard \
  --vm-size c1 \
  --enable-non-ssl-port false   # SSL only

# Get connection string
az redis list-keys \
  --name myapp-cache \
  --resource-group myapp-prod-rg
```

```typescript
import { createClient } from 'redis';

const redis = createClient({
  url: `rediss://:${process.env.REDIS_KEY}@myapp-cache.redis.cache.windows.net:6380`,
  // Note: rediss:// (with double s) for TLS
});

await redis.connect();
```

---

## Networking Services

### Virtual Network (VNet)

Azure's private network. Equivalent to AWS VPC.

```bash
# Create a VNet
az network vnet create \
  --name myapp-vnet \
  --resource-group myapp-prod-rg \
  --address-prefix 10.0.0.0/16

# Create subnets
az network vnet subnet create \
  --name app-subnet \
  --resource-group myapp-prod-rg \
  --vnet-name myapp-vnet \
  --address-prefix 10.0.1.0/24

az network vnet subnet create \
  --name db-subnet \
  --resource-group myapp-prod-rg \
  --vnet-name myapp-vnet \
  --address-prefix 10.0.2.0/24
```

### Application Gateway

Layer 7 load balancer with WAF capabilities. Equivalent to AWS ALB + WAF.

```bash
# Create an Application Gateway with WAF
az network application-gateway create \
  --name myapp-agw \
  --resource-group myapp-prod-rg \
  --location eastus \
  --vnet-name myapp-vnet \
  --subnet agw-subnet \
  --sku WAF_v2 \
  --capacity 2 \
  --http-settings-protocol Https \
  --frontend-port 443 \
  --cert-file ./cert.pfx \
  --cert-password "certpassword"
```

### Azure Front Door

Global CDN and load balancer. Routes traffic to the nearest healthy backend.

```bash
# Create a Front Door profile
az afd profile create \
  --profile-name myapp-fd \
  --resource-group myapp-prod-rg \
  --sku Standard_AzureFrontDoor

# Add an endpoint
az afd endpoint create \
  --resource-group myapp-prod-rg \
  --profile-name myapp-fd \
  --endpoint-name myapp-api \
  --enabled-state Enabled
```

---

## Security Services

### Azure Active Directory / Entra ID

Microsoft's identity platform. Used for both internal (employee) and external (customer) identity.

```bash
# Create a service principal for your application
az ad sp create-for-rbac \
  --name myapp-sp \
  --role Contributor \
  --scopes /subscriptions/SUBSCRIPTION_ID/resourceGroups/myapp-prod-rg

# Output:
# {
#   "appId": "...",
#   "displayName": "myapp-sp",
#   "password": "...",
#   "tenant": "..."
# }
```

**Managed Identity (recommended):** Instead of service principals, assign a managed identity to your App Service / Container App / VM. Azure automatically handles credential rotation.

```bash
# Enable system-assigned managed identity on an App Service
az webapp identity assign \
  --name myapp-api \
  --resource-group myapp-prod-rg

# Grant it access to Key Vault
az keyvault set-policy \
  --name myapp-kv \
  --object-id $(az webapp identity show --name myapp-api --resource-group myapp-prod-rg --query principalId -o tsv) \
  --secret-permissions get list
```

### Azure Key Vault

Secure secret, key, and certificate storage.

```bash
# Create Key Vault
az keyvault create \
  --name myapp-kv \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku standard \
  --enable-soft-delete true \
  --retention-days 90

# Store a secret
az keyvault secret set \
  --vault-name myapp-kv \
  --name database-url \
  --value "postgres://app:password@myapp-db.postgres.database.azure.com:5432/app"

# Retrieve a secret
az keyvault secret show \
  --vault-name myapp-kv \
  --name database-url \
  --query value -o tsv
```

From Node.js:

```typescript
import { SecretClient } from '@azure/keyvault-secrets';
import { DefaultAzureCredential } from '@azure/identity';

const credential = new DefaultAzureCredential();
const client = new SecretClient(
  `https://${process.env.KEY_VAULT_NAME}.vault.azure.net`,
  credential
);

async function getSecret(secretName: string): Promise<string> {
  const secret = await client.getSecret(secretName);
  if (!secret.value) throw new Error(`Secret ${secretName} has no value`);
  return secret.value;
}

// At startup
const DATABASE_URL = await getSecret('database-url');
```

### Azure Monitor & Application Insights

Metrics, logs, and distributed tracing in one place.

```bash
# Create an Application Insights resource
az monitor app-insights component create \
  --app myapp-insights \
  --resource-group myapp-prod-rg \
  --location eastus \
  --kind web

# Get instrumentation key
az monitor app-insights component show \
  --app myapp-insights \
  --resource-group myapp-prod-rg \
  --query instrumentationKey -o tsv
```

Application Insights from Node.js:

```typescript
import { useAzureMonitor } from '@azure/monitor-opentelemetry';

// Call before any other imports — auto-instruments HTTP, DB, etc.
useAzureMonitor({
  azureMonitorExporterOptions: {
    connectionString: process.env.APPLICATIONINSIGHTS_CONNECTION_STRING,
  },
});

// Custom metrics
import { metrics } from '@opentelemetry/api';
const meter = metrics.getMeter('myapp');
const requestCounter = meter.createCounter('requests_total');

// In your handler:
requestCounter.add(1, { route: '/users', method: 'GET' });
```

---

## Anti-Patterns to Avoid

### Storing Credentials in App Settings Directly

```bash
# BAD — password visible in portal and CLI output
az webapp config appsettings set \
  --name myapp-api \
  --settings DATABASE_PASSWORD=mysecretpassword

# GOOD — reference Key Vault
az webapp config appsettings set \
  --name myapp-api \
  --settings "DATABASE_URL=@Microsoft.KeyVault(SecretUri=https://myapp-kv.vault.azure.net/secrets/database-url/)"
```

### Not Using Managed Identity

Service principals require credential rotation. Managed Identity is free and auto-rotated:

```typescript
// BAD — credentials in code or environment
const client = new SecretClient(vaultUrl, new ClientSecretCredential(tenantId, clientId, clientSecret));

// GOOD — Managed Identity auto-discovered in Azure, falls back to az login locally
const client = new SecretClient(vaultUrl, new DefaultAzureCredential());
```

### Using a Single Resource Group for Everything

Separate resource groups by environment (dev/staging/prod) and optionally by team. This makes access control, cost tracking, and teardown much cleaner.

### Public Database Endpoints

```bash
# Always set --public-access None and use Private Endpoints or VNet integration
az postgres flexible-server create \
  --public-access None \
  ...
```

---

## Debugging & Troubleshooting

### App Service Logs

```bash
# Enable logging
az webapp log config \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --application-logging filesystem \
  --level information

# Stream live logs
az webapp log tail \
  --name myapp-api \
  --resource-group myapp-prod-rg

# Download logs
az webapp log download \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --log-file logs.zip
```

### Container Apps Logs

```bash
# Stream real-time logs
az containerapp logs show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --follow

# Check replica status
az containerapp replica list \
  --name myapp-api \
  --resource-group myapp-prod-rg
```

### Key Vault Access Issues

```bash
# Check if managed identity has access
az keyvault show --name myapp-kv --query properties.accessPolicies

# Add access policy
az keyvault set-policy \
  --name myapp-kv \
  --object-id MANAGED_IDENTITY_OBJECT_ID \
  --secret-permissions get list
```

---

## Real-World Scenarios

### Scenario 1: Deploy a Node.js API to Container Apps

```bash
# 1. Build and push image to Azure Container Registry
az acr build \
  --registry myregistry \
  --image myapp:latest \
  --file Dockerfile .

# 2. Create environment
az containerapp env create \
  --name myapp-env \
  --resource-group myapp-prod-rg \
  --location eastus

# 3. Deploy
az containerapp create \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --environment myapp-env \
  --image myregistry.azurecr.io/myapp:latest \
  --target-port 3000 \
  --ingress external \
  --min-replicas 1 \
  --max-replicas 5 \
  --registry-server myregistry.azurecr.io \
  --env-vars NODE_ENV=production

# 4. Get the URL
az containerapp show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --query properties.configuration.ingress.fqdn -o tsv
```

### Scenario 2: Rotate a Database Password Without Downtime

```bash
# 1. Store new password in Key Vault
az keyvault secret set \
  --vault-name myapp-kv \
  --name database-url \
  --value "postgres://app:newpassword@myapp-db.postgres.database.azure.com:5432/app"

# 2. Update database password
az postgres flexible-server update \
  --name myapp-db \
  --resource-group myapp-prod-rg \
  --admin-password newpassword

# 3. Restart app (picks up new Key Vault secret on restart)
az webapp restart \
  --name myapp-api \
  --resource-group myapp-prod-rg
```

---

## Further Reading

- [Azure Documentation](https://learn.microsoft.com/azure/)
- [Azure SDK for JavaScript](https://learn.microsoft.com/javascript/api/overview/azure/)
- [Azure CLI Reference](https://learn.microsoft.com/cli/azure/)
- [Azure Free Tier Services](https://azure.microsoft.com/free/free-account-faq/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [DefaultAzureCredential — auth chain explained](https://learn.microsoft.com/azure/developer/javascript/sdk/authentication/overview)

---

## Summary

| Service | Use Case | AWS Equivalent |
|---------|----------|----------------|
| App Service | Managed web apps, APIs | Elastic Beanstalk / App Runner |
| Container Apps | Serverless containers, microservices | ECS Fargate + App Mesh |
| Azure Functions | Event-driven serverless | Lambda |
| Blob Storage | Object storage | S3 |
| Azure Database for PostgreSQL | Managed Postgres | RDS PostgreSQL |
| Cosmos DB | Globally distributed NoSQL | DynamoDB |
| Azure Cache for Redis | Managed Redis | ElastiCache |
| Virtual Network | Isolated private network | VPC |
| Application Gateway | Layer 7 LB + WAF | ALB + WAF |
| Azure Front Door | Global CDN + LB | CloudFront + Route 53 |
| Azure AD / Entra ID | Identity and access | IAM + Cognito |
| Key Vault | Secret/key/cert storage | Secrets Manager + KMS |
| Application Insights | APM + distributed tracing | CloudWatch + X-Ray |

Azure's managed identity system is a standout feature — use it everywhere. Avoid service principals and hardcoded credentials entirely for anything running inside Azure.
