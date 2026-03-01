# Serviços Fundamentais da AWS

## Visão Geral

A AWS (Amazon Web Services) é a maior plataforma de cloud do mundo, com mais de 200 serviços. O objetivo não é conhecer todos eles — é dominar os 20 serviços que cobrem 90% dos casos de uso reais.

Este capítulo abrange os serviços AWS que todo desenvolvedor backend precisa conhecer: compute (EC2, ECS, Lambda), storage (S3, EBS, EFS), bancos de dados (RDS, DynamoDB, ElastiCache), redes (VPC, Route 53, CloudFront, ALB) e operações (IAM, CloudWatch, SSM).

O foco é no uso prático: quando escolher cada serviço, como configurá-lo corretamente e as armadilhas que custam tempo e dinheiro às equipes.

---

## Pré-requisitos

- Conta AWS (o free tier cobre a maioria dos exemplos)
- AWS CLI instalado e configurado
- Conceitos básicos de cloud (veja `cloud-concepts.md`)

```bash
# Instalar AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configurar
aws configure
# AWS Access Key ID: ...
# AWS Secret Access Key: ...
# Default region name: us-east-1
# Default output format: json

# Verificar
aws sts get-caller-identity
```

---

## Conceitos Fundamentais

### Categorias de Serviços

```
Compute:        EC2, ECS, EKS, Lambda, Fargate, Lightsail
Storage:        S3, EBS, EFS, Glacier, Storage Gateway
Databases:      RDS, Aurora, DynamoDB, ElastiCache, Redshift
Networking:     VPC, Route 53, CloudFront, ALB, NLB, API Gateway
Security:       IAM, KMS, Secrets Manager, WAF, Shield, GuardDuty
Operations:     CloudWatch, CloudTrail, SSM, Config, Trusted Advisor
Messaging:      SQS, SNS, EventBridge, Kinesis
Developer:      CodePipeline, CodeBuild, CodeDeploy, ECR
```

---

## Serviços de Compute

### EC2 (Elastic Compute Cloud)

Máquinas virtuais. O bloco fundamental do compute na AWS.

**Nomenclatura de instances:** `t3.medium`
- Família: `t` = burstable, `m` = uso geral, `c` = otimizado para compute, `r` = otimizado para memória
- Geração: `3` (número maior = mais recente, melhor custo-benefício)
- Tamanho: `nano` < `micro` < `small` < `medium` < `large` < `xlarge` < `2xlarge`

```bash
# Lançar uma instance
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \   # Amazon Linux 2023
  --instance-type t3.small \
  --key-name my-key-pair \
  --security-group-ids sg-abc12345 \
  --subnet-id subnet-abc12345 \
  --iam-instance-profile Name=MyInstanceProfile \
  --user-data file://bootstrap.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=myapp}]'

# Listar instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=myapp" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,IP:PublicIpAddress}'
```

`bootstrap.sh` (User Data — executa na primeira inicialização):
```bash
#!/bin/bash
yum update -y
curl -fsSL https://get.docker.com | sh
usermod -aG docker ec2-user
systemctl enable --now docker

# Instalar sua aplicação
docker pull ghcr.io/myorg/myapp:latest
docker run -d --restart=unless-stopped -p 3000:3000 ghcr.io/myorg/myapp:latest
```

**Quando usar EC2:**
- Precisa de controle total sobre o SO
- Executa cargas de trabalho de longa duração
- Tipos específicos de instance (GPU, alta memória)
- Quando ECS/Lambda não se encaixam no caso de uso

### ECS (Elastic Container Service)

Execute containers Docker na AWS. Dois tipos de launch:

**EC2 launch type:** Containers rodam em instances EC2 que você gerencia.
**Fargate launch type:** Containers serverless — a AWS gerencia a infraestrutura subjacente. Você paga por task, não por instance EC2.

Conceitos principais:
```
Cluster → agrupamento lógico de capacidade de compute
Task Definition → blueprint (como um docker-compose.yml para um serviço)
Task → instance em execução de uma Task Definition
Service → mantém N tasks rodando, gerencia rolling deploys
```

