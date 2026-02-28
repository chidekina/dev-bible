# Azure Architecture

## Overview

Azure's architecture patterns mirror AWS's Well-Architected Framework — Microsoft calls it the **Azure Well-Architected Framework** and organizes it into five pillars: Reliability, Security, Cost Optimization, Operational Excellence, and Performance Efficiency.

This chapter covers how to compose Azure services into production architectures: the containerized web application using Container Apps, the serverless event-driven pipeline, multi-region active-passive setup, and CI/CD with GitHub Actions. Each pattern is practical, opinionated, and grounded in real production deployments on Node.js/TypeScript stacks.

---

## Prerequisites

- Azure Core Services (`azure-core-services.md`)
- Cloud Concepts (`cloud-concepts.md`)
- Basic networking knowledge (VNets, subnets, private endpoints)

---

## Core Concepts

### The Azure Well-Architected Framework

| Pillar | Core Question |
|--------|--------------|
| **Reliability** | How do we design to withstand failures? |
| **Security** | How do we protect workloads end-to-end? |
| **Cost Optimization** | How do we deliver value at the lowest cost? |
| **Operational Excellence** | How do we deploy, monitor, and improve reliably? |
| **Performance Efficiency** | How do we scale efficiently to meet demand? |

### Azure Regions and Availability Zones

```
Azure Region (e.g., East US)
  ├── Availability Zone 1 (independent datacenter)
  ├── Availability Zone 2 (independent datacenter)
  └── Availability Zone 3 (independent datacenter)

Zone-redundant services replicate automatically across AZs:
  → Azure Database for PostgreSQL (Zone Redundant High Availability)
  → Application Gateway v2
  → Azure Cache for Redis (Premium tier)
  → Zone-redundant storage (ZRS)
```

**Region pairing:** Azure pairs regions for disaster recovery (East US ↔ West US, Brazil South ↔ South Central US). Geo-redundant backups replicate to the paired region automatically.

### Managed Identity — The Azure Auth Superpower

The single most important architecture principle in Azure: **never store credentials**. Managed Identity allows Azure resources to authenticate to other Azure services without any credentials in code or environment variables.

```
Container App (Managed Identity) → Key Vault (no password)
App Service (Managed Identity) → Blob Storage (no SAS token)
Azure Function (Managed Identity) → Service Bus (no connection string)
```

```typescript
// This pattern works across ALL Azure SDK clients
import { DefaultAzureCredential } from '@azure/identity';

// DefaultAzureCredential credential chain:
// 1. Environment variables (CI/CD pipelines)
// 2. Workload Identity (Kubernetes pods)
// 3. Managed Identity (Azure compute resources)
// 4. Azure CLI (local dev: az login)
// 5. VS Code Azure extension (local dev)
const credential = new DefaultAzureCredential();
```

---

## Architecture Patterns

### Pattern 1: Containerized API with Container Apps

The recommended architecture for most Node.js APIs on Azure. No cluster management, scales to zero, built-in HTTPS.

```
                    ┌───────────────┐
                    │  Azure DNS    │ (Custom domain)
                    └──────┬────────┘
                           │
                    ┌──────▼────────┐
                    │  Front Door   │ (Global CDN + WAF)
                    └──────┬────────┘
                           │
              ┌────────────▼─────────────┐
              │   Container Apps Env     │
              │  ┌─────────────────────┐ │
              │  │   Container App     │ │ ← auto-scales 1-10 replicas
              │  │   (Node.js API)     │ │
              │  └────────┬────────────┘ │
              └───────────┼──────────────┘
                          │ (VNet integration)
              ┌───────────▼──────────────┐
              │   Private Endpoints      │
              │  ┌─────────┐ ┌────────┐ │
              │  │Postgres │ │ Redis  │ │
              │  │Flex Srv │ │ Cache  │ │
              │  └─────────┘ └────────┘ │
              └──────────────────────────┘
```

**Infrastructure as Code (Bicep):**

