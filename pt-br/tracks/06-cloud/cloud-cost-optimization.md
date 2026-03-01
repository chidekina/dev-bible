# Otimização de Custos em Cloud

## Visão Geral

Os custos em cloud têm uma forma de crescer silenciosamente e então te surpreender. Um Lambda mal configurado, um bucket S3 acumulando logs para sempre, um banco de dados com 4x os IOPS provisionados de que precisa — esses problemas se acumulam em faturas que ultrapassam em muito as estimativas originais.

Otimização de custos não é ser barato. É pagar pelo que você realmente usa e não pagar pelo que não usa. As técnicas deste capítulo se aplicam a AWS, Azure e GCP e são baseadas em padrões reais de deploys backend com Node.js.

---

## Pré-requisitos

- AWS Core Services (`aws-core-services.md`) ou capítulo equivalente Azure/GCP
- Compreensão dos padrões de tráfego e uso de recursos da sua aplicação

---

## Conceitos Fundamentais

### O Modelo Mental FinOps

A otimização de custos em cloud segue um framework chamado FinOps:

```
Informar → Otimizar → Operar

Informar:  Entender o que você está gastando e por quê
Otimizar:  Aplicar mudanças para reduzir desperdício
Operar:    Monitorar continuamente e repetir
```

O maior erro dos times é pular "Informar" e ir direto para a otimização. Sem visibilidade, você otimiza as coisas erradas.

### De Onde Vêm as Faturas Cloud

Para um SaaS Node.js típico, as principais categorias de custo são:

```
1. Compute (EC2, ECS, App Service, Cloud Run)     40-60% da fatura
2. Banco de dados (RDS, Cloud SQL, Cosmos DB)     20-35%
3. Transferência de dados / egresso               5-20%
4. Storage (S3, Blob, Cloud Storage)              2-10%
5. Rede (Load balancers, NAT, VPN)                3-10%
6. Serviços gerenciados (ElastiCache, SQS, etc.)  2-10%
```

Comece por compute e bancos de dados — é onde vivem mais de 70% da maioria das faturas.

### Economia Unitária

A métrica certa não é o custo total, mas o custo por unidade de valor:

```
Custo por usuário ativo por mês
Custo por requisição de API
Custo por GB processado
Custo por transação
```

Uma fatura de $10.000/mês é aceitável se você está servindo 100.000 clientes pagantes. Uma fatura de $500/mês é um problema se você tem 10 usuários gratuitos.

---

## Exemplos Práticos

### Passo 1: Obter Visibilidade Primeiro

**AWS:**
```bash
# Custo por serviço nos últimos 30 dias
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}' \
  --output table

# Custo por tag (requer que todos os recursos estejam tagueados)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=TAG,Key=Environment

# Habilitar Cost Explorer (faça isso hoje se ainda não fez)
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Environment,Status=Active
```

**Azure:**
```bash
# Custo por resource group
az consumption usage list \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --query "sort_by([].{Name:instanceName,Cost:pretaxCost,Service:product}, &Cost)" \
  --output table

# Configurar alertas de custo antes de fazer qualquer outra coisa
az consumption budget create \
  --budget-name myapp-monthly \
  --amount 500 \
  --time-grain Monthly \
  --start-date 2024-01-01 \
  --end-date 2025-01-01 \
  --resource-group myapp-prod-rg
```

**GCP:**
```bash
# Habilitar exportação de billing para BigQuery (configuração única)
# Depois consultar:
# SELECT service.description, SUM(cost) as total_cost
# FROM `project.dataset.gcp_billing_export_v1_*`
# WHERE DATE(usage_start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
# GROUP BY 1 ORDER BY 2 DESC

# Recomendações
gcloud recommender recommendations list \
  --recommender google.compute.instance.MachineTypeRecommender \
  --location us-central1 \
  --project myapp-prod
```

### Passo 2: Taguear Tudo

Tags/labels são a base da atribuição de custos. Sem elas, você não consegue responder "quanto custa o serviço de API?" ou "quanto custa o ambiente de staging?"

**Estratégia de tags na AWS:**
```bash
# Taguear todos os recursos de forma consistente
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:ecs:us-east-1:123456789012:service/myapp-cluster/myapp-api \
  --tags \
    Project=myapp \
    Environment=production \
    Team=backend \
    Service=api \
    CostCenter=engineering

# Habilitar atribuição de custo baseada em tag
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status \
    TagKey=Project,Status=Active \
    TagKey=Environment,Status=Active \
    TagKey=Team,Status=Active

# Encontrar recursos sem tag
aws resourcegroupstaggingapi get-resources \
  --tag-filters '[]' \
  --query 'ResourceTagMappingList[?Tags==`[]`].ResourceARN' \
  --output text
```

