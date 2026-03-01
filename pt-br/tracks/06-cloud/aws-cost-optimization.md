# Otimização de Custos na AWS

## Visão Geral

As contas da AWS crescem de formas previsíveis: compute superdimensionado, bancos de dados com IOPS provisionados que você não usa, buckets S3 sem lifecycle rules e custos de transferência de dados que aparecem de surpresa conforme sua aplicação escala. A boa notícia é que cada um desses problemas tem uma solução clara.

Este capítulo cobre técnicas de otimização de custos específicas da AWS para deployments backend Node.js. Complementa `cloud-cost-optimization.md` (estratégias cross-cloud) com ferramentas, modelos de precificação e comandos CLI específicos da AWS.

---

## Pré-requisitos

- Serviços fundamentais da AWS (`aws-core-services.md`)
- Conta AWS com Cost Explorer habilitado
- Entendimento básico dos padrões de tráfego da sua aplicação

---

## Conceitos Fundamentais

### Modelos de Precificação da AWS

Entender os modelos de precificação é essencial antes de otimizar:

```
On-Demand:       Pague por hora/segundo. Sem compromisso. Preço mais alto.
                 Use para: cargas variáveis, desenvolvimento, novas aplicações

Savings Plans:   Comprometa-se com $/hora em EC2, Fargate, Lambda.
                 Compute Savings Plans: 1ano/3anos, sem-adiantamento/parcial/total.
                 Economia de até 66% vs on-demand. Mais flexível.

Reserved Instances: Comprometa-se com tipo de instance/region específicos.
                 Economia de até 72% vs on-demand. Menos flexível que Savings Plans.
                 Use para: RDS, ElastiCache, Redshift (Savings Plans não cobrem esses).

Spot Instances:  Capacidade ociosa da AWS com desconto de até 90%.
                 Pode ser interrompido com 2 minutos de aviso.
                 Use para: batch jobs, workers de CI/CD, cargas stateless fault-tolerant.

Free Tier:       Sempre gratuito: Lambda 1M requisições/mês, DynamoDB 25GB, S3 5GB.
                 Gratuito por 12 meses: EC2 t2.micro 750h/mês, RDS t2.micro 750h/mês.
```

### O Conjunto de Ferramentas de Otimização de Custos da AWS

```
Cost Explorer:          Visualize e analise gastos. Histórico e previsão.
AWS Budgets:            Alertas quando você ultrapassa limites. Configure no primeiro dia.
Compute Optimizer:      Recomendações de right-sizing com ML para EC2, ECS, Lambda.
Trusted Advisor:        Verificações de boas práticas incluindo otimização de custos.
Cost Anomaly Detection: Alertas baseados em ML quando gastos desviam do normal.
AWS Pricing Calculator: Estime custos antes de fazer deploy.
```

---

## Exemplos Práticos

### Passo 1: Configure Visibilidade de Billing

Faça isso antes de qualquer outra coisa. Você não pode otimizar o que não pode ver.

```bash
# Habilitar Cost Explorer (única vez — leva 24h para popular)
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active TagKey=Environment,Status=Active TagKey=Team,Status=Active

# Criar orçamento com alertas de e-mail a 80% e 100%
aws budgets create-budget \
  --account-id $(aws sts get-caller-identity --query Account --output text) \
  --budget '{
    "BudgetName": "monthly-total",
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY",
    "BudgetLimit": {"Amount": "500", "Unit": "USD"}
  }' \
  --notifications-with-subscribers '[
    {
      "Notification": {
        "NotificationType": "ACTUAL",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 80,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "admin@myapp.com"}]
    },
    {
      "Notification": {
        "NotificationType": "FORECASTED",
        "ComparisonOperator": "GREATER_THAN",
        "Threshold": 100,
        "ThresholdType": "PERCENTAGE"
      },
      "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "admin@myapp.com"}]
    }
  ]'

# Habilitar Cost Anomaly Detection
aws ce create-anomaly-monitor \
  --anomaly-monitor '{
    "MonitorName": "service-monitor",
    "MonitorType": "DIMENSIONAL",
    "MonitorDimension": "SERVICE"
  }'

aws ce create-anomaly-subscription \
  --anomaly-subscription '{
    "SubscriptionName": "daily-alerts",
    "MonitorArnList": ["arn:aws:ce::123456789012:anomalymonitor/MONITOR_ID"],
    "Subscribers": [{"Address": "admin@myapp.com", "Type": "EMAIL"}],
    "Threshold": 20,
    "Frequency": "DAILY"
  }'
```