```bicep
// main.bicep
param location string = resourceGroup().location
param appName string = 'myapp'
param imageTag string = 'latest'

// Container Apps Environment
resource caEnv 'Microsoft.App/managedEnvironments@2023-05-01' = {
  name: '${appName}-env'
  location: location
  properties: {
    vnetConfiguration: {
      infrastructureSubnetId: appSubnet.id
      internal: false
    }
  }
}

// Container App with system-assigned Managed Identity
resource containerApp 'Microsoft.App/containerApps@2023-05-01' = {
  name: '${appName}-api'
  location: location
  identity: {
    type: 'SystemAssigned'
  }
  properties: {
    managedEnvironmentId: caEnv.id
    configuration: {
      ingress: {
        external: true
        targetPort: 3000
        transport: 'http'
      }
      registries: [{
        server: containerRegistry.properties.loginServer
        identity: 'system'
      }]
    }
    template: {
      scale: {
        minReplicas: 1
        maxReplicas: 10
        rules: [{
          name: 'http-scaling'
          http: {
            metadata: {
              concurrentRequests: '100'
            }
          }
        }]
      }
      containers: [{
        name: 'api'
        image: '${containerRegistry.properties.loginServer}/${appName}:${imageTag}'
        resources: {
          cpu: json('0.5')
          memory: '1.0Gi'
        }
        env: [
          { name: 'NODE_ENV', value: 'production' }
          { name: 'KEY_VAULT_NAME', value: keyVault.name }
        ]
      }]
    }
  }
}

// Grant Container App access to Key Vault (RBAC model)
resource kvSecretsUser 'Microsoft.Authorization/roleAssignments@2022-04-01' = {
  name: guid(keyVault.id, containerApp.id, 'Key Vault Secrets User')
  scope: keyVault
  properties: {
    roleDefinitionId: subscriptionResourceId(
      'Microsoft.Authorization/roleDefinitions',
      '4633458b-17de-408a-b874-0445c86b69e6'   // Key Vault Secrets User
    )
    principalId: containerApp.identity.principalId
    principalType: 'ServicePrincipal'
  }
}
```

**Node.js startup — load secrets from Key Vault:**

```typescript
// src/config.ts
import { SecretClient } from '@azure/keyvault-secrets';
import { DefaultAzureCredential } from '@azure/identity';

interface Config {
  databaseUrl: string;
  redisUrl: string;
  jwtSecret: string;
  nodeEnv: string;
}

export async function loadConfig(): Promise<Config> {
  const kvName = process.env.KEY_VAULT_NAME;

  if (!kvName) {
    // Local dev — use .env file
    return {
      databaseUrl: process.env.DATABASE_URL!,
      redisUrl: process.env.REDIS_URL!,
      jwtSecret: process.env.JWT_SECRET!,
      nodeEnv: process.env.NODE_ENV ?? 'development',
    };
  }

  const client = new SecretClient(
    `https://${kvName}.vault.azure.net`,
    new DefaultAzureCredential()
  );

  const [databaseUrl, redisUrl, jwtSecret] = await Promise.all([
    client.getSecret('database-url').then(s => s.value!),
    client.getSecret('redis-url').then(s => s.value!),
    client.getSecret('jwt-secret').then(s => s.value!),
  ]);

  return { databaseUrl, redisUrl, jwtSecret, nodeEnv: process.env.NODE_ENV ?? 'production' };
}

// src/index.ts
import { loadConfig } from './config.js';
import Fastify from 'fastify';

const config = await loadConfig();
const app = Fastify({ logger: true });
// wire up routes...
await app.listen({ port: 3000, host: '0.0.0.0' });
```

### Pattern 2: Serverless Event-Driven Pipeline

For async processing: image resizing, report generation, email delivery.

```
Blob Storage ──→ Event Grid ──→ Service Bus ──→ Azure Functions
(upload)         (event)        (queue)         (process + store result)
                                    │
                                    └── Dead Letter Queue
                                        (alert on failures)
