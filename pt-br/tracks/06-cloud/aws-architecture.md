# Arquitetura AWS

## Visão Geral

Conhecer os serviços AWS individualmente não é suficiente. A habilidade real é compô-los em arquiteturas confiáveis, seguras, escaláveis e com boa relação custo-benefício. A AWS publicou o **Well-Architected Framework** — um conjunto de princípios de design e boas práticas organizados em seis pilares — para guiar exatamente isso.

Este capítulo aborda padrões arquiteturais para casos de uso comuns: a aplicação web três camadas, API serverless, processamento orientado a eventos e resiliência multi-region. Cada padrão está fundamentado nos princípios do Well-Architected Framework e mostra como os serviços de `aws-core-services.md` se encaixam.

---

## Pré-requisitos

- Serviços fundamentais da AWS (`aws-core-services.md`)
- Conceitos de cloud (`cloud-concepts.md`)
- Conhecimento básico de redes (VPC, subnets, roteamento)

---

## Conceitos Fundamentais

### O Well-Architected Framework

Seis pilares que guiam as decisões de arquitetura na AWS:

| Pilar | Pergunta Central |
|-------|------------------|
| **Operational Excellence** | Como operamos e melhoramos o sistema? |
| **Security** | Como protegemos informações e sistemas? |
| **Reliability** | Como nos recuperamos de falhas? |
| **Performance Efficiency** | Como usamos os recursos com eficiência? |
| **Cost Optimization** | Como evitamos custos desnecessários? |
| **Sustainability** | Como minimizamos o impacto ambiental? |

### Princípios de Design para Confiabilidade

Sistemas confiáveis são construídos sobre estas bases:

**1. Deploy multi-AZ:** Faça deploy em pelo menos duas Availability Zones. Falhas de AZ são raras, mas acontecem.

**2. Auto Scaling:** Substitua automaticamente instances não saudáveis e escale conforme a demanda.

**3. Health checks em todo lugar:** Health checks do ALB, do ECS e failover automático do RDS Multi-AZ.

**4. Circuit breakers:** Pare de chamar uma dependência com falha para deixá-la se recuperar.

**5. Degradação graciosa:** Retorne resultados em cache ou parciais em vez de falhar completamente.

```
Hierarquia de domínios de falha:
Instance → AZ → Region → Provedor

Projete para falhas de AZ (comuns).
Considere falhas de Region (raras) para sistemas críticos.
Multi-provedor raramente vale a complexidade.
```

### Princípios de Arquitetura de Segurança

**Defesa em profundidade:**
```
Internet
    ↓
WAF (bloqueia requisições maliciosas)
    ↓
CloudFront (mitigação de DDoS)
    ↓
Security Group (apenas portas 80/443)
    ↓
ALB (HTTPS termination)
    ↓
Subnet privada (sem IP público nos servidores de app)
Security Group (apenas do ALB)
    ↓
App (ECS Fargate, sem root)
    ↓
Subnet privada (camada de banco de dados)
Security Group (apenas do security group da app)
    ↓
RDS (criptografado, sem acesso público)
```

Nenhuma camada é suficiente sozinha. Cada uma adiciona fricção para um atacante.

---

## Padrões de Arquitetura

### Padrão 1: Aplicação Web Três Camadas

A arquitetura de produção web canônica para a maioria das aplicações.

```
                    ┌─────────────┐
                    │   Route 53  │ (DNS)
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │  CloudFront │ (CDN)
                    └──────┬──────┘
                           │
              ┌────────────▼─────────────┐
              │    Application Load      │
              │    Balancer (ALB)        │ ← Subnet pública
              └──────┬──────────┬────────┘
                     │          │
           ┌─────────▼──┐  ┌────▼────────┐
           │ ECS Fargate│  │ ECS Fargate │ ← Subnet privada (2 AZs)
           │ Task (AZ-a)│  │ Task (AZ-b) │
           └─────────┬──┘  └────┬────────┘
                     │          │
              ┌──────▼──────────▼──────┐
              │     RDS PostgreSQL     │ ← Subnet de banco de dados
              │  Primary   │  Standby  │
              │  (AZ-a)    │  (AZ-b)   │
              └────────────────────────┘
```

**Estrutura da VPC:**
```hcl
# Exemplo com Terraform
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "myapp-vpc" }
}

# Subnets públicas (ALB, NAT Gateway)
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
}

# Subnets privadas (tasks ECS)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# Subnets de banco de dados (RDS)
resource "aws_subnet" "database" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 20)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

**Security groups:**
```bash
# Security group do ALB — aceita tráfego da internet
aws ec2 create-security-group --group-name alb-sg --description "ALB" --vpc-id vpc-abc
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 443 --cidr 0.0.0.0/0

