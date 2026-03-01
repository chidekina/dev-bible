# Arquitetura Azure

## Visão Geral

Os padrões de arquitetura do Azure espelham o Well-Architected Framework da AWS — a Microsoft chama de **Azure Well-Architected Framework** e o organiza em cinco pilares: Reliability, Security, Cost Optimization, Operational Excellence e Performance Efficiency.

Este capítulo aborda como compor serviços Azure em arquiteturas de produção: a aplicação web containerizada com Container Apps, o pipeline serverless orientado a eventos, a configuração multi-region active-passive e CI/CD com GitHub Actions. Cada padrão é prático, opinativo e fundamentado em deploys reais de produção com stacks Node.js/TypeScript.

---

## Pré-requisitos

- Serviços fundamentais do Azure (`azure-core-services.md`)
- Conceitos de cloud (`cloud-concepts.md`)
- Conhecimento básico de redes (VNets, subnets, private endpoints)

---

## Conceitos Fundamentais

### O Azure Well-Architected Framework

| Pilar | Pergunta Central |
|-------|------------------|
| **Reliability** | Como projetamos para suportar falhas? |
| **Security** | Como protegemos cargas de trabalho de ponta a ponta? |
| **Cost Optimization** | Como entregamos valor ao menor custo? |
| **Operational Excellence** | Como fazemos deploy, monitoramos e melhoramos de forma confiável? |
| **Performance Efficiency** | Como escalamos com eficiência para atender à demanda? |

### Regions e Availability Zones no Azure

```
Azure Region (ex.: East US)
  ├── Availability Zone 1 (datacenter independente)
  ├── Availability Zone 2 (datacenter independente)
  └── Availability Zone 3 (datacenter independente)

Serviços zone-redundant replicam automaticamente entre AZs:
  → Azure Database for PostgreSQL (Zone Redundant High Availability)
  → Application Gateway v2
  → Azure Cache for Redis (nível Premium)
  → Zone-redundant storage (ZRS)
```

**Region pairing:** O Azure emparelha regions para recuperação de desastres (East US ↔ West US, Brazil South ↔ South Central US). Backups geo-redundantes replicam para a region emparelhada automaticamente.

### Managed Identity — O Superpoder de Auth do Azure

O princípio de arquitetura mais importante no Azure: **nunca armazene credenciais**. Managed Identity permite que recursos do Azure se autentiquem em outros serviços Azure sem nenhuma credencial no código ou em variáveis de ambiente.

```
Container App (Managed Identity) → Key Vault (sem senha)
App Service (Managed Identity) → Blob Storage (sem token SAS)
Azure Function (Managed Identity) → Service Bus (sem connection string)
```

```typescript
// Este padrão funciona em TODOS os clientes do Azure SDK
import { DefaultAzureCredential } from '@azure/identity';

// Cadeia de credenciais DefaultAzureCredential:
// 1. Variáveis de ambiente (pipelines CI/CD)
// 2. Workload Identity (pods Kubernetes)
// 3. Managed Identity (recursos de compute Azure)
// 4. Azure CLI (dev local: az login)
// 5. Extensão Azure do VS Code (dev local)
const credential = new DefaultAzureCredential();
```

---

## Padrões de Arquitetura

### Padrão 1: API Containerizada com Container Apps

A arquitetura recomendada para a maioria das APIs Node.js no Azure. Sem gerenciamento de cluster, escala até zero, HTTPS nativo.

```
                    ┌───────────────┐
                    │  Azure DNS    │ (Domínio customizado)
                    └──────┬────────┘
                           │
                    ┌──────▼────────┐
                    │  Front Door   │ (CDN global + WAF)
                    └──────┬────────┘
                           │
              ┌────────────▼─────────────┐
              │   Container Apps Env     │
              │  ┌─────────────────────┐ │
              │  │   Container App     │ │ ← autoscala 1-10 replicas
              │  │   (API Node.js)     │ │
              │  └────────┬────────────┘ │
              └───────────┼──────────────┘
                          │ (integração VNet)
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

// Container App com Managed Identity de sistema
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

// Conceder acesso do Container App ao Key Vault (modelo RBAC)
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

**Inicialização do Node.js — carregue secrets do Key Vault:**

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
    // Dev local — usa arquivo .env
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
// configurar rotas...
await app.listen({ port: 3000, host: '0.0.0.0' });
```

### Padrão 2: Pipeline Serverless Orientado a Eventos