### Passo 2: Tague Tudo desde o Primeiro Dia

```bash
# Taguear recursos individuais
aws ec2 create-tags \
  --resources i-abc123 \
  --tags \
    Key=Project,Value=myapp \
    Key=Environment,Value=production \
    Key=Team,Value=backend \
    Key=Service,Value=api

# Encontrar todos os recursos sem tags (requer Resource Groups Tagging API)
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance rds:db ecs:service elasticache:cluster \
  --tag-filters '[]' \
  --query 'ResourceTagMappingList[?Tags==`[]`].{ARN:ResourceARN}' \
  --output table

# Impor tags com regra do AWS Config
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "InputParameters": "{\"tag1Key\":\"Environment\",\"tag2Key\":\"Project\",\"tag3Key\":\"Team\"}"
  }'
```

### Passo 3: Right-Size EC2 e ECS

```bash
# Obter recomendações de right-sizing para EC2
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[?finding!=`OPTIMIZED`].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    Finding: finding,
    Recommended: recommendationOptions[0].instanceType,
    CPUSavings: recommendationOptions[0].estimatedMonthlySavings.value,
    CPUUtilP99: utilizationMetrics[?name==`CPU`].statistic[?name==`P99`] | [0] | value
  }' \
  --output table

# Obter recomendações de right-sizing para ECS/Fargate
aws compute-optimizer get-ecs-service-recommendations \
  --query 'ecsServiceRecommendations[?finding!=`Optimized`].{
    Service: serviceArn,
    Finding: finding,
    CurrentCPU: currentServiceConfiguration.cpu,
    CurrentMemory: currentServiceConfiguration.memory,
    MonthlySavings: serviceRecommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value
  }' \
  --output table

# Obter recomendações de right-sizing para Lambda
aws compute-optimizer get-lambda-function-recommendations \
  --query 'lambdaFunctionRecommendations[?finding!=`Optimized`].{
    Function: functionArn,
    Finding: finding,
    CurrentMemory: currentMemorySize,
    RecommendedMemory: memorySizeRecommendationOptions[0].memorySize,
    MonthlySavings: memorySizeRecommendationOptions[0].projectedUtilizationMetrics[0].estimated
  }' \
  --output table
```

**Right-sizing de uma task Fargate:**
```bash
# Passo 1: Revisar utilização real de CPU e memória nas últimas 2 semanas
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions Name=ClusterName,Value=myapp-cluster Name=ServiceName,Value=myapp-api \
  --start-time $(date -d '14 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average Maximum

# Passo 2: Se p99 CPU < 30% → reduza a alocação de CPU pela metade
# Se p99 Memory < 40% → reduza a memória

# Passo 3: Atualizar a task definition
aws ecs register-task-definition \
  --family myapp \
  --cpu 256 \        # era 512, reduzir pela metade economiza ~50%
  --memory 512 \     # era 1024
  ...

# Passo 4: Atualizar o service para usar a nova task definition
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --task-definition myapp:NOVA_REVISAO
```

### Passo 4: Savings Plans — A Ação com Maior ROI

Após 60-90 dias de uso estável em produção, adquira Savings Plans. Geralmente é a ação de otimização de custos com maior ROI disponível.

```bash
# Passo 1: Verificar seu nível de comprometimento on-demand por hora
# (média dos últimos 30 dias de gastos on-demand em EC2 + Fargate + Lambda)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --filter '{
    "Dimensions": {
      "Key": "USAGE_TYPE_GROUP",
      "Values": ["EC2: Running Hours", "ECS: Fargate vCPU Hours", "Lambda: GB-Seconds"]
    }
  }' \
  --query 'ResultsByTime[].Total.UnblendedCost.Amount' \
  --output table

# Passo 2: Usar recomendações de Savings Plans da AWS
aws savingsplans describe-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

# Passo 3: Comprar (faça no console para confirmação)
# Compute Savings Plans cobrem:
#   - EC2 (qualquer tipo de instance, qualquer region)
#   - Fargate (qualquer region)
#   - Lambda (qualquer region)
# 1 ano, sem adiantamento: ~27% de economia
# 1 ano, tudo adiantado: ~40% de economia
# 3 anos, tudo adiantado: ~66% de economia
```

