# Comparação entre Clouds

## Visão Geral

AWS, Azure e GCP resolvem os mesmos problemas de formas diferentes. Escolher o provider errado para o seu caso de uso não condena um projeto, mas cria fricção — tooling desconhecido, surpresas de preço e serviços que não se encaixam no padrão que você precisa.

Este capítulo dá uma base prática para escolher um provider e para trabalhar em diferentes providers ao longo de diferentes empregos ou projetos. As comparações são ancoradas no desenvolvimento backend real com Node.js/TypeScript: fazer deploy de APIs, armazenar dados, gerenciar secrets e operar em produção.

---

## Pré-requisitos

- AWS Core Services (`aws-core-services.md`)
- Azure Core Services (`azure-core-services.md`)
- GCP Core Services (`gcp-core-services.md`)

---

## Conceitos Fundamentais

### Posição de Mercado (em 2025)

```
AWS:   ~31% de participação — catálogo de serviços mais amplo, mais vagas de emprego
Azure: ~25% de participação — dominante em empresas Microsoft
GCP:   ~12% de participação — mais forte em dados/ML, Kubernetes, startups
```

Nenhum deles é objetivamente "melhor". Eles são significativamente diferentes em filosofia, preço e ecossistema. O seu contexto é que determina a escolha.

### Diferenças de Filosofia

**AWS:** Configuração explícita. Você controla tudo, mas deve configurar tudo. Segurança é opt-in (security groups abertos não geram alerta, simplesmente permitem o tráfego). O maior catálogo de serviços — existe um serviço AWS para quase qualquer caso de uso.

**Azure:** Integração em primeiro lugar. Integração profunda com o ecossistema Microsoft (Active Directory, Office 365, .NET, Windows Server). Managed Identity é o superpoder de autenticação — zero credenciais em qualquer lugar. Bicep e ARM templates para IaC.

**GCP:** Experiência do desenvolvedor e dados. Application Default Credentials eliminam o gerenciamento de credenciais. A infraestrutura de rede global é genuinamente excepcional. Kubernetes de melhor qualidade (GKE), BigQuery para analytics e Vertex AI para ML.

---

## Mapeamento de Serviços

### Compute

| Caso de Uso | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| Containers gerenciados (serverless) | ECS Fargate / App Runner | Container Apps | Cloud Run |
| Platform-as-a-Service | Elastic Beanstalk | App Service | App Engine (legado) |
| Funções serverless | Lambda | Azure Functions | Cloud Functions (2ª geração) |
| Kubernetes gerenciado | EKS | AKS | GKE Autopilot |
| Máquinas virtuais | EC2 | Azure VMs | Compute Engine |
| Batch / containers one-shot | ECS Fargate tasks | Container Apps Jobs | Cloud Run Jobs |

**Recomendação para a maioria das APIs Node.js:**
- AWS: ECS Fargate com um ALB
- Azure: Container Apps
- GCP: Cloud Run

Todas as três abordagens executam containers Docker com auto-scaling. As principais diferenças são o modelo operacional e o preço.

### Storage

| Caso de Uso | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| Object storage | S3 | Blob Storage | Cloud Storage |
| Block storage (discos de VM) | EBS | Managed Disks | Persistent Disk |
| File storage (NFS) | EFS | Azure Files | Filestore |
| CDN | CloudFront | Azure Front Door / CDN | Cloud CDN |

**Comparação de preços de object storage (aproximado, us-east-1/us):**
| Provider | Armazenamento (por GB/mês) | Egresso (por GB) | Operações PUT (por 1M) |
|----------|---------------------------|------------------|------------------------|
| AWS S3 Standard | $0,023 | $0,09 | $5,00 |
| Azure Blob (Hot) | $0,018 | $0,087 | $5,00 |
| GCP Cloud Storage | $0,020 | $0,12 | $5,00 |

O egresso da AWS é notavelmente mais barato para grandes transferências de dados. O Azure tem taxas de armazenamento levemente menores. O GCP tem egresso mais caro, mas é competitivo para a maioria das cargas de trabalho.

### Bancos de Dados

| Caso de Uso | AWS | Azure | GCP |
|-------------|-----|-------|-----|
| PostgreSQL gerenciado | RDS PostgreSQL / Aurora | Azure Database for PostgreSQL Flexible | Cloud SQL PostgreSQL |
| MySQL gerenciado | RDS MySQL / Aurora | Azure Database for MySQL Flexible | Cloud SQL MySQL |
| NoSQL document | DynamoDB | Cosmos DB | Firestore |
| Cache em memória | ElastiCache (Redis/Memcached) | Azure Cache for Redis | Memorystore (Redis/Memcached) |
| Analytics / data warehouse | Redshift | Azure Synapse | BigQuery |
| SQL distribuído global | Aurora Global | Cosmos DB (multi-master) | Spanner |