# Security group da App — aceita tráfego apenas do ALB
aws ec2 create-security-group --group-name app-sg --description "Servidores de app" --vpc-id vpc-abc
aws ec2 authorize-security-group-ingress --group-id sg-app --protocol tcp --port 3000 --source-group sg-alb

# Security group do DB — aceita tráfego apenas dos servidores de app
aws ec2 create-security-group --group-name db-sg --description "Banco de dados" --vpc-id vpc-abc
aws ec2 authorize-security-group-ingress --group-id sg-db --protocol tcp --port 5432 --source-group sg-app
```

**ECS Service com Auto Scaling:**
```bash
# Registrar alvo de auto scaling
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/myapp-cluster/myapp-api \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Escalar com base na utilização de CPU
aws application-autoscaling put-scaling-policy \
  --service-namespace ecs \
  --resource-id service/myapp-cluster/myapp-api \
  --scalable-dimension ecs:service:DesiredCount \
  --policy-name cpu-scaling \
  --policy-type TargetTrackingScaling \
  --target-tracking-scaling-policy-configuration '{
    "TargetValue": 70.0,
    "PredefinedMetricSpecification": {
      "PredefinedMetricType": "ECSServiceAverageCPUUtilization"
    },
    "ScaleInCooldown": 300,
    "ScaleOutCooldown": 60
  }'
```

### Padrão 2: API Serverless

Para APIs com tráfego imprevisível ou carga sustentada baixa:

```
Route 53 → CloudFront → API Gateway → Lambda → RDS Proxy → RDS

Por que RDS Proxy?
  Lambda escala de 0 a 1000+ execuções concorrentes instantaneamente.
  Cada Lambda abre uma conexão com o DB.
  Sem RDS Proxy: 1000 conexões sobrecarregam o Postgres → OOM.
  Com RDS Proxy: conexões são pooladas → Postgres vê 10-20 conexões.
```

Exemplo com AWS CDK (TypeScript):
```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as nodejs from 'aws-cdk-lib/aws-lambda-nodejs';

export class ServerlessApiStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Função Lambda
    const apiFunction = new nodejs.NodejsFunction(this, 'ApiFunction', {
      entry: 'src/lambda/api.ts',
      handler: 'handler',
      runtime: lambda.Runtime.NODEJS_22_X,
      timeout: cdk.Duration.seconds(30),
      memorySize: 512,
      environment: {
        NODE_ENV: 'production',
        DB_PROXY_ENDPOINT: rdsProxy.endpoint,
      },
    });

    // API Gateway
    const api = new apigateway.RestApi(this, 'Api', {
      restApiName: 'myapp-api',
      defaultCorsPreflightOptions: {
        allowOrigins: apigateway.Cors.ALL_ORIGINS,
        allowMethods: apigateway.Cors.ALL_METHODS,
      },
    });

    // Conectar rotas
    const users = api.root.addResource('users');
    users.addMethod('GET', new apigateway.LambdaIntegration(apiFunction));
    users.addMethod('POST', new apigateway.LambdaIntegration(apiFunction));

    const user = users.addResource('{id}');
    user.addMethod('GET', new apigateway.LambdaIntegration(apiFunction));
    user.addMethod('PUT', new apigateway.LambdaIntegration(apiFunction));
    user.addMethod('DELETE', new apigateway.LambdaIntegration(apiFunction));
  }
}
```

### Padrão 3: Processamento Orientado a Eventos

Para cargas assíncronas: processamento de imagens, envio de e-mails, geração de relatórios:

```
S3 (upload) ──→ S3 Event Notification ──→ Fila SQS ──→ Lambda
                                                              ↓
                                               Processa imagem, armazena resultado
                                                              ↓
                                                      SNS → e-mail/notificação push
```

```typescript
// Lambda disparado pelo SQS
import { SQSHandler, SQSEvent } from 'aws-lambda';
import { S3Client, GetObjectCommand } from '@aws-sdk/client-s3';
import { SNSClient, PublishCommand } from '@aws-sdk/client-sns';

const s3 = new S3Client({ region: 'us-east-1' });
const sns = new SNSClient({ region: 'us-east-1' });