**Terraform: impor tags por padrão:**
```hcl
# variables.tf
variable "default_tags" {
  default = {
    Project     = "myapp"
    Environment = "production"
    Team        = "backend"
    ManagedBy   = "terraform"
  }
}

# main.tf
provider "aws" {
  default_tags {
    tags = var.default_tags
  }
}
```

### Passo 3: Redimensionar Compute

Rodar instâncias superdimensionadas é uma das fontes mais comuns de desperdício.

**AWS — usar Compute Optimizer:**
```bash
# Obter recomendações para instâncias EC2
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    Finding: finding,
    RecommendedType: recommendationOptions[0].instanceType,
    MonthlySavings: recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table

# Obter recomendações para serviços ECS
aws compute-optimizer get-ecs-service-recommendations \
  --query 'ecsServiceRecommendations[].{
    Service: serviceArn,
    Finding: finding,
    CurrentCPU: currentServiceConfiguration.cpu,
    RecommendedCPU: serviceRecommendationOptions[0].containerRecommendations[0].cpu[0].upperBound,
    MonthlySavings: serviceRecommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value
  }'
```

**Redimensionamento prático para Node.js:**
```
Node.js é single-threaded → raramente se beneficia de mais de 2 vCPU por processo
                           → escale horizontalmente (mais instâncias), não verticalmente (instâncias maiores)

Ponto de partida para ECS Fargate:
  API (CRUD, baixo compute):    0,25-0,5 vCPU, 512MB-1GB de memória
  API (processamento intenso):  1-2 vCPU, 1-2GB de memória

Sinais de que você está superdimensionado:
  - CPU p95 < 20% consistentemente
  - Uso de memória < 30% consistentemente
  - Eventos de scale nunca são disparados

Sinais de que você está subdimensionado:
  - CPU p95 > 80%
  - OOM kills nos logs
  - Eventos de scale-out frequentes, tasks nunca fazem scale-in
```

### Passo 4: Comprar Capacidade Reservada

Após 2-3 meses de uso estável em produção, capacidade reservada economiza 30-60% no compute.

**AWS Savings Plans (flexível, recomendado em vez de Reserved Instances):**
```bash
# Verificar o gasto on-demand por hora para determinar o valor do compromisso
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-08 \
  --granularity DAILY \
  --metrics UsageQuantity UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}'

# Comprar via console (Compute Savings Plans cobre EC2, Fargate, Lambda)
# Economias típicas:
#   1 ano, sem adiantamento:  27% de economia
#   1 ano, tudo adiantado:    40% de economia
#   3 anos, tudo adiantado:   60% de economia
```

**Azure Reserved VM Instances:**
```bash
# Comprar via portal: Cost Management + Billing → Reservations
# Ou via CLI:
az reservations reservation-order purchase \
  --reservation-order-id ORDER_ID \
  --sku Standard_D2s_v3 \
  --location eastus \
  --reserved-resource-type VirtualMachines \
  --term P1Y \                # 1 ano
  --billing-scope /subscriptions/SUB \
  --quantity 2 \
  --applied-scope-type Shared
```

**GCP Committed Use Discounts:**
```bash
# Compromisso de 1 ano em CPU do Cloud Run
gcloud compute commitments create myapp-commitment \
  --plan 12-month \
  --resources vcpu=4,memory=16GB \
  --region us-central1

# Economias típicas no Cloud Run:
#   On-demand:        $0,00002400/vCPU-segundo
#   Compromisso 1yr:  $0,00001720/vCPU-segundo (28% de economia)
#   Compromisso 3yr:  $0,00001370/vCPU-segundo (43% de economia)
```

### Passo 5: Otimizar Storage

