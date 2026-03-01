# Serviços Fundamentais do Azure

## Visão Geral

O Microsoft Azure é o segundo maior provedor de cloud, com adoção especialmente forte em ambientes corporativos, equipes de Windows/.NET e organizações já investidas no ecossistema Microsoft. Os pontos fortes do Azure estão na sua história de hybrid cloud (Azure Arc), integração nativa com Active Directory e cobertura abrangente de compliance empresarial.

Este capítulo aborda os serviços Azure que todo desenvolvedor backend precisa conhecer: compute (App Service, Container Apps, Azure Functions, AKS), storage (Blob Storage, Azure Files), bancos de dados (Azure Database for PostgreSQL, Cosmos DB, Azure Cache for Redis), redes (Virtual Network, Application Gateway, Azure Front Door) e operações (Azure AD, Key Vault, Monitor, Log Analytics).

O foco é no uso prático com Node.js/TypeScript: comandos CLI, exemplos de SDK e padrões de configuração que funcionam em ambientes de produção reais.

---

## Pré-requisitos

- Conta Azure (free tier: $200 de crédito por 30 dias, depois serviços sempre gratuitos)
- Azure CLI instalado e configurado
- Conceitos básicos de cloud (veja `cloud-concepts.md`)

```bash
# Instalar Azure CLI (Linux/WSL)
curl -sL https://aka.ms/InstallAzureCLIDeb | sudo bash

# Fazer login
az login

# Definir assinatura padrão
az account set --subscription "My Subscription"

# Verificar
az account show
```

---

## Conceitos Fundamentais

### Hierarquia de Recursos

O Azure organiza recursos em uma hierarquia aninhada:

```
Management Groups          (governança entre assinaturas)
  └── Subscriptions        (fronteira de billing)
        └── Resource Groups (contêineres lógicos para recursos relacionados)
              └── Resources  (VMs, bancos de dados, contas de storage, etc.)
```

**Resource Groups** são a unidade organizacional chave. Pense neles como uma pasta para um projeto ou ambiente. Você pode deletar um resource group e todos os seus recursos de uma vez — muito útil para ambientes efêmeros.

```bash
# Criar um resource group
az group create \
  --name myapp-prod-rg \
  --location eastus \
  --tags Project=myapp Environment=production Team=backend

# Listar resource groups
az group list --output table
```

### Categorias de Serviços

```
Compute:        App Service, Container Apps, Functions, AKS, VMs
Storage:        Blob Storage, Azure Files, Queue Storage, Table Storage
Databases:      Azure Database for PostgreSQL, Cosmos DB, SQL Database
Caching:        Azure Cache for Redis
Networking:     Virtual Network, Application Gateway, Front Door, DNS
Security:       Azure AD, Key Vault, Managed Identity, Defender for Cloud
Operations:     Monitor, Log Analytics, Application Insights, Cost Management
Messaging:      Service Bus, Event Grid, Event Hubs
Developer:      Container Registry, DevOps, integração com GitHub Actions
```

---

## Serviços de Compute

### Azure App Service

Plataforma gerenciada para aplicações web. Suporta Node.js, Python, .NET, Java, PHP e containers Docker. Sem infraestrutura para gerenciar.

```bash
# Criar um App Service Plan (o compute subjacente)
az appservice plan create \
  --name myapp-plan \
  --resource-group myapp-prod-rg \
  --sku B2 \           # B1/B2/B3 = Basic, P1v3/P2v3 = Premium
  --is-linux

# Criar a web app (Node.js)
az webapp create \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --plan myapp-plan \
  --runtime "NODE:22-lts"

# Definir variáveis de ambiente
az webapp config appsettings set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --settings \
    NODE_ENV=production \
    DATABASE_URL="@Microsoft.KeyVault(SecretUri=https://myapp-kv.vault.azure.net/secrets/database-url)"

# Deploy a partir de uma imagem Docker
az webapp config container set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --docker-custom-image-name myregistry.azurecr.io/myapp:latest \
  --docker-registry-server-url https://myregistry.azurecr.io

# Habilitar auto-scaling
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

**Níveis do App Service:**
| Nível | vCPU | RAM | Caso de uso |
|-------|------|-----|-------------|
| B1 | 1 | 1,75 GB | Dev/test |
| B2 | 2 | 3,5 GB | Prod pequeno |
| P1v3 | 2 | 8 GB | Produção |
| P2v3 | 4 | 16 GB | Alto tráfego |

### Azure Container Apps

Plataforma de containers serverless totalmente gerenciada, construída sobre Kubernetes. Sem cluster para gerenciar — você tem scaling (incluindo scale-to-zero), service discovery e ingress prontos.

```bash
# Criar um ambiente Container Apps
az containerapp env create \
  --name myapp-env \
  --resource-group myapp-prod-rg \
  --location eastus