```

```typescript
// Azure Function triggered by Service Bus
import { app, InvocationContext } from '@azure/functions';
import { BlobServiceClient } from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

interface ProcessImageMessage {
  blobName: string;
  userId: string;
  containerName: string;
}

app.serviceBusQueue('processImage', {
  connection: 'SERVICE_BUS_CONNECTION',
  queueName: 'image-processing',
  handler: async (message: ProcessImageMessage, context: InvocationContext) => {
    const { blobName, userId, containerName } = message;
    context.log({ blobName, userId }, 'Processing image');

    const blobClient = new BlobServiceClient(
      `https://${process.env.STORAGE_ACCOUNT}.blob.core.windows.net`,
      new DefaultAzureCredential()
    );

    const inputContainer = blobClient.getContainerClient(containerName);
    const outputContainer = blobClient.getContainerClient('processed');

    // Download original
    const blob = inputContainer.getBlockBlobClient(blobName);
    const response = await blob.download(0);
    const buffer = await streamToBuffer(response.readableStreamBody!);

    // Process (resize, compress, etc.)
    const resized = await resizeImage(buffer, { width: 800, height: 600 });

    // Store result
    const outputBlob = outputContainer.getBlockBlobClient(`${userId}/${blobName}`);
    await outputBlob.upload(resized, resized.length, {
      blobHTTPHeaders: { blobContentType: 'image/jpeg' },
    });

    await updateUserAsset(userId, blobName, outputBlob.url);
    context.log({ blobName, userId }, 'Image processing complete');
  },
});
```

**Service Bus setup:**

```bash
# Create Service Bus namespace
az servicebus namespace create \
  --name myapp-sb \
  --resource-group myapp-prod-rg \
  --sku Standard \
  --location eastus

# Create queue with dead letter queue (enabled by default, configure max delivery count)
az servicebus queue create \
  --name image-processing \
  --namespace-name myapp-sb \
  --resource-group myapp-prod-rg \
  --max-delivery-count 3 \
  --lock-duration PT5M \
  --default-message-time-to-live P1D

# Wire Blob Storage events → Service Bus via Event Grid
az eventgrid event-subscription create \
  --name blob-to-servicebus \
  --source-resource-id /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.Storage/storageAccounts/myappstorage \
  --endpoint-type servicebusqueue \
  --endpoint /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.ServiceBus/namespaces/myapp-sb/queues/image-processing \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/
```

### Pattern 3: Multi-Region Active-Passive

For applications that need regional failover with an RTO under 30 minutes.

```
Primary: East US
  ├── Azure Front Door (routes to nearest healthy origin automatically)
  ├── Container Apps (myapp-api-eastus) — active
  ├── PostgreSQL Flexible Server — primary (writes)
  └── Azure Cache for Redis — primary

Secondary: West US (standby — start on failover)
  ├── Container Apps (myapp-api-westus) — min-replicas 0, starts on failover
  ├── PostgreSQL Flexible Server — geo-replica (read-only until promoted)
  └── Azure Cache for Redis — secondary
```

```bash
# Create geo-replica of PostgreSQL
az postgres flexible-server geo-restore \
  --name myapp-db-westus \
  --resource-group myapp-prod-rg \
  --source-server myapp-db \
  --location westus

# Front Door health probes route away from unhealthy origins automatically
az afd origin create \
  --resource-group myapp-prod-rg \
  --profile-name myapp-fd \
  --origin-group-name api-origins \
  --origin-name eastus \
  --host-name myapp-api-eastus.azurecontainerapps.io \
  --priority 1 \
  --weight 100

az afd origin create \
  --resource-group myapp-prod-rg \
  --profile-name myapp-fd \
  --origin-group-name api-origins \
  --origin-name westus \
  --host-name myapp-api-westus.azurecontainerapps.io \
  --priority 2 \
  --weight 100
```

**Failover runbook:**

```bash
# 1. Promote geo-replica to standalone primary
az postgres flexible-server promote \
  --name myapp-db-westus \
  --resource-group myapp-prod-rg \
  --mode planned   # use 'forced' only during actual outage