**Estratégia de compra de Savings Plans:**
```
Conservadora (recomendada para a primeira compra):
  Comprometa-se com 70% do seu gasto médio atual por hora.
  On-demand cobre os 30% restantes (cargas variáveis, picos).

Agressiva:
  Comprometa-se com 90% do seu gasto mínimo por hora.
  Maximiza economia, deixa menos margem para crescimento.

Nunca se comprometa com 100%:
  Se você escalar para baixo (mudanças no time, nova arquitetura),
  ainda paga o compromisso. Você pode vender Savings Plans não usados
  no Reserved Instance Marketplace.
```

### Passo 5: Otimização de Custos do RDS

O RDS é frequentemente o segundo maior item de custo para aplicações Node.js.

```bash
# Verificar utilização real do RDS
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-prod \
  --start-time $(date -d '14 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Average Maximum \
  --output table

# Verificar contagem de conexões
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-prod \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Maximum

# Redimensionar instance RDS (baixo downtime com Multi-AZ — failover para standby durante redimensionamento)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --db-instance-class db.t3.medium \     # reduzido de db.m5.large
  --apply-immediately                     # aciona failover imediatamente, ~60s de downtime

# Mudar para gp3 storage (mesma performance, custo menor que gp2)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --storage-type gp3 \
  --apply-immediately
  # gp3: $0,115/GB vs gp2: $0,115/GB mas gp3 inclui 3000 IOPS gratuitos
  # gp2: IOPS adicionais custam $0,10/IOPS-mês → pode economizar significativamente

# Comprar Reserved Instances para RDS (Savings Plans não cobrem RDS)
# 1 ano, tudo adiantado para db.t3.medium PostgreSQL: ~42% de economia
```

**Comparação de custo RDS vs Aurora Serverless v2:**
```
Caso de uso: API com tráfego variável, média de 200 conexões, picos em 800

RDS db.t3.medium (Multi-AZ):
  db.t3.medium: $0,082/hr × 2 (Multi-AZ) × 720h = $118/mês
  Storage: 50GB gp3 = $5,75/mês
  Total: ~$124/mês

Aurora Serverless v2 (0,5-4 ACU, Multi-AZ):
  ACU: $0,12/ACU-hr × média 1,5 ACU × 720h = $130/mês
  Storage: $0,10/GB × 50GB = $5/mês
  Total: ~$135/mês

Conclusão: Aurora Serverless v2 é ~9% mais caro para essa carga
mas autoscala CPU/memória sem janelas de manutenção.
Vale a pena quando a CPU é muito irregular ou o time não quer
gerenciar operações de redimensionamento.
```

### Passo 6: Otimização de Custos do S3

```bash
# Encontrar buckets com grande volume de storage (economia potencial com lifecycle rules)
aws s3api list-buckets --query 'Buckets[].Name' --output text | while read bucket; do
  SIZE=$(aws cloudwatch get-metric-statistics \
    --namespace AWS/S3 \
    --metric-name BucketSizeBytes \
    --dimensions Name=BucketName,Value="$bucket" Name=StorageType,Value=StandardStorage \
    --start-time "$(date -d '2 days ago' --iso-8601)" \
    --end-time "$(date --iso-8601)" \
    --period 86400 \
    --statistics Average \
    --query 'Datapoints[0].Average' --output text 2>/dev/null)
  echo "$bucket: $(echo "scale=2; ${SIZE:-0}/1073741824" | bc) GB"
done

# Habilitar Intelligent-Tiering para buckets com padrões de acesso desconhecidos
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket myapp-uploads \
  --id AllObjects \
  --intelligent-tiering-configuration '{
    "Id": "AllObjects",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'
# Intelligent-Tiering move objetos automaticamente para tiers mais baratos
# Sem penalidade de recuperação para tiers de Acesso Frequente/Infrequente

# Limpar uploads multipart incompletos (custo oculto comum)
aws s3api list-multipart-uploads --bucket myapp-uploads
aws s3api put-bucket-lifecycle-configuration \
  --bucket myapp-uploads \
  --lifecycle-configuration '{
    "Rules": [{
      "ID": "cleanup-incomplete-multipart",
      "Status": "Enabled",
      "Filter": {},
      "AbortIncompleteMultipartUpload": {"DaysAfterInitiation": 7}
    }]
  }'

# Habilitar métricas de requisições para ver se você paga por muitas chamadas de API
aws s3api put-bucket-metrics-configuration \
  --bucket myapp-uploads \
  --id all-objects \
  --metrics-configuration '{"Id": "all-objects"}'
```