export const handler: SQSHandler = async (event: SQSEvent) => {
  for (const record of event.Records) {
    const message = JSON.parse(record.body);
    const { bucket, key, userId } = message;

    try {
      // Baixar arquivo do S3
      const s3Response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
      const buffer = await streamToBuffer(s3Response.Body as NodeJS.ReadableStream);

      // Processar o arquivo (ex.: redimensionar imagem)
      const processed = await processImage(buffer);

      // Armazenar resultado
      await s3.send(new PutObjectCommand({
        Bucket: process.env.OUTPUT_BUCKET!,
        Key: `processed/${key}`,
        Body: processed,
      }));

      // Notificar usuário
      await sns.send(new PublishCommand({
        TopicArn: process.env.NOTIFICATION_TOPIC_ARN!,
        Message: JSON.stringify({ userId, status: 'complete', outputKey: `processed/${key}` }),
      }));
    } catch (err) {
      console.error({ err, messageId: record.messageId }, 'Falha ao processar mensagem');
      throw err;   // SQS fará retry (até maxReceiveCount, depois envia para DLQ)
    }
  }
};
```

Boas práticas com SQS:
```bash
# Criar fila SQS com Dead Letter Queue
aws sqs create-queue --queue-name myapp-image-processing-dlq

aws sqs create-queue \
  --queue-name myapp-image-processing \
  --attributes '{
    "VisibilityTimeout": "300",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123:myapp-image-processing-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

### Padrão 4: Site Estático + API (Jamstack)

```
CloudFront ──→ S3 (exportação estática React/Next.js)
     └──────→ ALB / API Gateway (API Node.js)

Benefícios:
- Assets estáticos servidos do edge (global, rápido, barato)
- API escala independentemente
- S3 + CloudFront = escala praticamente ilimitada para o frontend
- Sem servidor para gerenciar no frontend
```

```bash
# Build e deploy do frontend
npm run build

# Sincronizar para S3
aws s3 sync ./out s3://myapp-static/ --delete

# Invalidar cache do CloudFront
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"
```

Configuração do CloudFront para SPA com API:
```json
{
  "Origins": [
    {
      "Id": "static",
      "DomainName": "myapp-static.s3.amazonaws.com"
    },
    {
      "Id": "api",
      "DomainName": "api.myapp.com",
      "CustomOriginConfig": {
        "OriginProtocolPolicy": "https-only"
      }
    }
  ],
  "CacheBehaviors": [
    {
      "PathPattern": "/api/*",
      "TargetOriginId": "api",
      "CachePolicyId": "4135ea2d-6df8-44a3-9df3-4b5a84be39ad",
      "ViewerProtocolPolicy": "https-only"
    }
  ],
  "DefaultCacheBehavior": {
    "TargetOriginId": "static",
    "ViewerProtocolPolicy": "redirect-to-https"
  }
}
```

---

## Padrões Operacionais

### Deploy Blue-Green

Execute dois ambientes idênticos e alterne o tráfego entre eles:

```bash
# Criar target group green (igual ao blue, aponta para nova task definition)
aws elbv2 create-target-group --name myapp-green-tg ...

# Fazer deploy da nova versão no service ECS green
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-green \
  --task-definition myapp:new

# Aguardar o green ficar saudável
aws ecs wait services-stable --cluster myapp-cluster --services myapp-green

# Alternar listener do ALB para green
aws elbv2 modify-listener \
  --listener-arn arn:... \
  --default-actions Type=forward,TargetGroupArn=arn:.../myapp-green-tg

# Se algo der errado:
aws elbv2 modify-listener \
  --listener-arn arn:... \
  --default-actions Type=forward,TargetGroupArn=arn:.../myapp-blue-tg
```

### Pipeline CI/CD com CodePipeline

```yaml
# buildspec.yml — configuração do CodeBuild
version: 0.2

phases:
  install:
    runtime-versions:
      nodejs: 22
    commands:
      - npm ci

  pre_build:
    commands:
      - npm test
      - npm run lint
      - aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_REGISTRY

  build:
    commands:
      - docker build -t $IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION .
      - docker tag $IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION $ECR_REGISTRY/$IMAGE_NAME:latest

  post_build:
    commands:
      - docker push $ECR_REGISTRY/$IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION
      - docker push $ECR_REGISTRY/$IMAGE_NAME:latest
      - |
        printf '[{"name":"api","imageUri":"%s"}]' \
          "$ECR_REGISTRY/$IMAGE_NAME:$CODEBUILD_RESOLVED_SOURCE_VERSION" \
          > imagedefinitions.json

artifacts:
  files:
    - imagedefinitions.json
```

### Planejamento de Recuperação de Desastres

RTO (Recovery Time Objective) vs RPO (Recovery Point Objective):