Para processamento assíncrono: redimensionamento de imagens, geração de relatórios, envio de e-mails.

```
Blob Storage ──→ Event Grid ──→ Service Bus ──→ Azure Functions
(upload)         (evento)       (fila)          (processa + armazena resultado)
                                    │
                                    └── Dead Letter Queue
                                        (alerta em falhas)
```

```typescript
// Azure Function disparada pelo Service Bus
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
    context.log({ blobName, userId }, 'Processando imagem');

    const blobClient = new BlobServiceClient(
      `https://${process.env.STORAGE_ACCOUNT}.blob.core.windows.net`,
      new DefaultAzureCredential()
    );

    const inputContainer = blobClient.getContainerClient(containerName);
    const outputContainer = blobClient.getContainerClient('processed');

    // Baixar original
    const blob = inputContainer.getBlockBlobClient(blobName);
    const response = await blob.download(0);
    const buffer = await streamToBuffer(response.readableStreamBody!);

    // Processar (redimensionar, comprimir, etc.)
    const resized = await resizeImage(buffer, { width: 800, height: 600 });

    // Armazenar resultado
    const outputBlob = outputContainer.getBlockBlobClient(`${userId}/${blobName}`);
    await outputBlob.upload(resized, resized.length, {
      blobHTTPHeaders: { blobContentType: 'image/jpeg' },
    });

    await updateUserAsset(userId, blobName, outputBlob.url);
    context.log({ blobName, userId }, 'Processamento de imagem concluído');
  },
});
```

**Configuração do Service Bus:**

```bash
# Criar namespace do Service Bus
az servicebus namespace create \
  --name myapp-sb \
  --resource-group myapp-prod-rg \
  --sku Standard \
  --location eastus

# Criar fila com dead letter queue (habilitada por padrão, configure max delivery count)
az servicebus queue create \
  --name image-processing \
  --namespace-name myapp-sb \
  --resource-group myapp-prod-rg \
  --max-delivery-count 3 \
  --lock-duration PT5M \
  --default-message-time-to-live P1D

# Conectar eventos do Blob Storage → Service Bus via Event Grid
az eventgrid event-subscription create \
  --name blob-to-servicebus \
  --source-resource-id /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.Storage/storageAccounts/myappstorage \
  --endpoint-type servicebusqueue \
  --endpoint /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.ServiceBus/namespaces/myapp-sb/queues/image-processing \
  --included-event-types Microsoft.Storage.BlobCreated \
  --subject-begins-with /blobServices/default/containers/uploads/
```

### Padrão 3: Multi-Region Active-Passive

Para aplicações que precisam de failover regional com RTO abaixo de 30 minutos.

```
Primário: East US
  ├── Azure Front Door (roteia para a origin saudável mais próxima automaticamente)
  ├── Container Apps (myapp-api-eastus) — ativo
  ├── PostgreSQL Flexible Server — primário (escritas)
  └── Azure Cache for Redis — primário

Secundário: West US (standby — inicia no failover)
  ├── Container Apps (myapp-api-westus) — min-replicas 0, inicia no failover
  ├── PostgreSQL Flexible Server — geo-replica (somente leitura até promoção)
  └── Azure Cache for Redis — secundário
```

```bash
# Criar geo-replica do PostgreSQL
az postgres flexible-server geo-restore \
  --name myapp-db-westus \
  --resource-group myapp-prod-rg \
  --source-server myapp-db \
  --location westus

# Health probes do Front Door roteiam para longe de origins não saudáveis automaticamente
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

**Runbook de failover:**

```bash
# 1. Promover geo-replica a primário standalone
az postgres flexible-server promote \
  --name myapp-db-westus \
  --resource-group myapp-prod-rg \
  --mode planned   # use 'forced' apenas durante indisponibilidade real

# 2. Atualizar secret DATABASE_URL no Key Vault do West US
az keyvault secret set \
  --vault-name myapp-kv-westus \
  --name database-url \
  --value "postgres://app:password@myapp-db-westus.postgres.database.azure.com:5432/app"

# 3. Escalar o Container App standby
az containerapp update \
  --name myapp-api-westus \
  --resource-group myapp-prod-rg \
  --min-replicas 2

# Front Door detecta automaticamente falha no health probe primário
# e roteia 100% do tráfego para a origin westus de prioridade 2
```

### Padrão 4: GitHub Actions CI/CD para Container Apps