### Passo 7: Otimização de Custos do Lambda

Precificação do Lambda:
- $0,0000002 por requisição (primeiro 1M/mês gratuito)
- $0,0000166667 por GB-segundo (primeiros 400.000 GB-segundos/mês gratuitos)

```bash
# Verificar alocação de memória Lambda vs uso real
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name MaxMemoryUsed \
  --dimensions Name=FunctionName,Value=myapp-processor \
  --start-time $(date -d '14 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Maximum Average

# Se MaxMemoryUsed p99 = 180MB e alocação é 512MB → reduza para 256MB
aws lambda update-function-configuration \
  --function-name myapp-processor \
  --memory-size 256   # ~50% de redução de custo, sem impacto na performance

# Verificar cold starts do Lambda (indica over-scaling ou necessidade de concorrência provisionada)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name InitDuration \
  --dimensions Name=FunctionName,Value=myapp-processor \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics SampleCount Maximum Average

# Concorrência provisionada (para funções sensíveis a latência): reduz cold starts
# mas custa mesmo quando ocioso ($0,0000041 por segundo de concorrência provisionada)
aws lambda put-provisioned-concurrency-config \
  --function-name myapp-api \
  --qualifier ALIAS_OR_VERSION \
  --provisioned-concurrent-executions 5

# Lambda Power Tuning: encontre a alocação de memória ideal
# https://github.com/alexcasalboni/aws-lambda-power-tuning
# Executa sua função com tamanhos de memória diferentes e encontra o ponto ideal de custo/performance
```

### Passo 8: Reduza Custos de Transferência de Dados

```bash
# Verificar custos atuais de transferência de dados
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --filter '{
    "Dimensions": {
      "Key": "USAGE_TYPE_GROUP",
      "Values": ["EC2: Data Transfer - Internet (Out)", "CloudFront: Data Transfer"]
    }
  }'

# Configurar VPC endpoints para S3 e DynamoDB (gratuito, elimina cobranças de NAT Gateway)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-abc123 rtb-def456

aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.dynamodb \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-abc123

# Verificar taxa de cache hit do CloudFront (taxa baixa = mais fetches na origem = mais custo)
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=E1234567890ABC \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average

# Se taxa de cache hit < 80%: revise seus headers Cache-Control
# Melhore no Node.js:
# reply.header('Cache-Control', 'public, max-age=300, stale-while-revalidate=60');
```

### Passo 9: NAT Gateway — O Custo Oculto

O NAT Gateway é frequentemente a maior surpresa nas contas da AWS. Custa $0,045/hora + $0,045/GB de dados processados.

```bash
# Verificar custos do NAT Gateway
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Virtual Private Cloud"]}}'

# Otimização comum: use VPC endpoints em vez de rotear pelo NAT
# Serviços com VPC endpoints gratuitos (tipo Gateway): S3, DynamoDB
# Serviços com Interface endpoints (custam $0,01/hr): Secrets Manager, SSM, ECR, CloudWatch

# Para pulls do ECR (camadas de imagens grandes pelo NAT = caro):
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ecr.dkr \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc subnet-def \
  --security-group-ids sg-endpoints

aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.ecr.api \
  --vpc-endpoint-type Interface \
  --subnet-ids subnet-abc subnet-def \
  --security-group-ids sg-endpoints

# Cálculo de custo: Interface endpoint a $0,01/hr × 2 (HA) × 720h = $14,40/mês
# Se pulls do ECR custam >$14,40/mês em cobranças de NAT → vale a pena
```