```
RPO = Quanto de dados você pode perder?
      → Configure a frequência de backup menor que o RPO
      → RDS: backups automatizados a cada 5 minutos (PITR)

RTO = Por quanto tempo o sistema pode ficar fora?
      → RDS Multi-AZ: ~60s de failover
      → Multi-region: 5-30 minutos (se pré-aquecido)
      → Region única a partir de backup: horas
```

Estratégias de DR por nível:

| Estratégia | RTO | RPO | Custo | Complexidade |
|------------|-----|-----|-------|--------------|
| Backup & restore | Horas | Horas | $ | Baixa |
| Pilot light | 10-30 min | Minutos | $$ | Média |
| Warm standby | 5-10 min | Segundos | $$$ | Alta |
| Active-Active | Quase zero | Quase zero | $$$$ | Muito alta |

---

## Padrões de Otimização de Custos

### Right-Sizing

```bash
# AWS Compute Optimizer dá recomendações baseadas no uso real
aws compute-optimizer get-ec2-instance-recommendations \
  --account-ids 123456789012 \
  --query 'instanceRecommendations[].{
    Instance: instanceArn,
    Finding: finding,
    RecommendedType: recommendationOptions[0].instanceType,
    MonthlySavings: recommendationOptions[0].estimatedMonthlySavings.value
  }'
```

### Reserved Instances / Savings Plans

Após executar uma aplicação por 2-3 meses com uso estável, adquira capacidade reservada:

```
On-demand:    $0,0832/hr (t3.medium)
1yr Reserved: $0,0520/hr (economia de 37%)
3yr Reserved: $0,0336/hr (economia de 60%)

Para 2 instances t3.medium rodando 24/7:
On-demand:   2 × $0,0832 × 8760h = $1.456/ano
3yr Reserved: 2 × $0,0336 × 8760h = $589/ano
Economia: $867/ano
```

### S3 Intelligent-Tiering

```bash
# Mover objetos automaticamente para classes de storage mais baratas com base nos padrões de acesso
aws s3api put-bucket-intelligent-tiering-configuration \
  --bucket myapp-uploads \
  --id "intelligent-tiering" \
  --intelligent-tiering-configuration '{
    "Id": "intelligent-tiering",
    "Status": "Enabled",
    "Tierings": [
      {"Days": 90, "AccessTier": "ARCHIVE_ACCESS"},
      {"Days": 180, "AccessTier": "DEEP_ARCHIVE_ACCESS"}
    ]
  }'
```

---

## Anti-Padrões a Evitar

### Sem Estratégia de Tags

Sem tags, você não consegue filtrar custos por time, ambiente ou projeto:

```bash
# Tague tudo desde o início
aws ec2 create-tags --resources i-abc123 --tags \
  Key=Project,Value=myapp \
  Key=Environment,Value=production \
  Key=Team,Value=backend \
  Key=CostCenter,Value=engineering
```

### Region Única, AZ Única

Um deploy em AZ única é equivalente a um servidor único — uma falha de hardware encerra seu serviço. Sempre faça deploy em pelo menos duas AZs para produção.

### Não Configurar Alertas de Billing

```bash
aws budgets create-budget \
  --account-id 123456789012 \
  --budget '{
    "BudgetName": "monthly-limit",
    "BudgetType": "COST",
    "TimeUnit": "MONTHLY",
    "BudgetLimit": {"Amount": "200", "Unit": "USD"}
  }' \
  --notifications-with-subscribers '[{
    "Notification": {
      "NotificationType": "ACTUAL",
      "ComparisonOperator": "GREATER_THAN",
      "Threshold": 80
    },
    "Subscribers": [{"SubscriptionType": "EMAIL", "Address": "admin@myapp.com"}]
  }]'
```

Configure isso no primeiro dia. Um Lambda mal configurado ou uma instance EC2 esquecida pode gerar centenas de dólares em custos.

### Usar Subnets Públicas para Bancos de Dados

Bancos de dados nunca devem ficar em subnets públicas, mesmo que os security groups sejam restritivos. Defesa em profundidade — sem IP público significa sem exposição.

### Configuração Manual pelo Console

```
Clicou pelo console para criar recursos → não dá para reproduzir → não dá para revisar mudanças
→ Use CloudFormation, CDK ou Terraform desde o início
```

---

## Debugging e Solução de Problemas

### Problemas Comuns de Arquitetura