# Fazer deploy de um container app
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

# Adicionar um secret
az containerapp secret set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --secrets "database-url=postgres://..."

# Regras de scaling (baseado em HTTP)
az containerapp update \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --scale-rule-name http-scaling \
  --scale-rule-type http \
  --scale-rule-metadata concurrentRequests=100
```

**Container Apps vs App Service:**
- Container Apps: melhor para microsserviços, scale-to-zero, scaling orientado a eventos, integração com Dapr
- App Service: mais simples para apps com um único container, melhores deployment slots, mais familiar para desenvolvedores web

### Azure Functions

Funções serverless. Cobrado por execução (primeiro 1M execuções/mês gratuitas).

```typescript
// src/functions/httpTrigger.ts
import { app, HttpRequest, HttpResponseInit, InvocationContext } from '@azure/functions';

app.http('users', {
  methods: ['GET', 'POST'],
  authLevel: 'anonymous',
  route: 'users/{id?}',
  handler: async (request: HttpRequest, context: InvocationContext): Promise<HttpResponseInit> => {
    context.log('Processando requisição de usuários');

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
# Instalar Azure Functions Core Tools
npm install -g azure-functions-core-tools@4

# Inicializar novo projeto Functions
func init myapp-functions --typescript

# Iniciar desenvolvimento local
func start

# Fazer deploy para o Azure
func azure functionapp publish myapp-functions
```

**Tipos de trigger de Function:**
- `http` — Requisições HTTP/HTTPS
- `timer` — Agendamentos CRON
- `serviceBusTrigger` — Mensagens do Azure Service Bus
- `blobTrigger` — Uploads no Blob Storage
- `eventGridTrigger` — Eventos do Event Grid
- `cosmosDBTrigger` — Change feed do Cosmos DB

---

## Serviços de Storage

### Azure Blob Storage

Object storage equivalente ao S3. Três níveis de acesso:

```bash
# Criar uma conta de storage
az storage account create \
  --name myappstorageprod \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku Standard_LRS \
  --kind StorageV2 \
  --access-tier Hot \
  --https-only true \
  --min-tls-version TLS1_2

# Criar um container (como uma "pasta" de bucket S3)
az storage container create \
  --name uploads \
  --account-name myappstorageprod \
  --auth-mode login

# Fazer upload de um arquivo
az storage blob upload \
  --account-name myappstorageprod \
  --container-name uploads \
  --name documents/report.pdf \
  --file ./report.pdf

# Gerar SAS URL (acesso por tempo limitado)
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

Azure Blob Storage a partir do Node.js (`@azure/storage-blob`):

```typescript
import {
  BlobServiceClient,
  StorageSharedKeyCredential,
  generateBlobSASQueryParameters,
  BlobSASPermissions,
} from '@azure/storage-blob';
import { DefaultAzureCredential } from '@azure/identity';

// Produção: use Managed Identity (sem credenciais no código)
const blobClient = new BlobServiceClient(
  `https://${process.env.STORAGE_ACCOUNT}.blob.core.windows.net`,
  new DefaultAzureCredential()
);

const containerClient = blobClient.getContainerClient('uploads');

// Fazer upload de um arquivo
async function uploadBlob(blobName: string, data: Buffer, contentType: string): Promise<string> {
  const blockBlobClient = containerClient.getBlockBlobClient(blobName);
  await blockBlobClient.upload(data, data.length, {
    blobHTTPHeaders: { blobContentType: contentType },
  });
  return blockBlobClient.url;
}

// Listar blobs
async function listBlobs(prefix: string): Promise<string[]> {
  const blobs: string[] = [];
  for await (const blob of containerClient.listBlobsFlat({ prefix })) {
    blobs.push(blob.name);
  }
  return blobs;
}

// Baixar um blob
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

**Níveis de acesso do Blob:**
| Nível | Caso de uso | Custo de recuperação | Custo de storage |
|-------|-------------|----------------------|------------------|
| Hot | Acessado frequentemente | Baixo | Alto |
| Cool | Acesso pouco frequente (mínimo 30 dias) | Médio | Médio |
| Cold | Acessado raramente (mínimo 90 dias) | Alto | Baixo |
| Archive | Acessado raramente (mínimo 180 dias) | Muito alto | Muito baixo |

---

## Serviços de Banco de Dados

### Azure Database for PostgreSQL — Flexible Server

PostgreSQL totalmente gerenciado. Flexible Server é o nível recomendado (substitui o Single Server).

```bash
# Criar um Flexible Server
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
  --public-access None    # apenas acesso privado

# Criar um banco de dados
az postgres flexible-server db create \
  --resource-group myapp-prod-rg \
  --server-name myapp-db \
  --database-name app

# Conectar via CLI
az postgres flexible-server connect \
  --name myapp-db \
  --admin-user appuser \
  --admin-password mysecretpassword \
  --database-name app
```

**Nomenclatura de SKU:**
- `Burstable_B1ms` — dev/test, carga variável
- `GeneralPurpose_Standard_D2s_v3` — maioria das cargas de produção
- `MemoryOptimized_Standard_E2s_v3` — cargas intensivas em memória (grandes working sets)

### Azure Cosmos DB

Banco de dados NoSQL multi-modelo e globalmente distribuído. Suporta múltiplas APIs: NoSQL (antes Core SQL), MongoDB, Cassandra, Gremlin, Table.

```bash
# Criar uma conta Cosmos DB (API NoSQL)
az cosmosdb create \
  --name myapp-cosmos \
  --resource-group myapp-prod-rg \
  --kind GlobalDocumentDB \
  --default-consistency-level Session \
  --locations regionName=eastus failoverPriority=0 isZoneRedundant=true \
  --enable-automatic-failover true

# Criar banco de dados e container
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

API NoSQL do Cosmos DB a partir do Node.js:

```typescript
import { CosmosClient } from '@azure/cosmos';
import { DefaultAzureCredential } from '@azure/identity';

const client = new CosmosClient({
  endpoint: process.env.COSMOS_ENDPOINT!,
  aadCredentials: new DefaultAzureCredential(),
});

const container = client.database('app').container('events');

// Criar um documento
async function createEvent(event: { userId: string; type: string; data: unknown }) {
  const { resource } = await container.items.create({
    ...event,
    id: crypto.randomUUID(),
    createdAt: new Date().toISOString(),
  });
  return resource;
}

// Consultar documentos
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

**Quando usar Cosmos DB vs PostgreSQL:**
- Cosmos DB: escritas globalmente distribuídas, schema variável, escala extrema (milhões de ops/seg), active-active multi-region
- PostgreSQL: dados relacionais, consultas complexas, transações ACID, expertise SQL existente — cobre 95% dos casos de uso

### Azure Cache for Redis

Serviço Redis gerenciado. Mesmo cliente npm `redis` de qualquer Redis.

```bash
# Criar um cache Redis
az redis create \
  --name myapp-cache \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku Standard \
  --vm-size c1 \
  --enable-non-ssl-port false   # somente SSL

# Obter connection string
az redis list-keys \
  --name myapp-cache \
  --resource-group myapp-prod-rg
```

```typescript
import { createClient } from 'redis';

const redis = createClient({
  url: `rediss://:${process.env.REDIS_KEY}@myapp-cache.redis.cache.windows.net:6380`,
  // Nota: rediss:// (com dois s) para TLS
});