---

## Padrões e Boas Práticas

### Script de Revisão Mensal de Custos

```bash
#!/bin/bash
# Revisão mensal de custos — execute na primeira semana de cada mês
# Requer jq instalado

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
LAST_MONTH_START=$(date -d 'last month' +%Y-%m-01)
LAST_MONTH_END=$(date +%Y-%m-01)

echo "=== Custo por Serviço ($LAST_MONTH_START a $LAST_MONTH_END) ==="
aws ce get-cost-and-usage \
  --time-period Start=$LAST_MONTH_START,End=$LAST_MONTH_END \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'sort_by(ResultsByTime[0].Groups, &Metrics.UnblendedCost.Amount)[-10:] | [*].{Service:Keys[0], Cost:Metrics.UnblendedCost.Amount}' \
  --output table

echo "=== IPs Elásticos Não Anexados ==="
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocationId}' \
  --output table

echo "=== Volumes EBS Não Anexados ==="
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}' \
  --output table

echo "=== Instances EC2 Paradas ==="
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Stopped:StateTransitionReason}' \
  --output table

echo "=== Snapshots EBS Antigos (>90 dias) ==="
aws ec2 describe-snapshots \
  --owner-ids self \
  --query "Snapshots[?StartTime<'$(date -d '90 days ago' --iso-8601)'].{ID:SnapshotId,Size:VolumeSize,Created:StartTime}" \
  --output table
```

### Imposição de Tags de Alocação de Custos

```bash
# Regra do AWS Config: recursos não-conformes são sinalizados
aws configservice put-config-rule \
  --config-rule '{
    "ConfigRuleName": "required-tags-ec2",
    "Source": {
      "Owner": "AWS",
      "SourceIdentifier": "REQUIRED_TAGS"
    },
    "Scope": {
      "ComplianceResourceTypes": ["AWS::EC2::Instance", "AWS::RDS::DBInstance",
                                   "AWS::ECS::Service", "AWS::Lambda::Function"]
    },
    "InputParameters": "{\"tag1Key\":\"Project\",\"tag2Key\":\"Environment\",\"tag3Key\":\"Team\"}"
  }'

# Enviar resultados do Config para SNS para notificação
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:config-alerts"
  }'
```

---

## Anti-Padrões a Evitar

### IPs Elásticos Não Anexados

```bash
# Cada EIP não anexado custa $3,65/mês (a AWS cobra por EIPs não usados)
# Encontre e libere-os
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].AllocationId' \
  --output text | while read alloc; do
    echo "Liberando $alloc"
    aws ec2 release-address --allocation-id "$alloc"
done
```

### NAT Gateway em Ambiente de Dev

```bash
# Ambientes de dev não precisam de NAT Gateway ($32/mês)
# Opções:
# 1. Use subnets públicas para dev (menos seguro, mas aceitável para não-produção)
# 2. Use uma NAT Instance (EC2 t3.nano a $3,50/mês) em vez de NAT Gateway
# 3. Use VPC endpoints para S3/DynamoDB e aceite sem outro egress

# Configuração de NAT Instance (não recomendado para prod, mas ok para dev)
aws ec2 run-instances \
  --image-id ami-00a9d4a05375b2763 \    # Amazon NAT AMI
  --instance-type t3.nano \
  --subnet-id subnet-public-abc \
  --security-group-ids sg-nat \
  --source-dest-check false             # necessário para o NAT funcionar
```

### Manter Versões Antigas do Lambda

```bash
# Lambda armazena todas as versões deployadas (cada uma é cobrada pelo storage)
# Limpe versões antigas

aws lambda list-versions-by-function \
  --function-name myapp-processor \
  --query 'Versions[?Version!=`$LATEST`].Version' \
  --output text | while read version; do
    if [ "$version" -lt "$(( $(aws lambda list-versions-by-function \
      --function-name myapp-processor \
      --query 'max_by(Versions, &LastModified).Version' \
      --output text) - 5 ))" ]; then
      aws lambda delete-function \
        --function-name myapp-processor \
        --qualifier "$version"
    fi
done
```