```yaml
# .github/workflows/deploy.yml
name: Build and Deploy

on:
  push:
    branches: [main]

permissions:
  id-token: write    # Necessário para OIDC — sem credenciais Azure armazenadas
  contents: read

jobs:
  build-and-deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configurar Node.js
        uses: actions/setup-node@v4
        with:
          node-version: '22'
          cache: 'npm'

      - run: npm ci
      - run: npm test
      - run: npm run build

      - name: Azure Login (OIDC — sem secrets armazenados no GitHub)
        uses: azure/login@v2
        with:
          client-id: ${{ vars.AZURE_CLIENT_ID }}
          tenant-id: ${{ vars.AZURE_TENANT_ID }}
          subscription-id: ${{ vars.AZURE_SUBSCRIPTION_ID }}

      - name: Build e push da imagem para ACR
        run: |
          az acr build \
            --registry ${{ vars.ACR_NAME }} \
            --image myapp:${{ github.sha }} \
            --file Dockerfile .

      - name: Deploy para Container Apps
        run: |
          az containerapp update \
            --name myapp-api \
            --resource-group myapp-prod-rg \
            --image ${{ vars.ACR_NAME }}.azurecr.io/myapp:${{ github.sha }}

      - name: Verificar saúde do deploy
        run: |
          sleep 15
          curl -f https://myapp-api.azurecontainerapps.io/health || exit 1
```

**Configurar OIDC (setup único — sem secrets no GitHub necessário):**

```bash
# Criar um service principal
SP=$(az ad sp create-for-rbac --name myapp-github-actions --output json)
APP_ID=$(echo $SP | jq -r .appId)

# Criar credencial federada para a branch main
az ad app federated-credential create \
  --id $APP_ID \
  --parameters "{
    \"name\": \"github-actions-main\",
    \"issuer\": \"https://token.actions.githubusercontent.com\",
    \"subject\": \"repo:myorg/myapp:ref:refs/heads/main\",
    \"audiences\": [\"api://AzureADTokenExchange\"]
  }"

# Conceder Contributor no resource group
az role assignment create \
  --assignee $APP_ID \
  --role Contributor \
  --scope /subscriptions/SUB/resourceGroups/myapp-prod-rg
```

---

## Padrões Operacionais

### Deploy Sem Downtime com Traffic Splitting por Revisão

Container Apps suporta migração gradual de tráfego entre revisões:

```bash
# Deploy da nova versão — 0% de tráfego inicialmente
az containerapp update \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --image myregistry.azurecr.io/myapp:v2 \
  --revision-suffix v2

az containerapp ingress traffic set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --traffic-weight myapp-api--v1=100 myapp-api--v2=0

# Canary: 10% do tráfego para v2
az containerapp ingress traffic set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --traffic-weight myapp-api--v1=90 myapp-api--v2=10

# Observe taxas de erro no Application Insights, depois faça corte total
az containerapp ingress traffic set \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --traffic-weight myapp-api--v2=100
```

### Monitoramento com Azure Monitor

```bash
# Criar alerta: erros 5xx acima do limite
az monitor metrics alert create \
  --name myapp-5xx-alert \
  --resource-group myapp-prod-rg \
  --scopes /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.App/containerApps/myapp-api \
  --condition "count Http5xx > 10" \
  --window-size 5m \
  --evaluation-frequency 1m \
  --action myapp-action-group
```

Consulta do Log Analytics (Kusto) para taxa de erros:

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

## Padrões e Boas Práticas

- **Separe resource groups por ambiente** (`myapp-dev-rg`, `myapp-staging-rg`, `myapp-prod-rg`). Permite controle de acesso, rastreamento de custos e limpeza independentes.
- **Tague tudo desde o primeiro dia:** `Project`, `Environment`, `Team`, `CostCenter`.
- **Use RBAC em vez de access policies do Key Vault** — RBAC é o modelo moderno e suporta conditional access.
- **Habilite Diagnostic Settings** em todos os recursos de produção. Roteie logs para um workspace central do Log Analytics.
- **Use Azure Container Registry Tasks** para build de imagens dentro do Azure — evita push de imagens grandes a partir do seu computador.

---

## Anti-Padrões a Evitar

### Pular a Integração com VNet

Sem integração com VNet, seu Container App ou App Service conversa com bancos de dados pela internet pública (mesmo com regras de firewall). Use integração com VNet + private endpoints para todos os bancos de dados de produção.