**Políticas de ciclo de vida para S3 / Cloud Storage:**
```bash
# AWS — regra de ciclo de vida: mover para IA após 30 dias, Glacier após 90
aws s3api put-bucket-lifecycle-configuration \
  --bucket myapp-uploads \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "move-to-cheaper-storage",
      "Status": "Enabled",
      "Filter": {},
      "Transitions": [
        {"Days": 30, "StorageClass": "STANDARD_IA"},
        {"Days": 90, "StorageClass": "GLACIER_IR"},
        {"Days": 365, "StorageClass": "DEEP_ARCHIVE"}
      ],
      "Expiration": {"Days": 2555}
    }]
  }'

# Deletar uploads multipart incompletos (se acumulam silenciosamente)
aws s3api put-bucket-lifecycle-configuration \
  --bucket myapp-uploads \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "abort-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
  }'

# GCP — configuração de ciclo de vida
cat > lifecycle.json << 'EOF'
{
  "lifecycle": {
    "rule": [
      {
        "action": {"type": "SetStorageClass", "storageClass": "NEARLINE"},
        "condition": {"age": 30, "matchesStorageClass": ["STANDARD"]}
      },
      {
        "action": {"type": "SetStorageClass", "storageClass": "COLDLINE"},
        "condition": {"age": 90, "matchesStorageClass": ["NEARLINE"]}
      },
      {
        "action": {"type": "Delete"},
        "condition": {"age": 2555}
      }
    ]
  }
}
EOF

gcloud storage buckets update gs://myapp-uploads --lifecycle-file=lifecycle.json
```

**Retenção de logs:**
```bash
# AWS CloudWatch Logs: definir retenção em todos os log groups
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].logGroupName' \
  --output text | while read -r lg; do
    aws logs put-retention-policy \
      --log-group-name "$lg" \
      --retention-in-days 30
done

# Azure: retenção do workspace Log Analytics
az monitor log-analytics workspace update \
  --workspace-name myapp-workspace \
  --resource-group myapp-prod-rg \
  --retention-time 30
```

### Passo 6: Reduzir Transferência de Dados / Egresso

A transferência de dados é frequentemente um custo ignorado, especialmente conforme as aplicações crescem.

**Coloque seu CDN na frente de tudo:**
```bash
# AWS — CloudFront na frente do S3 e do ALB
# CloudFront → S3: $0 de transferência (mesma rede)
# CloudFront → Usuários: $0,0085/GB (muito mais barato que o egresso do ALB a $0,08/GB)

# Habilitar cache para respostas de API que podem ser cacheadas
# Na sua API Node.js:
app.get('/api/products', async (req, reply) => {
  const products = await getProducts();
  reply
    .header('Cache-Control', 'public, max-age=300, stale-while-revalidate=60')
    .send(products);
});

# Para dados específicos de usuário: usar TTLs curtos ou cache privado
reply.header('Cache-Control', 'private, max-age=60');
```

**Manter o tráfego dentro da rede cloud:**
```bash
# RUIM: RDS em us-east-1, app chamando API externa em eu-west-1 → egresso entre regiões
# BOM: Usar VPC endpoints para manter o tráfego interno

# AWS: S3 VPC endpoint (gratuito, evita egresso pela internet)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-abc123

# Mesmo princípio: usar private endpoints para serviços gerenciados do Azure/GCP
```

### Passo 7: Escalar para Zero em Ambientes Não-Produtivos

Ambientes de desenvolvimento e staging não deveriam rodar 24/7 em escala de produção.

```bash
# AWS: escalar serviço ECS para 0 fora do horário comercial
# Usando CloudWatch Events (EventBridge) + Lambda

# Criar Lambda para escalar o serviço
aws lambda create-function \
  --function-name scale-nonprod \
  --runtime nodejs22.x \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::123456789012:role/lambda-role

# Agendamento: escalar para baixo às 20h, escalar para cima às 8h nos dias úteis
aws events put-rule \
  --name scale-down-nonprod \
  --schedule-expression "cron(0 23 ? * MON-FRI *)"    # 20h EST = 23:00 UTC

aws events put-rule \
  --name scale-up-nonprod \
  --schedule-expression "cron(0 12 ? * MON-FRI *)"    # 8h EST = 12:00 UTC

# Economia: ~65% de redução nos custos de compute não-produtivo
# (rodando 12h nos dias úteis vs 24/7 = 36h vs 168h por semana)
```

```bash
# Azure: Container Apps pode escalar para 0 com min-replicas=0
az containerapp update \
  --name myapp-api-staging \
  --resource-group myapp-staging-rg \
  --min-replicas 0   # escala para zero quando não há tráfego

# GCP: Cloud Run escala para zero por padrão
# Definir min-instances=0 para staging (aceitar a penalidade de cold start)
gcloud run services update myapp-api-staging \
  --region us-central1 \
  --min-instances 0
```