### Logs do CloudWatch Sem Expiração

Cada função Lambda, task ECS e instance EC2 cria log groups no CloudWatch. Por padrão eles nunca expiram e custam $0,50/GB armazenado.

```bash
# Definir retenção em todos os log groups (30 dias geralmente é suficiente)
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].logGroupName' \
  --output text | tr '\t' '\n' | while read lg; do
    echo "Definindo retenção de 30 dias em $lg"
    aws logs put-retention-policy \
      --log-group-name "$lg" \
      --retention-in-days 30
done
```

---

## Debugging e Solução de Problemas

### Investigação de Picos na Conta

```bash
# 1. Encontrar o serviço causando o pico
aws ce get-cost-and-usage \
  --time-period Start=2024-01-15,End=2024-01-22 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[].{Date:TimePeriod.Start, Groups:Groups[*].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}}' \
  --output json | jq '.[] | .Date as $date | .Groups[] | {date: $date, service: .Service, cost: (.Cost | tonumber)} | select(.cost > 5)'

# 2. Aprofundar no serviço específico
aws ce get-cost-and-usage \
  --time-period Start=2024-01-15,End=2024-01-22 \
  --granularity DAILY \
  --metrics UnblendedCost UsageQuantity \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["AWS Lambda"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# Causas comuns:
# Lambda: função presa em loop de retry infinito → verifique DLQ do SQS e métricas de erro do Lambda
# EC2: Auto Scaling lançou muitas instances e não escalou de volta → verifique histórico de atividade do ASG
# Transferência de dados: adicionou chamada de API externa retornando payloads grandes
# Requisições S3: bug gravando no S3 em loop
# IOPS RDS: migração ou carga de dados causou pico de IOPS
```

### Entendendo a Precificação do Fargate

```bash
# Precificação do Fargate:
# vCPU: $0,04048 por vCPU-hora
# Memória: $0,004445 por GB-hora

# Calcular custo mensal para uma task definition
VCPU=0.5
MEMORY_GB=1
HOURS_PER_MONTH=720

VCPU_COST=$(echo "$VCPU * 0.04048 * $HOURS_PER_MONTH" | bc)
MEMORY_COST=$(echo "$MEMORY_GB * 0.004445 * $HOURS_PER_MONTH" | bc)
TOTAL=$(echo "$VCPU_COST + $MEMORY_COST" | bc)

echo "Custo de vCPU: \$$VCPU_COST/mês"
echo "Custo de Memória: \$$MEMORY_COST/mês"
echo "Total por task: \$$TOTAL/mês"

# Para 2 tasks (mínimo para HA): ~$20/mês
# Para 10 tasks (sob carga): ~$100/mês
# O custo escala linearmente com o número de tasks
```

---

## Cenários do Mundo Real

### Cenário 1: Conta de $800/mês → $480/mês em 4 Semanas

```
Detalhamento inicial:
  ECS Fargate (4 tasks, 1 vCPU, 2GB cada, 24/7): $232/mês
  RDS db.m5.large PostgreSQL Multi-AZ:            $280/mês
  NAT Gateway (2 AZs) + processamento de dados:   $120/mês
  CloudWatch Logs (sem retenção):                  $45/mês
  S3 (sem lifecycle rules):                        $78/mês
  ElastiCache cache.t3.small:                      $28/mês
  Misc:                                            $17/mês
  Total:                                          $800/mês

Ações tomadas (4 semanas):
  Semana 1: Right-sized tasks ECS para 0,5 vCPU, 1GB
            → Fargate: $232 → $116/mês
  Semana 1: Definida retenção de 30 dias em logs do CloudWatch
            → Logs: $45 → $8/mês
  Semana 2: Adicionados VPC endpoints para S3, ECR, Secrets Manager
            → NAT: $120 → $68/mês (tráfego de pull do ECR reduzido)
  Semana 2: Definidas lifecycle rules S3 (Intelligent-Tiering)
            → S3: $78 → $45/mês
  Semana 3: RDS redimensionado para db.t3.medium (p99 da CPU era 15%)
            → RDS: $280 → $124/mês
  Semana 4: Comprados Compute Savings Plans de 1 ano a 70% do novo gasto
            → Desconto Fargate: ~$30/mês

Novo total: $116 + $8 + $68 + $45 + $124 + $28 + $17 - $30 = $376/mês
Economia: $424/mês (redução de 53%)
```