```bash
az containerapp env create \
  --name myapp-env \
  --resource-group myapp-prod-rg \
  --location eastus \
  --infrastructure-subnet-resource-id /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.Network/virtualNetworks/myapp-vnet/subnets/infra-subnet
```

### Um Único Ambiente para Tudo

Separe resource groups por ambiente. Staging deve poder ser apagado sem afetar prod.

### Não Habilitar Purge Protection no Key Vault

```bash
az keyvault update \
  --name myapp-kv \
  --resource-group myapp-prod-rg \
  --enable-purge-protection true
```

Sem purge protection, um secret deletado pode ser destruído permanentemente dentro da janela de soft-delete.

---

## Debugging e Solução de Problemas

### Container App Não Inicia

```bash
# Verificar status de revisões e replicas
az containerapp revision list \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --query "[].{name:name,active:properties.active,state:properties.runningState}" \
  --output table

# Stream de logs do sistema
az containerapp logs show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --type system

# Verificar logs do container
az containerapp logs show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --follow
```

### Acesso ao Key Vault Negado (403)

```bash
# Verificar atribuições RBAC atuais no Key Vault
az role assignment list \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.KeyVault/vaults/myapp-kv \
  --query "[].{principalId:principalId,role:roleDefinitionName}"

# Atribuir role Key Vault Secrets User à Managed Identity
az role assignment create \
  --assignee PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.KeyVault/vaults/myapp-kv
```

### Conexão Recusada do Container App para PostgreSQL

```bash
# Verificar se o private endpoint está provisionado
az network private-endpoint list \
  --resource-group myapp-prod-rg \
  --query "[?contains(name,'postgres')].{name:name,state:provisioningState}"

# Verificar se a private DNS zone está vinculada à VNet
az network private-dns link vnet list \
  --resource-group myapp-prod-rg \
  --zone-name privatelink.postgres.database.azure.com
```

---

## Cenários do Mundo Real

### Cenário 1: MVP SaaS no Azure

```
Orçamento: $80-120/mês
Stack:  API Node.js + React SPA + PostgreSQL

Recursos:
  Azure Static Web Apps (frontend)  — Nível gratuito
  Container Apps (API, 0,5 CPU)     — ~$18/mês
  PostgreSQL Flexible Server B1ms   — ~$12/mês
  Azure Container Registry Basic    — ~$5/mês
  Key Vault Standard                — ~$1/mês
  Application Insights (5 GB grátis) — $0
  Front Door Standard               — ~$35/mês (opcional no MVP)
  Total:                              ~$36/mês sem Front Door
```

### Cenário 2: Deploy Sem Downtime

Veja "Deploy Sem Downtime com Traffic Splitting por Revisão" acima. Use deploy canary para migrar 10% → 50% → 100% ao longo de 30 minutos, monitorando a taxa de erros no Application Insights a cada etapa.

---

## Leitura Adicional

- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [Azure Architecture Center](https://learn.microsoft.com/azure/architecture/)
- [Documentação do Azure Container Apps](https://learn.microsoft.com/azure/container-apps/)
- [Documentação do Bicep](https://learn.microsoft.com/azure/azure-resource-manager/bicep/)
- [Configuração de OIDC com GitHub Actions e Azure](https://learn.microsoft.com/azure/developer/github/connect-from-azure)
- [Azure Static Web Apps](https://learn.microsoft.com/azure/static-web-apps/)

---

## Resumo

| Padrão | Quando Usar |
|--------|-------------|
| Container Apps | Maioria das APIs Node.js — containers serverless, scale-to-zero |
| App Service | Apps com um único container, times familiarizados com PaaS, deployment slots |
| Orientado a eventos (Service Bus) | Processamento assíncrono: imagens, e-mails, relatórios, webhooks |
| Multi-region active-passive | Apps críticas precisando de failover regional |
| GitHub Actions + OIDC | CI/CD sem credenciais armazenadas |
| Managed Identity em todo lugar | Substitua todas as credenciais por autenticação baseada em identidade |
| Integração VNet + private endpoints | Sempre em produção — bancos de dados nunca públicos |

Comece com Container Apps + PostgreSQL Flexible Server + Key Vault + Managed Identity. Isso cobre 95% dos requisitos de produção com mínimo overhead operacional. Adicione Front Door quando precisar de CDN global ou WAF. Adicione Service Bus quando precisar de processamento assíncrono confiável.