await redis.connect();
```

---

## Serviços de Rede

### Virtual Network (VNet)

Rede privada do Azure. Equivalente à AWS VPC.

```bash
# Criar uma VNet
az network vnet create \
  --name myapp-vnet \
  --resource-group myapp-prod-rg \
  --address-prefix 10.0.0.0/16

# Criar subnets
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

Load balancer de camada 7 com capacidades WAF. Equivalente a AWS ALB + WAF.

```bash
# Criar um Application Gateway com WAF
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

CDN global e load balancer. Roteia tráfego para o backend saudável mais próximo.

```bash
# Criar um perfil Front Door
az afd profile create \
  --profile-name myapp-fd \
  --resource-group myapp-prod-rg \
  --sku Standard_AzureFrontDoor

# Adicionar um endpoint
az afd endpoint create \
  --resource-group myapp-prod-rg \
  --profile-name myapp-fd \
  --endpoint-name myapp-api \
  --enabled-state Enabled
```

---

## Serviços de Segurança

### Azure Active Directory / Entra ID

Plataforma de identidade da Microsoft. Usado tanto para identidade interna (funcionários) quanto externa (clientes).

```bash
# Criar um service principal para sua aplicação
az ad sp create-for-rbac \
  --name myapp-sp \
  --role Contributor \
  --scopes /subscriptions/SUBSCRIPTION_ID/resourceGroups/myapp-prod-rg