### Cenário 2: Modelo de Custo para um SaaS em Crescimento

```
Tráfego: 10.000 usuários, 50 requisições/usuário/dia = 500.000 requisições/dia = 15M/mês
API: 2× tasks Fargate (0,5 vCPU, 1GB), autoscaling 2-8 tasks
Database: RDS db.t3.medium PostgreSQL Multi-AZ
Cache: ElastiCache cache.t3.micro Redis
Storage: S3 (~100GB de arquivos enviados, CDN via CloudFront)

Compute:
  Fargate média 3 tasks × $14,54/task/mês = $43,62/mês
  Savings Plans 1 ano (40% de desconto): $26,17/mês

Database:
  RDS db.t3.medium Multi-AZ: $124/mês
  Reserved Instance 1 ano (42% de desconto): $72/mês

Cache:
  ElastiCache cache.t3.micro: $12,24/mês

Storage + CDN:
  S3: 100GB × $0,023 = $2,30/mês
  CloudFront: 15M req × $0,0075/10k + 1TB × $0,0085 = $11,25 + $8,70 = $19,95/mês

Rede:
  NAT Gateway (2 AZs): $32/mês

Misc:
  Route 53: $1/mês
  Secrets Manager: $1/mês
  CloudWatch: $5/mês

Total: $26,17 + $72 + $12,24 + $19,95 + $32 + $1 + $1 + $5 + $2,30 = $171,66/mês

Custo unitário: $171,66 / 10.000 usuários = $0,017/usuário/mês
Se usuários pagam $10/mês: COGS = 0,17% da receita → muito saudável
```

---

## Leitura Adicional

- [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/)
- [AWS Compute Optimizer](https://aws.amazon.com/compute-optimizer/)
- [AWS Savings Plans](https://aws.amazon.com/savingsplans/)
- [AWS Trusted Advisor](https://aws.amazon.com/premiumsupport/technology/trusted-advisor/)
- [AWS Pricing Calculator](https://calculator.aws/pricing/2/home)
- [Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) — encontre a memória ideal para Lambda
- [infracost](https://www.infracost.io/) — estime custos de mudanças no Terraform antes de aplicar

---

## Resumo

| Ação | Esforço | Economia Potencial | Prioridade |
|------|---------|-------------------|------------|
| Configurar alertas de billing + orçamento | 30 min | Previne surpresas | Fazer hoje |
| Taguear todos os recursos | 1-2 dias | Viabiliza atribuição | Fazer essa semana |
| Habilitar Cost Anomaly Detection | 15 min | Detecção precoce de picos | Fazer hoje |
| Right-size tasks EC2/ECS | 2-4 horas | 20-50% em compute | Alta |
| Comprar Compute Savings Plans | 1 hora | 27-40% em EC2/Fargate/Lambda | Alta (após 60 dias) |
| Comprar RDS Reserved Instances | 1 hora | 42% em RDS | Alta (após 60 dias) |
| Lifecycle rules S3 + Intelligent-Tiering | 1 hora | 30-70% em storage | Média |
| Definir retenção de logs CloudWatch | 30 min | 80%+ em logging | Ganho fácil |
| VPC endpoints para S3 + DynamoDB | 1 hora | Elimina custos de NAT para esses serviços | Média |
| Limpar EIPs não anexados | 15 min | $3,65/EIP/mês | Ganho fácil |
| Remover log groups antigos do CloudWatch | 30 min | Variável | Ganho fácil |
| Right-sizing de memória do Lambda | 1-2 horas | 20-40% em Lambda | Média |

Otimização de custos na AWS é uma prática contínua, não um projeto único. Configure alertas de billing no primeiro dia, revise custos mensalmente, faça right-sizing após 60 dias de dados de produção e compre capacidade reservada após 90 dias de uso estável.