```bash
# Registrar uma task definition
aws ecs register-task-definition --cli-input-json '{
  "family": "myapp",
  "requiresCompatibilities": ["FARGATE"],
  "networkMode": "awsvpc",
  "cpu": "512",
  "memory": "1024",
  "executionRoleArn": "arn:aws:iam::123456789012:role/ecsTaskExecutionRole",
  "containerDefinitions": [{
    "name": "api",
    "image": "ghcr.io/myorg/myapp:latest",
    "portMappings": [{"containerPort": 3000}],
    "environment": [
      {"name": "NODE_ENV", "value": "production"}
    ],
    "secrets": [
      {"name": "DATABASE_URL", "valueFrom": "arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/database-url"}
    ],
    "logConfiguration": {
      "logDriver": "awslogs",
      "options": {
        "awslogs-group": "/ecs/myapp",
        "awslogs-region": "us-east-1",
        "awslogs-stream-prefix": "api"
      }
    },
    "healthCheck": {
      "command": ["CMD-SHELL", "curl -f http://localhost:3000/health || exit 1"],
      "interval": 30,
      "timeout": 5,
      "retries": 3,
      "startPeriod": 15
    }
  }]
}'

# Criar um service
aws ecs create-service \
  --cluster myapp-cluster \
  --service-name myapp-api \
  --task-definition myapp:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-abc],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=api,containerPort=3000"
```

**Quando usar Fargate (vs EC2 launch type):**
- Para a maioria das aplicações — mais simples, sem gerenciamento de EC2
- Padrões de tráfego irregulares (paga por task)
- Requisitos de isolamento de segurança

### Lambda (Funções Serverless)

Execute código sem servidores. Cobrado a cada 100ms de tempo de execução e por número de invocações.

```typescript
// Handler de função Lambda (Node.js/TypeScript)
import { Handler, APIGatewayProxyEvent, APIGatewayProxyResult } from 'aws-lambda';

export const handler: Handler<APIGatewayProxyEvent, APIGatewayProxyResult> = async (event) => {
  const { httpMethod, path, body } = event;

  try {
    if (httpMethod === 'GET' && path === '/users') {
      const users = await getUsers();
      return {
        statusCode: 200,
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(users),
      };
    }

    return { statusCode: 404, body: JSON.stringify({ error: 'Not found' }) };
  } catch (err) {
    console.error(err);
    return { statusCode: 500, body: JSON.stringify({ error: 'Internal error' }) };
  }
};
```

```bash
# Fazer deploy de uma função Lambda
zip function.zip index.js

aws lambda create-function \
  --function-name myapp-users \
  --runtime nodejs22.x \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 256

# Atualizar código
aws lambda update-function-code \
  --function-name myapp-users \
  --zip-file fileb://function.zip

# Invocar diretamente
aws lambda invoke \
  --function-name myapp-users \
  --payload '{"httpMethod":"GET","path":"/users"}' \
  response.json && cat response.json
```

**Limites e armadilhas do Lambda:**
- Tempo máximo de execução: 15 minutos
- Memória máxima: 10 GB
- Cold start: 100ms-1s (Java é pior, Node.js/Python são melhores)
- Execuções concorrentes por conta: 1000 (soft limit)
- Tamanho do pacote: 50 MB comprimido, 250 MB descomprimido

**Quando usar Lambda:**
- Processamento orientado a eventos (uploads no S3, mensagens no SQS, streams do DynamoDB)
- APIs com tráfego imprevisível ou baixo
- Cron jobs / tarefas agendadas
- Webhooks (paga por chamada, não por tempo ocioso)

---

## Serviços de Storage

### S3 (Simple Storage Service)

Object storage. Capacidade praticamente ilimitada, 11 noves de durabilidade. O serviço AWS mais usado.

```bash
# Criar um bucket
aws s3 mb s3://myapp-uploads-prod --region us-east-1

# Upload
aws s3 cp ./logo.png s3://myapp-uploads-prod/images/logo.png

# Upload de diretório inteiro
aws s3 sync ./public/ s3://myapp-static-prod/ --delete

# Listar
aws s3 ls s3://myapp-uploads-prod/

# Gerar presigned URL (para arquivos privados)
aws s3 presign s3://myapp-uploads-prod/documents/report.pdf --expires-in 3600

# Definir bucket policy para leitura pública
aws s3api put-bucket-policy --bucket myapp-static-prod --policy '{
  "Version": "2012-10-17",
  "Statement": [{
    "Effect": "Allow",
    "Principal": "*",
    "Action": "s3:GetObject",
    "Resource": "arn:aws:s3:::myapp-static-prod/*"
  }]
}'
```