**Comparação de PostgreSQL gerenciado:**
- **AWS RDS / Aurora:** O mais maduro. Aurora é mais rápido (3-5x em relação ao MySQL padrão, significativo em relação ao Postgres) e caro. Aurora Serverless v2 faz autoscaling de compute.
- **Azure Database for PostgreSQL Flexible:** Zone Redundant HA é sólido. Bom custo-benefício no tier General Purpose.
- **GCP Cloud SQL:** Conecta via Auth Proxy (melhor experiência para o desenvolvedor em autenticação). HA regional incluído. Ligeiramente menos performático que os outros no mesmo preço, mas a integração com ADC é impecável.

### Segurança e Identidade

| Conceito | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Gestão de identidade e acesso | IAM | Azure AD / Entra ID | IAM |
| Identidade de serviço (sem credenciais) | IAM Roles | Managed Identity | Service Accounts + ADC |
| Armazenamento de secrets | Secrets Manager | Key Vault | Secret Manager |
| Chaves de criptografia | KMS | Key Vault | Cloud KMS |
| Web application firewall | WAF | Application Gateway WAF / Front Door WAF | Cloud Armor |
| Proteção DDoS | Shield | DDoS Protection | Cloud Armor |

**Filosofia de identidade:**
- **AWS IAM Roles:** Associa a EC2/ECS/Lambda. O SDK descobre as credenciais via metadata service. Funciona bem; a configuração é mais explícita.
- **Azure Managed Identity:** O padrão ouro. Zero gerenciamento de credenciais. `DefaultAzureCredential` descobre a identidade automaticamente em qualquer contexto Azure. Funciona com Key Vault, Blob Storage, Service Bus — tudo.
- **GCP ADC (Application Default Credentials):** Equivalente à abordagem do Azure. `gcloud auth application-default login` para desenvolvimento local; service account associada ao Cloud Run/GKE para produção. Zero credenciais no código ou em variáveis de ambiente.

### Rede

| Conceito | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Rede privada | VPC | Virtual Network (VNet) | VPC |
| Load balancer (HTTP/L7) | Application Load Balancer (ALB) | Application Gateway | Cloud Load Balancing |
| Load balancer global | ALB + Route 53 | Azure Front Door | Cloud Load Balancing (global por padrão) |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| DNS privado | Route 53 Resolver | Azure Private DNS | Cloud DNS (zonas privadas) |
| VPN | Site-to-Site VPN | VPN Gateway | Cloud VPN |
| Conectividade privada | Direct Connect | ExpressRoute | Cloud Interconnect |

**O load balancer global do GCP é genuinamente diferente:** os load balancers da AWS e do Azure são regionais por padrão (você compõe múltiplos LBs regionais para roteamento global). O Cloud Load Balancing do GCP é global por design — um único endereço IP, roteamento anycast, requisições chegam ao ponto de presença Google mais próximo.

### Operações

| Conceito | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Métricas e monitoramento | CloudWatch | Azure Monitor | Cloud Monitoring |
| Logging | CloudWatch Logs | Log Analytics / Azure Monitor Logs | Cloud Logging |
| APM (Application Performance Monitoring) | CloudWatch + X-Ray | Application Insights | Cloud Trace + Error Reporting |
| Infrastructure as Code | CloudFormation / CDK | ARM Templates / Bicep | Deployment Manager / Terraform |
| CI/CD gerenciado | CodePipeline + CodeBuild | Azure DevOps / GitHub Actions | Cloud Build + Cloud Deploy |
| Registro de artifacts | ECR | Azure Container Registry | Artifact Registry |

**Recomendação para IaC:**
- Se usar apenas um cloud: CDK (AWS), Bicep (Azure), Deployment Manager (GCP) — ferramentas nativas com melhor suporte de primeira classe
- Se multi-cloud ou já conhece: Terraform funciona nos três

---

## Exemplos Práticos

### Fazendo Deploy da Mesma API Node.js nos Três Clouds

Considere uma API Node.js/TypeScript em um container Docker, escutando na porta 3000, conectando ao PostgreSQL.

**AWS (ECS Fargate):**
```bash
# Enviar para o ECR
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
docker push $ECR_URI/myapp:latest

# Atualizar o serviço ECS (assume que o cluster e o serviço já existem)
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --force-new-deployment

# Verificar o rollout
aws ecs wait services-stable --cluster myapp-cluster --services myapp-api
```

**Azure (Container Apps):**
```bash
# Build no ACR
az acr build --registry myregistry --image myapp:$SHA .

# Atualizar a container app
az containerapp update \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --image myregistry.azurecr.io/myapp:$SHA
```

**GCP (Cloud Run):**
```bash
# Build com Cloud Build
gcloud builds submit --tag us-central1-docker.pkg.dev/myapp-prod/myapp/api:$SHA .

# Deploy
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:$SHA \
  --region us-central1
```