# Saída:
# {
#   "appId": "...",
#   "displayName": "myapp-sp",
#   "password": "...",
#   "tenant": "..."
# }
```

**Managed Identity (recomendado):** Em vez de service principals, atribua uma Managed Identity ao seu App Service / Container App / VM. O Azure gerencia a rotação de credenciais automaticamente.

```bash
# Habilitar Managed Identity de sistema no App Service
az webapp identity assign \
  --name myapp-api \
  --resource-group myapp-prod-rg

# Conceder acesso ao Key Vault
az keyvault set-policy \
  --name myapp-kv \
  --object-id $(az webapp identity show --name myapp-api --resource-group myapp-prod-rg --query principalId -o tsv) \
  --secret-permissions get list
```

### Azure Key Vault

Armazenamento seguro de secrets, chaves e certificados.

```bash
# Criar Key Vault
az keyvault create \
  --name myapp-kv \
  --resource-group myapp-prod-rg \
  --location eastus \
  --sku standard \
  --enable-soft-delete true \
  --retention-days 90

# Armazenar um secret
az keyvault secret set \
  --vault-name myapp-kv \
  --name database-url \
  --value "postgres://app:password@myapp-db.postgres.database.azure.com:5432/app"

# Recuperar um secret
az keyvault secret show \
  --vault-name myapp-kv \
  --name database-url \
  --query value -o tsv
```

A partir do Node.js:

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
  if (!secret.value) throw new Error(`Secret ${secretName} sem valor`);
  return secret.value;
}

// Na inicialização da aplicação
const DATABASE_URL = await getSecret('database-url');
```

### Azure Monitor e Application Insights

Métricas, logs e rastreamento distribuído em um único lugar.

```bash
# Criar um recurso Application Insights
az monitor app-insights component create \
  --app myapp-insights \
  --resource-group myapp-prod-rg \
  --location eastus \
  --kind web

# Obter instrumentation key
az monitor app-insights component show \
  --app myapp-insights \
  --resource-group myapp-prod-rg \
  --query instrumentationKey -o tsv
```

Application Insights a partir do Node.js:

```typescript
import { useAzureMonitor } from '@azure/monitor-opentelemetry';

// Chame antes de qualquer outro import — auto-instrumenta HTTP, DB, etc.
useAzureMonitor({
  azureMonitorExporterOptions: {
    connectionString: process.env.APPLICATIONINSIGHTS_CONNECTION_STRING,
  },
});

// Métricas customizadas
import { metrics } from '@opentelemetry/api';
const meter = metrics.getMeter('myapp');
const requestCounter = meter.createCounter('requests_total');

// No seu handler:
requestCounter.add(1, { route: '/users', method: 'GET' });
```

---

## Anti-Padrões a Evitar

### Armazenar Credenciais Diretamente nas App Settings

```bash
# RUIM — senha visível no portal e na saída do CLI
az webapp config appsettings set \
  --name myapp-api \
  --settings DATABASE_PASSWORD=mysecretpassword

# BOM — referenciar o Key Vault
az webapp config appsettings set \
  --name myapp-api \
  --settings "DATABASE_URL=@Microsoft.KeyVault(SecretUri=https://myapp-kv.vault.azure.net/secrets/database-url/)"
```

### Não Usar Managed Identity

Service principals precisam de rotação de credenciais. Managed Identity é gratuita e rotacionada automaticamente:

```typescript
// RUIM — credenciais no código ou no ambiente
const client = new SecretClient(vaultUrl, new ClientSecretCredential(tenantId, clientId, clientSecret));

// BOM — Managed Identity auto-descoberta no Azure, cai de volta para az login localmente
const client = new SecretClient(vaultUrl, new DefaultAzureCredential());
```

### Usar um Único Resource Group para Tudo