# 2. Update DATABASE_URL secret in West US Key Vault
az keyvault secret set \
  --vault-name myapp-kv-westus \
  --name database-url \
  --value "postgres://app:password@myapp-db-westus.postgres.database.azure.com:5432/app"

# 3. Scale up the standby Container App
az containerapp update \
  --name myapp-api-westus \
  --resource-group myapp-prod-rg \
  --min-replicas 2

# Front Door automatically detects primary health probe failure
# and routes 100% traffic to westus origin at priority 2
```

### Pattern 4: GitHub Actions CI/CD to Container Apps

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write    # Required for OIDC — no stored Azure credentials
  contents: read

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Set up Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npm test
      - run: npm run build

      - name: Azure Login (OIDC — no secrets stored in GitHub)
        uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Build and push image to ACR
        run: |
          az acr build \
            --registry ${{ vars.ACR_NAME }} \
            --image myapp:${{ github.sha }} \
            --file Dockerfile .

      - name: Deploy to Container Apps
        run: |
          az containerapp update \
            --name myapp-api \
            --resource-group myapp-prod-rg \
            --image ${{ vars.ACR_NAME }}.azurecr.io/myapp:${{ github.sha }}

      - name: Verify deployment health
        run: |
          sleep 15
          curl -f https://myapp-api.azurecontainerapps.io/health || exit 1
```

**Set up OIDC (one-time setup — no GitHub secrets required):**

```bash
# Create a service principal
SP=$(az ad sp create-for-rbac --name myapp-github-actions --output json)
APP_ID=$(echo $SP | jq -r .appId)

# Create federated credential for the main branch
az ad app federated-credential create \
  --id $APP_ID \
  --parameters "{
    \"name\": \"github-actions-main\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:myorg/myapp:ref:refs/heads/main\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

# Grant Contributor on the resource group
az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope /subscriptions/SUB/resourceGroups/myapp-prod-rg
```

---

## Operational Patterns

### Zero-Downtime Deploy with Revision Traffic Splitting

Container Apps supports gradual traffic shifting between revisions:

```bash
# Deploy new version — send 0% traffic initially
az containerapp update \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --image myregistry.azurecr.io/myapp:v2 \
  --revision-suffix v2

az containerapp ingress traffic set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --traffic-weight myapp-api--v1=100 myapp-api--v2=0

# Canary: 10% of traffic to v2
az containerapp ingress traffic set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --traffic-weight myapp-api--v1=90 myapp-api--v2=10

# Watch error rates in Application Insights, then full cutover
az containerapp ingress traffic set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --traffic-weight myapp-api--v2=100
```

### Monitoring with Azure Monitor

```bash
# Create alert: 5xx errors above threshold
az monitor metrics alert create \
  --name myapp-5xx-alert \
  --resource-group myapp-prod-rg \
  --scopes /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.App/containerApps/myapp-api \
  --condition "count Http5xx > 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action myapp-action-group
```

Log Analytics (Kusto) query for error rate:

```kusto
requests
| where timestamp > ago(1h)
| summarize
    total = count(),
    errors = countif(resultCode >= 500)
    by bin(timestamp, 5m)
| extend errorRate = round(100.0 * errors / total, 2)
| project timestamp, errorRate
| render timechart
```

---

## Common Patterns & Best Practices

- **Separate resource groups per environment** (`myapp-dev-rg`, `myapp-staging-rg`, `myapp-prod-rg`). Enables independent access control, cost tracking, and teardown.
- **Tag everything from day one:** `Project`, `Environment`, `Team`, `CostCenter`.
- **Use RBAC over Key Vault access policies** for Key Vault — RBAC is the modern model and supports conditional access.
- **Enable Diagnostic Settings** on all production resources. Route logs to a central Log Analytics workspace.
- **Use Azure Container Registry Tasks** to build images inside Azure — avoids pushing large images from your laptop.

---

## Anti-Patterns to Avoid

### Skipping VNet Integration