---

## Padrões e Boas Práticas

### O Checklist de Otimização

Execute mensalmente:

```
[ ] Recursos não utilizados        — instâncias EC2 paradas, volumes EBS/discos desanexados
[ ] Snapshots órfãos               — snapshots EBS/disk com mais de 90 dias
[ ] Load balancers vazios          — LBs sem targets saudáveis (ainda são cobrados)
[ ] EIPs desanexados               — Elastic IPs não associados a uma instância ($3,65/mês cada)
[ ] Bancos de dados ociosos        — RDS com menos de 1 conexão/hora por uma semana
[ ] Retenção de logs               — CloudWatch logs sem expiração definida
[ ] Classes de storage S3          — objetos sem acesso por 90+ dias em tier Standard
[ ] Timeouts Lambda                — funções com timeout > p99 da função (pagando a mais por compute)
[ ] Imagens ECR não utilizadas     — centenas de versões antigas de imagens de container
[ ] Transferência de dados         — o egresso está inesperadamente alto? Adicionou uma nova dependência externa?
```

**Encontrar recursos não utilizados com scripts AWS:**
```bash
# Volumes EBS desanexados
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Age:CreateTime}' \
  --output table

# Instâncias EC2 com CPU médio < 5% (candidatas para redimensionamento)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-abc123 \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average

# NAT Gateways não utilizados (fonte comum de desperdício em ambientes de dev)
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available \
  --query 'NatGateways[].{ID:NatGatewayId,VPC:VpcId,Created:CreateTime}'
```

### Otimização de Custos de Banco de Dados

Bancos de dados são frequentemente superprovisionados porque os times têm medo de tempo de indisponibilidade durante o redimensionamento.

```bash
# AWS RDS: redimensionar com mínimo de downtime (failover Multi-AZ)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --db-instance-class db.t3.medium \    # reduzindo de db.m5.large
  --apply-immediately                   # ou --no-apply-immediately para a próxima janela de manutenção

# Monitorar para validar que o redimensionamento funcionou
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-prod \
  --start-time $(date -d '24 hours ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Average Maximum

# ElastiCache: verificar uso de memória antes de redimensionar
aws cloudwatch get-metric-statistics \
  --namespace AWS/ElastiCache \
  --metric-name DatabaseMemoryUsagePercentage \
  --dimensions Name=CacheClusterId,Value=myapp-cache \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Maximum
```

---

## Anti-Padrões a Evitar

### Otimizar Sem Medir

```
RUIM: "Acho que o Lambda está muito caro, vamos migrar para ECS"
     → Migrou para ECS, custos aumentaram (tinha esquecido do tempo ocioso do Fargate)

BOM: Puxar os custos reais do Lambda → comparar com o modelo de custo do ECS Fargate
     para a mesma carga de trabalho → então decidir
```

### Otimização Prematura

Não otimize com $50/mês. Com $50/mês, 20% de economia é $10/mês — menos de uma hora de engenharia. Otimize quando:
- A fatura mensal superar $500 (mesmo assim, comece pela visibilidade)
- Um serviço específico tiver custos inesperadamente altos
- Você estiver planejando compras de capacidade reservada

### Deletar Sem Tirar Snapshots

```bash
# Sempre tirar snapshot antes de deletar
aws ec2 create-snapshot \
  --volume-id vol-abc123 \
  --description "Backup antes da exclusão $(date)"

# Então deletar
aws ec2 delete-volume --volume-id vol-abc123
```

### Não Definir Políticas de Ciclo de Vida Antes do Storage Crescer

Storage é fácil de ignorar a $0,023/GB. Com 1TB, são $23/mês. Com 10TB (não incomum para uma aplicação coletando imagens/logs por alguns anos), são $230/mês — mais custos de recuperação se você moveu para cold storage sem um plano.

---

## Debugging e Troubleshooting

### Investigação de Pico de Custo Inesperado

```bash
# AWS: encontrar o pico no Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table

# Culpados comuns:
# - Transferência de dados: você adicionou uma chamada de API externa que retorna payloads grandes?
# - EC2: o Auto Scaling criou muitas instâncias e não fez scale-down?
# - RDS: uma migração ou carregamento de dados causou um pico de IOPS?
# - Requisições S3 PUT: você introduziu um bug que escreve no S3 em loop?
# - Lambda: um Lambda ficou preso em um loop infinito de retentativas?

# Verificar métricas de erros do Lambda
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=myapp-processor \
  --start-time $(date -d '3 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Sum
```