Todas as três abordagens fazem deploy de um container. As principais diferenças são o registro de imagens (ECR, ACR, Artifact Registry) e o mecanismo de atualização.

### Lendo um Secret em Node.js

```typescript
// AWS
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
const client = new SecretsManagerClient({ region: 'us-east-1' });
const { SecretString } = await client.send(new GetSecretValueCommand({ SecretId: 'myapp/db-url' }));

// Azure
import { SecretClient } from '@azure/keyvault-secrets';
import { DefaultAzureCredential } from '@azure/identity';
const client = new SecretClient('https://myapp-kv.vault.azure.net', new DefaultAzureCredential());
const { value } = await client.getSecret('database-url');

// GCP
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';
const client = new SecretManagerServiceClient();
const [version] = await client.accessSecretVersion({
  name: 'projects/myapp-prod/secrets/database-url/versions/latest',
});
const value = version.payload!.data!.toString();
```

Os três usam o SDK do cloud com credenciais descobertas automaticamente do ambiente. O padrão é idêntico em essência.

---

## Padrões e Boas Práticas

### Quando Escolher AWS

- O time já tem expertise ou certificações em AWS
- Precisa do catálogo de serviços mais amplo (Kinesis, SageMaker, Rekognition, etc.)
- Startup focada nos EUA contratando desenvolvedores — AWS é a expectativa padrão do mercado
- Precisa do Kubernetes gerenciado mais maduro (EKS é mais battle-tested para clusters grandes em alguns cenários, embora o GKE esteja se aproximando)
- Uso intenso de serverless: Lambda tem o ecossistema mais maduro (layers, Lambda@Edge, runtimes customizados)

### Quando Escolher Azure

- Organização é fortemente Microsoft: Active Directory, Office 365, Windows Server, .NET
- Requisitos de compliance empresarial: Azure tem o maior número de certificações governamentais e de indústria
- Hybrid cloud: Azure Arc é a melhor solução para workloads on-premises + cloud
- Usa GitHub (da Microsoft): GitHub Actions + Azure com OIDC é integração impecável
- Workloads SQL Server: Azure SQL é o destino natural

### Quando Escolher GCP

- Workloads de dados e ML: BigQuery, Vertex AI, Dataflow são líderes do setor
- Uso intenso de Kubernetes: GKE inventou o Kubernetes; é o K8s gerenciado mais maduro
- Startup que valoriza experiência do desenvolvedor: ADC + Cloud Run + Cloud Build é o DX mais fluido
- Integração com Google Workspace (Gmail, Drive, Sheets)
- Precisa da melhor rede global: o backbone privado do GCP genuinamente reduz a latência para usuários globais

### Quando a Escolha do Provider Não Importa Muito

Para um SaaS Node.js típico (API + PostgreSQL + Redis + object storage + CDN), os três providers entregam resultados idênticos a custos similares. O diferenciador é a familiaridade do time e o fit com o ecossistema, não a capacidade técnica.

---

## Anti-Padrões a Evitar

### Multi-Cloud por Padrão

```
RUIM: "Devemos usar múltiplos clouds para evitar vendor lock-in."

Realidade:
- Multi-cloud aumenta a complexidade operacional em 3-5x
- Cada cloud tem networking, IAM, billing e tooling diferentes
- Pouquíssimos times têm expertise para operar múltiplos clouds bem
- Camadas de abstração (Terraform, Crossplane) ajudam mas não eliminam a complexidade

BOM: Escolha um cloud para sua carga de trabalho principal.
     Use um segundo cloud apenas quando houver uma razão técnica forte
     (ex.: GCP BigQuery para analytics + AWS para a aplicação).
```

### Ignorar Custos de Egresso

Os três providers cobram por dados saindo da rede deles:
- AWS: $0,09/GB após os primeiros 100GB/mês (tier gratuito)
- Azure: $0,087/GB
- GCP: $0,12/GB (para a internet)

Para aplicações que transferem terabytes de dados por mês, os custos de egresso podem superar os custos de compute. Meça antes de assumir que é negligenciável.

### Assumir que "Gerenciado" Significa "Zero Ops"

Serviços gerenciados reduzem a carga operacional, mas não a eliminam:
- RDS/Cloud SQL ainda precisa de monitoramento de storage e tuning de parâmetros
- ElastiCache/Redis ainda precisa de monitoramento de memória e política de eviction
- EKS/GKE ainda precisa de gerenciamento de node pools e agendamento de upgrades

"Gerenciado" significa que o provider cuida do patching do OS e do hardware. Você ainda é responsável pela configuração, capacidade e performance.

---

## Debugging e Troubleshooting

### Referência de Tooling por Provider