Separe resource groups por ambiente (dev/staging/prod) e opcionalmente por time. Isso torna o controle de acesso, rastreamento de custos e limpeza muito mais organizados.

### Endpoints Públicos de Banco de Dados

```bash
# Sempre defina --public-access None e use Private Endpoints ou integração com VNet
az postgres flexible-server create \
  --public-access None \
  ...
```

---

## Debugging e Solução de Problemas

### Logs do App Service

```bash
# Habilitar logging
az webapp log config \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --application-logging filesystem \
  --level information

# Stream de logs ao vivo
az webapp log tail \
  --name myapp-api \
  --resource-group myapp-prod-rg

# Baixar logs
az webapp log download \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --log-file logs.zip
```

### Logs do Container Apps

```bash
# Stream de logs em tempo real
az containerapp logs show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --follow

# Verificar status das replicas
az containerapp replica list \
  --name myapp-api \
  --resource-group myapp-prod-rg
```

### Problemas de Acesso ao Key Vault

```bash
# Verificar se a Managed Identity tem acesso
az keyvault show --name myapp-kv --query properties.accessPolicies

# Adicionar access policy
az keyvault set-policy \
  --name myapp-kv \
  --object-id MANAGED_IDENTITY_OBJECT_ID \
  --secret-permissions get list
```

---

## Cenários do Mundo Real

### Cenário 1: Deploy de uma API Node.js no Container Apps

```bash
# 1. Build e push da imagem para Azure Container Registry
az acr build \
  --registry myregistry \
  --image myapp:latest \
  --file Dockerfile .

# 2. Criar ambiente
az containerapp env create \
  --name myapp-env \
  --resource-group myapp-prod-rg \
  --location eastus

# 3. Fazer deploy
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

# 4. Obter a URL
az containerapp show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --query properties.configuration.ingress.fqdn -o tsv
```

### Cenário 2: Rotacionar Senha do Banco de Dados Sem Downtime

```bash
# 1. Armazenar nova senha no Key Vault
az keyvault secret set \
  --vault-name myapp-kv \
  --name database-url \
  --value "postgres://app:newpassword@myapp-db.postgres.database.azure.com:5432/app"

# 2. Atualizar senha do banco de dados
az postgres flexible-server update \
  --name myapp-db \
  --resource-group myapp-prod-rg \
  --admin-password newpassword

# 3. Reiniciar app (pega o novo secret do Key Vault no restart)
az webapp restart \
  --name myapp-api \
  --resource-group myapp-prod-rg
```

---

## Leitura Adicional

- [Documentação do Azure](https://learn.microsoft.com/azure/)
- [Azure SDK para JavaScript](https://learn.microsoft.com/javascript/api/overview/azure/)
- [Referência do Azure CLI](https://learn.microsoft.com/cli/azure/)
- [Serviços gratuitos do Azure](https://azure.microsoft.com/free/free-account-faq/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [DefaultAzureCredential — cadeia de autenticação explicada](https://learn.microsoft.com/azure/developer/javascript/sdk/authentication/overview)

---

## Resumo

| Serviço | Caso de Uso | Equivalente AWS |
|---------|-------------|-----------------|
| App Service | Apps web e APIs gerenciadas | Elastic Beanstalk / App Runner |
| Container Apps | Containers serverless, microsserviços | ECS Fargate + App Mesh |
| Azure Functions | Serverless orientado a eventos | Lambda |
| Blob Storage | Object storage | S3 |
| Azure Database for PostgreSQL | Postgres gerenciado | RDS PostgreSQL |
| Cosmos DB | NoSQL globalmente distribuído | DynamoDB |
| Azure Cache for Redis | Redis gerenciado | ElastiCache |
| Virtual Network | Rede privada isolada | VPC |
| Application Gateway | LB camada 7 + WAF | ALB + WAF |
| Azure Front Door | CDN global + LB | CloudFront + Route 53 |
| Azure AD / Entra ID | Identidade e acesso | IAM + Cognito |
| Key Vault | Storage de secrets/chaves/certs | Secrets Manager + KMS |
| Application Insights | APM + rastreamento distribuído | CloudWatch + X-Ray |

O sistema de Managed Identity do Azure é um recurso de destaque — use-o em todo lugar. Evite completamente service principals e credenciais hardcoded para qualquer coisa rodando dentro do Azure.