### Custos de Egresso Maiores que o Esperado

```bash
# Verificar transferência de dados do CloudFront
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name BytesDownloaded \
  --dimensions Name=DistributionId,Value=E1234567890ABC \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Sum

# Verificar se a taxa de cache hit está baixa (hit baixo = mais fetches na origem = mais compute + egresso)
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=E1234567890ABC \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average
```

---

## Cenários Reais

### Cenário 1: Reduzindo uma Fatura AWS de $2.000/mês em 40%

```
Detalhamento inicial da fatura:
  EC2 (m5.large × 4, 24/7):          $560/mês
  RDS (db.m5.large Multi-AZ):        $380/mês
  ElastiCache (r6g.large):           $180/mês
  Transferência de dados:             $340/mês
  S3:                                 $220/mês
  Diversos:                           $320/mês
  Total:                             $2.000/mês

Ações realizadas:
  1. Redimensionou EC2 para m5.medium (CPU médio era 18%): -$280/mês
  2. Redimensionou RDS para db.t3.large (mesma performance, burstable): -$120/mês
  3. Comprou Compute Savings Plans de 1 ano no EC2: -$90/mês
  4. Adicionou CloudFront → cortou transferência de dados em 60%: -$200/mês
  5. Configurou ciclo de vida S3 para mover objetos antigos para IA/Glacier: -$80/mês
  6. Definiu retenção de 30 dias nos CloudWatch logs: -$40/mês

Resultado: $1.190/mês (redução de 41% em 3 semanas de trabalho)
```

### Cenário 2: Modelo de Custo do Cloud Run para uma API SaaS

```typescript
// Preços do Cloud Run:
//   CPU: $0,00002400/vCPU-segundo (durante o processamento de requisições)
//   Memória: $0,00000250/GiB-segundo
//   Requisições: $0,40 por milhão

// Para uma API processando 1M requisições/mês, 200ms de tempo de resposta médio:

const requestsPerMonth = 1_000_000;
const avgResponseTimeSeconds = 0.2;
const vCPU = 1;
const memoryGiB = 0.5;

const cpuCost = requestsPerMonth * avgResponseTimeSeconds * vCPU * 0.000024;
// = 1.000.000 * 0,2 * 1 * 0,000024 = $4,80

const memoryCost = requestsPerMonth * avgResponseTimeSeconds * memoryGiB * 0.0000025;
// = 1.000.000 * 0,2 * 0,5 * 0,0000025 = $0,25

const requestCost = (requestsPerMonth / 1_000_000) * 0.40;
// = 1 * 0,40 = $0,40

const totalCost = cpuCost + memoryCost + requestCost;
// = $5,45/mês

// A 10M de requisições: ~$55/mês
// A 100M de requisições: ~$550/mês
// Escala previsível e linear — sem necessidade de capacidade reservada
```

---

## Leitura Adicional

- [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/)
- [AWS Savings Plans](https://aws.amazon.com/savingsplans/)
- [Azure Cost Management](https://azure.microsoft.com/services/cost-management/)
- [GCP Cost Management](https://cloud.google.com/cost-management)
- [infracost — estimar custos de IaC antes do deploy](https://www.infracost.io/)
- [The FinOps Framework](https://www.finops.org/framework/)

---

## Resumo

| Ação | Esforço | Economia Típica |
|------|---------|-----------------|
| Habilitar alertas de custo + orçamentos | 30 min | Evita susto na fatura |
| Taguear todos os recursos | 1-2 dias | Permite atribuição de custos |
| Revisar e redimensionar compute | 2-4 horas | 20-40% no compute |
| Comprar capacidade reservada (1 ano) | 1 hora | 27-40% nos serviços comprometidos |
| Definir políticas de ciclo de vida S3/GCS | 1 hora | 30-70% no storage |
| Definir políticas de retenção de logs | 30 min | 20-60% no logging |
| Escalar ambientes não-produtivos para zero | 2-4 horas | 60-80% no compute não-produtivo |
| Adicionar caching CDN | 2-4 horas | 40-80% na transferência de dados |
| Limpar recursos órfãos | 2-3 horas | Varia |

Comece pela visibilidade (tags + alertas de custo). Depois redimensione. Só compre capacidade reservada após o seu uso estar estável e medido. Você não consegue otimizar o que não consegue medir.