Without VNet integration, your Container App or App Service talks to databases over the public internet (even with firewall rules). Use VNet integration + private endpoints for all production databases.

```bash
az containerapp env create \
  --name myapp-env \
  --resource-group myapp-prod-rg \
  --location eastus \
  --infrastructure-subnet-resource-id /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.Network/virtualNetworks/myapp-vnet/subnets/infra-subnet
```

### One Environment for Everything

Separate resource groups per environment. Staging should be able to be wiped without affecting prod.

### Not Enabling Purge Protection on Key Vault

```bash
az keyvault update \
  --name myapp-kv \
  --resource-group myapp-prod-rg \
  --enable-purge-protection true
```

Without purge protection, a deleted secret can be permanently destroyed within the soft-delete window.

---

## Debugging & Troubleshooting

### Container App Not Starting

```bash
# Check revision and replica status
az containerapp revision list \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --query "[].{name:name,active:properties.active,state:properties.runningState}" \
  --output table

# Stream system logs
az containerapp logs show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --type system

# Check container logs
az containerapp logs show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --follow
```

### Key Vault Access Denied (403)

```bash
# Check current RBAC assignments on Key Vault
az role assignment list \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.KeyVault/vaults/myapp-kv \
  --query "[].{principalId:principalId,role:roleDefinitionName}"

# Assign Key Vault Secrets User role to managed identity
az role assignment create \
  --assignee PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.KeyVault/vaults/myapp-kv
```

### PostgreSQL Connection Refused from Container App

```bash
# Verify private endpoint is provisioned
az network private-endpoint list \
  --resource-group myapp-prod-rg \
  --query "[?contains(name,'postgres')].{name:name,state:provisioningState}"

# Verify private DNS zone is linked to VNet
az network private-dns link vnet list \
  --resource-group myapp-prod-rg \
  --zone-name privatelink.postgres.database.azure.com
```

---

## Real-World Scenarios

### Scenario 1: SaaS MVP on Azure

```
Budget: $80-120/month
Stack:  Node.js API + React SPA + PostgreSQL

Resources:
  Azure Static Web Apps (frontend)  — Free tier
  Container Apps (API, 0.5 CPU)     — ~$18/mo
  PostgreSQL Flexible Server B1ms   — ~$12/mo
  Azure Container Registry Basic    — ~$5/mo
  Key Vault Standard                — ~$1/mo
  Application Insights (5 GB free)  — $0
  Front Door Standard               — ~$35/mo (optional at MVP stage)
  Total:                              ~$36/mo without Front Door
```

### Scenario 2: Zero-Downtime Deploy

See "Zero-Downtime Deploy with Revision Traffic Splitting" above. Use canary deployment to shift 10% → 50% → 100% over 30 minutes, monitoring Application Insights error rate at each step.

---

## Further Reading

- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [Azure Container Apps documentation](https://learn.microsoft.com/azure/container-apps/)
- [Bicep documentation](https://learn.microsoft.com/azure/azure-resource-manager/bicep/)
- [Azure GitHub Actions OIDC setup](https://learn.microsoft.com/azure/developer/github/connect-from-azure)
- [Azure Static Web Apps](https://learn.microsoft.com/azure/static-web-apps/)

---

## Summary

| Pattern | When to Use |
|---------|-------------|
| Container Apps | Most Node.js APIs — serverless containers, scale-to-zero |
| App Service | Single-container apps, teams familiar with PaaS, deployment slots |
| Event-driven (Service Bus) | Async processing: images, emails, reports, webhooks |
| Multi-region active-passive | Critical apps needing regional failover |
| GitHub Actions + OIDC | CI/CD with zero stored credentials |
| Managed Identity everywhere | Replace all credentials with identity-based auth |
| VNet integration + private endpoints | Always for production — databases never public |

Start with Container Apps + PostgreSQL Flexible Server + Key Vault + Managed Identity. This covers 95% of production requirements with minimal operational overhead. Add Front Door when you need a global CDN or WAF. Add Service Bus when you need reliable async processing.