S3 a partir do Node.js (AWS SDK v3):
```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });

// Fazer upload de um arquivo
async function uploadFile(key: string, body: Buffer, contentType: string): Promise<string> {
  await s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
    Body: body,
    ContentType: contentType,
  }));
  return `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${key}`;
}

// Gerar presigned URL para acesso temporário
async function getPresignedUrl(key: string, expiresIn = 3600): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
  });
  return getSignedUrl(s3, command, { expiresIn });
}
```

**Classes de storage do S3:**
| Classe | Caso de uso | Recuperação | Custo |
|--------|-------------|-------------|-------|
| Standard | Dados acessados frequentemente | Instantâneo | $$$ |
| Standard-IA | Acesso pouco frequente | Instantâneo | $$ |
| One Zone-IA | Acesso infrequente não-crítico | Instantâneo | $ |
| Glacier Instant | Arquivo, acessado trimestralmente | Instantâneo | $ |
| Glacier Flexible | Arquivo, acessado raramente | 1-12 horas | $$ |
| Glacier Deep Archive | Arquivo de longo prazo | 12-48 horas | $ |

Use lifecycle rules para transicionar objetos automaticamente entre classes.

### EBS (Elastic Block Store)

Storage de bloco persistente para instances EC2. Pense como um HD de rede acoplado.

```bash
# Criar um volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --size 50 \
  --throughput 125 \
  --iops 3000

# Anexar a uma instance
aws ec2 attach-volume \
  --volume-id vol-abc123 \
  --instance-id i-abc123 \
  --device /dev/sdf
```

**Tipos de volume:**
| Tipo | Caso de uso | IOPS |
|------|-------------|------|
| gp3 | Uso geral (padrão) | 3000-16000 |
| io2 Block Express | Bancos de dados de alta performance | Até 256.000 |
| sc1 | HDD frio, otimizado para throughput | 250 |
| st1 | HDD otimizado para throughput | 500 |

**Fatos importantes sobre EBS:**
- Volumes EBS são específicos por AZ (não podem ser anexados a instances em AZs diferentes)
- Snapshots replicam entre AZs dentro de uma region
- Volumes EBS persistem independentemente do ciclo de vida da instance EC2

---

## Serviços de Banco de Dados

### RDS (Relational Database Service)

Bancos de dados relacionais gerenciados: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Aurora.

```bash
# Criar uma instance RDS PostgreSQL
aws rds create-db-instance \
  --db-instance-identifier myapp-prod \
  --db-instance-class db.t3.medium \
  --engine postgres \
  --engine-version 16.2 \
  --master-username app \
  --master-user-password "$(openssl rand -base64 32)" \
  --db-name app \
  --allocated-storage 20 \
  --storage-type gp3 \
  --multi-az \                         # deploy standby em outra AZ
  --storage-encrypted \
  --backup-retention-period 7 \        # 7 dias de backups automatizados
  --deletion-protection \              # previne exclusão acidental
  --no-publicly-accessible \           # somente subnet privada
  --vpc-security-group-ids sg-abc123
```

**RDS vs Aurora:**
- Aurora é o engine MySQL/Postgres-compatível da AWS
- Aurora é 3x-5x mais rápido que MySQL padrão
- Aurora Serverless v2: autoscala compute, ótimo para cargas variáveis
- Aurora é ~20% mais caro, mas inclui alta disponibilidade por padrão

**Armadilhas do RDS:**
- `db.t3.micro` e menores são burstable — créditos de CPU podem se esgotar sob carga sustentada
- O standby Multi-AZ não é uma read replica — só ativa em caso de failover
- A recuperação point-in-time retorna no mínimo 5 minutos atrás

### ElastiCache

Cache em memória gerenciado: Redis e Memcached.

```bash
# Criar um cluster Redis
aws elasticache create-replication-group \
  --replication-group-id myapp-cache \
  --description "Cache da aplicação" \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-clusters 2 \   # primary + uma replica
  --security-group-ids sg-abc123 \
  --subnet-group-name myapp-cache-subnet-group \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled
```

ElastiCache a partir do Node.js:
```typescript
import { createClient } from 'redis';

const redis = createClient({
  url: `redis://${process.env.ELASTICACHE_ENDPOINT}:6379`,
  socket: {
    tls: process.env.NODE_ENV === 'production',   // criptografia em trânsito do ElastiCache
  },
});

await redis.connect();