**Task ECS ficando reiniciando:**
```bash
# Verificar motivo da task parada
aws ecs describe-tasks \
  --cluster myapp-cluster \
  --tasks TASK_ARN \
  --query 'tasks[].stoppedReason'

# Verificar logs do CloudWatch
aws logs get-log-events \
  --log-group-name /ecs/myapp \
  --log-stream-name ecs/api/TASK_ID \
  --limit 50
```

**ALB 502 Bad Gateway:**
```bash
# Verificar saúde do target group
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...

# Causas comuns:
# - Health check path retorna não-2xx
# - Container não iniciado na porta correta
# - Security group bloqueia comunicação ALB → ECS
```

**Lambda com timeout:**
```bash
# Verificar métricas do CloudWatch
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=myapp-api \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics p95 p99
```

**Conexões RDS esgotadas:**
```bash
# Verificar conexões ativas
aws rds describe-db-log-files \
  --db-instance-identifier myapp-prod

# Use RDS Proxy para pooling de conexões em cargas Lambda
# Ou aumente max_connections no parameter group para cargas ECS
```

---

## Cenários do Mundo Real

### Cenário 1: Arquitetura para um MVP SaaS

```
Orçamento: $100-200/mês
Usuários: 0-1000 (crescimento desconhecido)

Arquitetura:
  DNS:        Route 53 ($0,50/zone)
  CDN:        CloudFront (free tier para o primeiro 1TB/mês)
  Frontend:   S3 static hosting ($0,023/GB)
  API:        1× ECS Fargate task (equivalente a t, ~$15/mês)
  Database:   RDS db.t3.micro PostgreSQL ($13/mês)
  Cache:      ElastiCache cache.t3.micro ($13/mês) [opcional no MVP]
  Secrets:    Secrets Manager ($0,40/secret)
  Monit.:     CloudWatch (free tier, depois ~$5/mês)
  Total:      ~$50/mês

Quando escalar:
  - API: Adicione segunda task + ALB quando uptime importar (~+$20/mês)
  - DB: Mude para db.t3.small quando CPU estiver consistentemente >70%
  - Cache: Adicione quando CPU do DB subir ou latência P95 > 200ms
```

### Cenário 2: Preparando para um Pico de Tráfego

```bash
# Pré-aquecer o service ECS antes de um pico conhecido
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --desired-count 10

# Solicitar aumento temporário de concorrência Lambda ao suporte da AWS
# (se usando serverless)

# Habilitar cache do CloudFront para respostas de API que podem ser cacheadas
# Cache-Control: public, max-age=60 para endpoints de listagem

# Após o pico:
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --desired-count 2
```

### Cenário 3: Configurando Infrastructure as Code

```bash
# Instalar AWS CDK
npm install -g aws-cdk

# Bootstrap do CDK na sua conta
cdk bootstrap aws://ACCOUNT_ID/us-east-1

# Criar novo app CDK
mkdir myapp-infra && cd myapp-infra
cdk init app --language typescript

# Fazer deploy
cdk deploy --all

# Ver o que vai mudar antes de fazer deploy
cdk diff
```

---

## Leitura Adicional

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Solutions Library](https://aws.amazon.com/solutions/) — arquiteturas de referência validadas
- [CDK Patterns](https://cdkpatterns.com/) — padrões de arquitetura CDK open-source
- [The Cloud Resume Challenge](https://cloudresumechallenge.dev/) — aprendizado prático de AWS
- [AWS Pricing Calculator](https://calculator.aws/)

---

## Resumo

| Padrão | Quando Usar |
|--------|-------------|
| Aplicação web três camadas | Maioria das aplicações de produção com estado persistente |
| API serverless | Tráfego imprevisível, orientada a eventos, carga sustentada baixa |
| Processamento orientado a eventos | Cargas assíncronas: processamento de imagens, e-mail, relatórios |
| Jamstack | Sites com muito conteúdo, SPAs, frontends que podem ser exportados estaticamente |
| Deploy blue-green | Deploys sem downtime com rollback instantâneo |
| Multi-AZ | Sempre para bancos de dados e serviços de produção |
| RDS Proxy | Qualquer função Lambda que conecta ao RDS |
| Infrastructure as Code | Sempre — use CDK, Terraform ou CloudFormation |
| Alertas de billing | No primeiro dia — sempre |
| Tagging de recursos | No primeiro dia — viabiliza atribuição e filtragem de custos |

Arquitetura é a arte de fazer as trocas certas. Comece simples (region única, duas AZs, ECS Fargate, RDS Multi-AZ), meça e adicione complexidade apenas quando tiver evidências de que é necessário. O Well-Architected Framework é uma lente, não um checklist — use-o para fazer perguntas melhores, não para implementar cegamente cada recomendação.