**AWS:**
```bash
# Saúde do serviço
aws health describe-events --filter eventStatusCodes=open

# Verificar cotas de serviço
aws service-quotas list-service-quotas --service-code ecs

# Investigação de pico de custo
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

**Azure:**
```bash
# Saúde do serviço
az resource show --ids /subscriptions/SUB/providers/Microsoft.ResourceHealth/...

# Activity log (trilha de auditoria)
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --start-time 2024-01-01 \
  --output table

# Custo por serviço
az consumption usage list \
  --start-date 2024-01-01 --end-date 2024-01-31 \
  --query "[].{Service:instanceName,Cost:pretaxCost}" \
  --output table
```

**GCP:**
```bash
# Saúde do serviço
gcloud alpha monitoring services list

# Logs de auditoria
gcloud logging read \
  'logName="projects/myapp-prod/logs/cloudaudit.googleapis.com%2Factivity"' \
  --limit 50

# Exportação de custos (requer billing export para BigQuery habilitado)
# Query no console do BigQuery:
# SELECT service.description, SUM(cost) as total
# FROM `myapp-billing.billing_export.gcp_billing_export_v1_*`
# WHERE DATE(usage_start_time) BETWEEN '2024-01-01' AND '2024-01-31'
# GROUP BY 1 ORDER BY 2 DESC
```

---

## Cenários Reais

### Cenário 1: Startup Escolhendo seu Primeiro Cloud

```
Contexto:
  - 3 engenheiros, todos com alguma experiência em AWS
  - API Node.js + SPA React + PostgreSQL
  - Orçamento: $200/mês inicialmente
  - Plano de crescer e escalar em 18 meses

Recomendação: AWS

Motivos:
  - Familiaridade do time → mais rápido para produção
  - A maioria dos candidatos tem experiência em AWS
  - O free tier é generoso para estágio inicial
  - ECS Fargate + RDS + S3 é bem documentado

Estimativa de custo:
  ECS Fargate (0,5 vCPU, 1GB): ~$25/mês
  RDS db.t3.micro PostgreSQL: ~$13/mês
  S3 + CloudFront: ~$5/mês
  Route 53: ~$1/mês
  Total: ~$44/mês
```

### Cenário 2: Empresa Migrando do On-Premises

```
Contexto:
  - Empresa de 500 pessoas, uso intenso de Active Directory
  - Requisitos de compliance (SOC 2, ISO 27001)
  - Time de TI familiarizado com Windows Server e SQL Server
  - Workloads mistos de .NET e Node.js

Recomendação: Azure

Motivos:
  - Azure AD integra diretamente com o AD on-premises (Azure AD Connect)
  - Azure tem o maior número de certificações de compliance
  - Azure SQL é o destino natural para workloads SQL Server
  - Managed Identity elimina o gerenciamento de credenciais em toda a plataforma
  - Azure DevOps + GitHub Actions + Azure é a melhor história end-to-end
```

### Cenário 3: Plataforma Analítica com Grandes Volumes de Dados

```
Contexto:
  - Processamento de 50GB+ de eventos por dia
  - Treinamento e serving de modelos ML
  - Dashboards de analytics para clientes

Recomendação: GCP

Motivos:
  - BigQuery é genuinamente o melhor data warehouse gerenciado
  - Vertex AI é forte para treinamento e serving de ML
  - Dataflow para processamento de streaming
  - Cloud Run para a camada de API
  - Preço competitivo para transferência de dados dentro do GCP (BigQuery → Cloud Run é gratuito)
```

---

## Leitura Adicional

- [Comparação de calculadoras de preço AWS vs Azure vs GCP](https://cloudprice.net/)
- [Comparação de SLA dos providers](https://uptime.is/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [CNCF Landscape — tooling cloud-native entre providers](https://landscape.cncf.io/)

---

## Resumo

| Dimensão | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Participação de mercado | Maior | Segunda | Terceira |
| Amplitude de serviços | Mais ampla | Muito ampla | Focada |
| Experiência do desenvolvedor | Boa | Boa | Excelente |
| Compliance/enterprise | Forte | Melhor | Crescendo |
| Kubernetes | EKS (sólido) | AKS (bom) | GKE (melhor) |
| Containers serverless | ECS Fargate | Container Apps | Cloud Run |
| Gerenciamento de credenciais | IAM Roles | Managed Identity | ADC + SA |
| Dados/ML | SageMaker | Azure ML | BigQuery + Vertex AI |
| Rede global | Boa | Boa | Excelente |
| Free tier | Generoso | Bom | Bom |
| Melhor fit | Maioria dos casos, maior pool de talentos | Empresas Microsoft, enterprise | Dados/ML, K8s, DX forte |

Não existe escolha universalmente correta. Escolha o provider que seu time conhece e então invista em aprendê-lo profundamente. Conhecimento superficial de múltiplos clouds é pior do que conhecimento profundo de um.