// Padrão cache-aside
async function getCachedUser(userId: string) {
  const cacheKey = `user:${userId}`;
  const cached = await redis.get(cacheKey);

  if (cached) return JSON.parse(cached);

  const user = await db.user.findUnique({ where: { id: userId } });
  if (user) {
    await redis.setEx(cacheKey, 300, JSON.stringify(user));  // TTL: 5 minutos
  }
  return user;
}
```

---

## Serviços de Rede

### VPC (Virtual Private Cloud)

Sua rede privada isolada na AWS. Todos os recursos rodam dentro de uma VPC.

```
VPC: 10.0.0.0/16

  Subnets Públicas (têm rota para o Internet Gateway)
    10.0.1.0/24 (us-east-1a) → Load Balancers, NAT Gateway
    10.0.2.0/24 (us-east-1b) → Load Balancers

  Subnets Privadas (sem acesso direto à internet)
    10.0.10.0/24 (us-east-1a) → ECS tasks, instances EC2
    10.0.11.0/24 (us-east-1b) → ECS tasks, instances EC2

  Subnets de Banco de Dados (ainda mais restritas)
    10.0.20.0/24 (us-east-1a) → RDS, ElastiCache
    10.0.21.0/24 (us-east-1b) → RDS, ElastiCache
```

**Security Groups:** Firewall stateful no nível do recurso.
**Network ACLs (NACLs):** Firewall stateless no nível da subnet.

### Application Load Balancer (ALB)

Load balancer de camada 7 (HTTP/HTTPS). Distribui tráfego, gerencia SSL termination, suporta roteamento baseado em path e host.

```bash
# Criar target group
aws elbv2 create-target-group \
  --name myapp-tg \
  --protocol HTTP \
  --port 3000 \
  --vpc-id vpc-abc123 \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30

# Criar ALB
aws elbv2 create-load-balancer \
  --name myapp-alb \
  --subnets subnet-abc subnet-def \
  --security-groups sg-abc123 \
  --scheme internet-facing

# Adicionar listener HTTPS
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:...:certificate/abc \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...
```

### Route 53

Serviço de DNS da AWS. Gerencia registro de domínios, registros DNS e políticas de roteamento.

```bash
# Criar um record set
aws route53 change-resource-record-sets \
  --hosted-zone-id Z1234567890ABC \
  --change-batch '{
    "Changes": [{
      "Action": "UPSERT",
      "ResourceRecordSet": {
        "Name": "api.myapp.com",
        "Type": "A",
        "AliasTarget": {
          "HostedZoneId": "Z35SXDOTRQ7X7K",
          "DNSName": "myapp-alb-123456.us-east-1.elb.amazonaws.com",
          "EvaluateTargetHealth": true
        }
      }
    }]
  }'
```

### CloudFront (CDN)

Rede de entrega de conteúdo global. Faz cache de conteúdo em edge locations ao redor do mundo, reduzindo drasticamente a latência.

```bash
# Criar uma distribuição CloudFront para site estático no S3
aws cloudfront create-distribution --distribution-config '{
  "Origins": {
    "Quantity": 1,
    "Items": [{
      "Id": "S3-myapp-static",
      "DomainName": "myapp-static.s3.amazonaws.com",
      "S3OriginConfig": {"OriginAccessIdentity": ""}
    }]
  },
  "DefaultCacheBehavior": {
    "TargetOriginId": "S3-myapp-static",
    "ViewerProtocolPolicy": "redirect-to-https",
    "CachePolicyId": "658327ea-f89d-4fab-a63d-7e88639e58f6",
    "Compress": true
  },
  "Enabled": true,
  "HttpVersion": "http2and3",
  "DefaultRootObject": "index.html"
}'
```

---

## Serviços de Segurança

### IAM (Identity and Access Management)

Controle quem pode acessar o quê na AWS. Tudo na AWS é uma chamada IAM.

```bash
# Criar uma role IAM para tasks ECS
aws iam create-role \
  --role-name ecsTaskRole \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Anexar uma policy (exemplo de acesso ao S3)
aws iam put-role-policy \
  --role-name ecsTaskRole \
  --policy-name S3Access \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::myapp-uploads-prod/*"
    }]
  }'
```

### Secrets Manager

Armazene e rotacione secrets (senhas de banco de dados, API keys) com segurança.

```bash
# Armazenar um secret
aws secretsmanager create-secret \
  --name myapp/database-url \
  --secret-string "postgres://app:mysecretpassword@rds-host:5432/app"

# Recuperar um secret
aws secretsmanager get-secret-value \
  --secret-id myapp/database-url \
  --query SecretString \
  --output text
```

A partir do Node.js:
```typescript
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const secretsManager = new SecretsManagerClient({ region: 'us-east-1' });

async function getSecret(secretId: string): Promise<string> {
  const { SecretString } = await secretsManager.send(
    new GetSecretValueCommand({ SecretId: secretId })
  );
  if (!SecretString) throw new Error(`Secret ${secretId} sem valor de string`);
  return SecretString;
}

// Na inicialização da aplicação
const DATABASE_URL = await getSecret('myapp/database-url');
```

### CloudWatch

Serviço de monitoramento, logging e alertas da AWS.

```bash
# Criar um alarme de métrica
aws cloudwatch put-metric-alarm \
  --alarm-name myapp-error-rate-high \
  --alarm-description "Taxa de erros acima de 5%" \
  --namespace AWS/ApplicationELB \
  --metric-name HTTPCode_Target_5XX_Count \
  --dimensions Name=LoadBalancer,Value=app/myapp-alb/abc123 \
  --statistic Sum \
  --period 60 \
  --threshold 10 \
  --comparison-operator GreaterThanThreshold \
  --evaluation-periods 3 \
  --alarm-actions arn:aws:sns:us-east-1:123456789012:myapp-alerts \
  --treat-missing-data notBreaching

# Consultar logs da aplicação
aws logs filter-log-events \
  --log-group-name /ecs/myapp \
  --start-time $(date -d '1 hour ago' +%s000) \
  --filter-pattern '"level":50'   # nível de erro do Pino
```

---

## Anti-Padrões a Evitar

### Credenciais AWS no Código

```typescript
// RUIM — nunca coloque credenciais no código
const s3 = new S3Client({
  credentials: {
    accessKeyId: 'AKIAIOSFODNN7EXAMPLE',
    secretAccessKey: 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
  },
});

// BOM — use roles IAM (EC2/ECS), ou variáveis de ambiente para dev local
// O AWS SDK descobre credenciais automaticamente: role IAM, env vars, ~/.aws/credentials
const s3 = new S3Client({ region: 'us-east-1' });
```

### Usar Credenciais da Conta Root

Crie usuários e roles IAM. Nunca use API keys da conta root para nada.

### Esquecer de Habilitar Versionamento nos Buckets S3

```bash
aws s3api put-bucket-versioning \
  --bucket myapp-critical-data \
  --versioning-configuration Status=Enabled
```

Sem versionamento, deletar ou sobrescrever um objeto é irrecuperável.

### Security Groups Abertos Demais

```bash
# RUIM — 0.0.0.0/0 em todas as portas
aws ec2 authorize-security-group-ingress \
  --group-id sg-abc \
  --protocol -1 \
  --port -1 \
  --cidr 0.0.0.0/0

# BOM — apenas o necessário, de onde é necessário
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp \
  --port 5432 \
  --source-group sg-ecs-tasks   # apenas do security group das tasks ECS
```

---

## Leitura Adicional

- [Documentação da AWS](https://docs.aws.amazon.com/)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS SDK para JavaScript v3](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/)
- [Referência do AWS CLI](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [CloudFormation / CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html)

---

## Resumo

| Serviço | Caso de Uso |
|---------|-------------|
| EC2 | VMs de longa duração, controle total do SO |
| ECS Fargate | Cargas containerizadas, containers serverless |
| Lambda | Funções orientadas a eventos, de curta duração, webhooks |
| S3 | Object storage, sites estáticos, backups, uploads de arquivos |
| EBS | Storage de bloco persistente para EC2 |
| RDS | Bancos de dados relacionais gerenciados (Postgres, MySQL) |
| ElastiCache | Redis/Memcached gerenciado para cache e sessões |
| VPC | Rede privada isolada — base para tudo |
| ALB | Load balancing camada 7, HTTPS termination, roteamento |
| Route 53 | Gerenciamento de DNS, roteamento por saúde |
| CloudFront | CDN — cache de assets estáticos e APIs no edge |
| IAM | Autenticação e autorização para todos os recursos AWS |
| Secrets Manager | Storage seguro de secrets com rotação automática |
| CloudWatch | Métricas, logs, alarmes e dashboards |

A AWS é vasta, mas esses serviços formam o núcleo da maioria das arquiteturas de produção. Domine-os antes de explorar os serviços especializados na periferia.
