# Segurança em Cloud

## Visão Geral

As falhas de segurança em cloud quase sempre seguem os mesmos padrões: roles IAM excessivamente permissivos, credenciais expostas, dados sem criptografia e recursos públicos que deveriam ser privados. As ferramentas para prevenir todos esses problemas existem e estão bem documentadas — o problema são os times que as pulam sob pressão de prazo.

Este capítulo cobre os controles de segurança que todo deploy em cloud precisa. Os princípios se aplicam a AWS, Azure e GCP. Fornecemos exemplos de código e comandos de CLI para cada um onde eles diferem.

---

## Pré-requisitos

- AWS Core Services (`aws-core-services.md`) ou equivalente Azure/GCP
- Compreensão básica de conceitos IAM (usuários, roles, políticas)
- Aplicação Node.js/TypeScript rodando em infraestrutura cloud

---

## Conceitos Fundamentais

### O Modelo de Responsabilidade Compartilhada

```
O Provider de Cloud é responsável por:
  Segurança física, hardware, hypervisor, infraestrutura de rede,
  patching de serviços gerenciados (OS do RDS, control plane do ECS, etc.)

VOCÊ é responsável por:
  Configuração de IAM, configuração de rede (security groups, regras de firewall),
  criptografia de dados, segurança da aplicação, gerenciamento de secrets,
  patching de OS (para VMs que você gerencia), configuração de compliance
```

O equívoco mais comum: "Está na AWS, então está seguro." A AWS protege a infraestrutura. Você protege a configuração. Um bucket S3 mal configurado com acesso público de leitura é problema seu, não da AWS.

### Defesa em Profundidade

Nenhum controle único é suficiente. Segurança eficaz em cloud aplica múltiplos controles em camadas:

```
Camada 1: Identidade (IAM, MFA, credenciais de curta duração)
Camada 2: Rede (VPC, security groups, subnets privadas)
Camada 3: Criptografia de dados (em repouso e em trânsito)
Camada 4: Gerenciamento de secrets (sem secrets em texto puro no código/env/logs)
Camada 5: Auditoria e detecção (CloudTrail, Defender for Cloud, Security Command Center)
Camada 6: Segurança da aplicação (validação de input, autenticação, rate limiting)
```

Se uma camada for violada, as outras limitam o raio de impacto.

### Princípio do Menor Privilégio

Todo principal IAM (usuário, role, service account) deve ter exatamente as permissões necessárias para fazer seu trabalho — nada além disso.

```
RUIM:  Função Lambda com AdministratorAccess
BOM:   Função Lambda com s3:GetObject em bucket específico + secretsmanager:GetSecretValue em secret específico

RUIM:  RDS acessível a partir de 0.0.0.0/0
BOM:   RDS acessível apenas a partir do security group da aplicação na porta 5432

RUIM:  Todos os engenheiros têm acesso ao console de produção
BOM:   Acesso à produção exige aprovação separada + credenciais com prazo limitado
```

---

## Exemplos Práticos

### Gerenciamento de Identidade e Acesso

**AWS: Permissões mínimas para uma API Node.js no ECS Fargate:**

```bash
# Criar task role com apenas o que o app precisa
aws iam create-role \
  --role-name myapp-task-role \
  --assume-role-policy-document '{
    "Version": "2012-10-17",
    "Statement": [{
      "Effect": "Allow",
      "Principal": {"Service": "ecs-tasks.amazonaws.com"},
      "Action": "sts:AssumeRole"
    }]
  }'

# Conceder permissões específicas
aws iam put-role-policy \
  --role-name myapp-task-role \
  --policy-name myapp-permissions \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "S3Uploads",
        "Effect": "Allow",
        "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
        "Resource": "arn:aws:s3:::myapp-uploads-prod/*"
      },
      {
        "Sid": "Secrets",
        "Effect": "Allow",
        "Action": "secretsmanager:GetSecretValue",
        "Resource": "arn:aws:secretsmanager:us-east-1:123456789012:secret:myapp/*"
      },
      {
        "Sid": "SQSSend",
        "Effect": "Allow",
        "Action": "sqs:SendMessage",
        "Resource": "arn:aws:sqs:us-east-1:123456789012:myapp-jobs"
      }
    ]
  }'
```

**GCP: Roles mínimas para um serviço Cloud Run:**

```bash
# Criar service account dedicada
gcloud iam service-accounts create myapp-api-sa \
  --display-name "MyApp API Service Account"

# Conceder apenas o que o serviço precisa
gcloud storage buckets add-iam-policy-binding gs://myapp-uploads-prod \
  --member "serviceAccount:myapp-api-sa@myapp-prod.iam.gserviceaccount.com" \
  --role roles/storage.objectAdmin

gcloud secrets add-iam-policy-binding database-url \
  --project myapp-prod \
  --member "serviceAccount:myapp-api-sa@myapp-prod.iam.gserviceaccount.com" \
  --role roles/secretmanager.secretAccessor

gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-api-sa@myapp-prod.iam.gserviceaccount.com" \
  --role roles/cloudsql.client

# Associar ao serviço
gcloud run deploy myapp-api \
  --service-account myapp-api-sa@myapp-prod.iam.gserviceaccount.com \
  ...
```

**Azure: Managed Identity + RBAC:**

```bash
# Habilitar managed identity atribuída ao sistema
az containerapp identity assign \
  --name myapp-api \
  --resource-group myapp-prod-rg

PRINCIPAL_ID=$(az containerapp identity show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --query principalId -o tsv)

# Conceder Key Vault Secrets User (leitura de secrets)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.KeyVault/vaults/myapp-kv

# Conceder Storage Blob Data Contributor (leitura/escrita de blobs)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.Storage/storageAccounts/myappstorage/blobServices/default/containers/uploads
```

### Segurança de Rede

**AWS: Regras de security group — princípio do menor privilégio:**

```bash
# Security group do ALB: aceitar apenas tráfego da internet nas portas 80/443
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Internet-facing ALB" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 443 --cidr 0.0.0.0/0

# Security group da app: aceitar apenas tráfego do ALB (sem acesso direto da internet)
aws ec2 create-security-group \
  --group-name app-sg \
  --description "Application servers" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 3000 \
  --source-group sg-alb   # apenas do ALB, não da internet

# Security group do DB: aceitar apenas tráfego dos servidores de aplicação
aws ec2 create-security-group \
  --group-name db-sg \
  --description "Database" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-db \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app   # apenas dos servidores de aplicação

# NUNCA faça isso:
# --cidr 0.0.0.0/0 em um security group de banco de dados
```

**GCP: Regras de firewall VPC:**

```bash
# Negar todo ingresso por padrão (política padrão do GCP)
# Adicionar regras de permissão explícitas apenas quando necessário

# Permitir HTTPS da internet para o load balancer (health checks dos ranges do Google)
gcloud compute firewall-rules create allow-https-lb \
  --network myapp-vpc \
  --direction INGRESS \
  --action ALLOW \
  --rules tcp:443 \
  --source-ranges 0.0.0.0/0 \
  --target-tags load-balancer

# Permitir que servidores de aplicação acessem o banco de dados
gcloud compute firewall-rules create allow-db-from-app \
  --network myapp-vpc \
  --direction INGRESS \
  --action ALLOW \
  --rules tcp:5432 \
  --source-tags app-server \
  --target-tags database
```

### Gerenciamento de Secrets — Nunca no Código

O erro de segurança #1 em cloud: secrets no código-fonte, em variáveis de ambiente ou em imagens de container.

```typescript
// RUIM — qualquer um desses compromete o secret quando o código é compartilhado/feito deploy
const DB_URL = 'postgres://admin:MyPassw0rd@rds-host.example.com:5432/app';
process.env.DATABASE_URL = 'postgres://admin:MyPassw0rd@rds-host.example.com:5432/app';

// BOM — recuperar em runtime a partir de um secret store gerenciado
// A aplicação tem uma IAM role/managed identity que concede acesso ao secret
// O secret nunca fica no código ou em variáveis de ambiente em texto puro

// AWS
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';
const client = new SecretsManagerClient({ region: 'us-east-1' });
const { SecretString } = await client.send(
  new GetSecretValueCommand({ SecretId: 'myapp/database-url' })
);
const DATABASE_URL = SecretString!;

// Azure
import { SecretClient } from '@azure/keyvault-secrets';
import { DefaultAzureCredential } from '@azure/identity';
const kv = new SecretClient('https://myapp-kv.vault.azure.net', new DefaultAzureCredential());
const { value: DATABASE_URL } = await kv.getSecret('database-url');

// GCP
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';
const sm = new SecretManagerServiceClient();
const [version] = await sm.accessSecretVersion({
  name: 'projects/myapp-prod/secrets/database-url/versions/latest',
});
const DATABASE_URL = version.payload!.data!.toString();
```

**Varredura de secrets no CI/CD:**

```bash
# Instalar git-secrets ou truffleHog para evitar commit de secrets
pip install truffleHog3

# Escanear o repositório atual
trufflehog git file://. --since-commit HEAD~10 --only-verified

# Adicionar ao hook de pre-commit
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
trufflehog git file://. --since-commit HEAD --only-verified --fail
EOF
chmod +x .git/hooks/pre-commit

# Ou usar o secret scanning do GitHub (gratuito para repositórios públicos, pago para privados)
# Ele bloqueia pushes que contêm secrets detectados
```

**Rotação automatizada de secrets (AWS):**

```bash
# Habilitar rotação automática para um secret de banco de dados
aws secretsmanager rotate-secret \
  --secret-id myapp/database-url \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSRotation \
  --rotation-rules AutomaticallyAfterDays=30

# O Lambda rotaciona a senha tanto no Secrets Manager quanto no RDS
# Sua aplicação lê o secret na inicialização — sem alterações de código necessárias
```

### Criptografia em Repouso e em Trânsito

**AWS: Impor criptografia em todo lugar:**

```bash
# S3: impor criptografia server-side
aws s3api put-bucket-encryption \
  --bucket myapp-uploads-prod \
  --server-side-encryption-configuration '{
    "Rules": [{
      "ApplyServerSideEncryptionByDefault": {
        "SSEAlgorithm": "aws:kms",
        "KMSMasterKeyID": "arn:aws:kms:us-east-1:123456789012:key/abc-123"
      },
      "BucketKeyEnabled": true
    }]
  }'

# S3: bloquear acesso público no nível da conta (faça isso no primeiro dia)
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# RDS: habilitar criptografia em repouso (deve ser configurado na criação)
aws rds create-db-instance \
  --db-instance-identifier myapp-prod \
  --storage-encrypted \                       # criptografia AES-256
  --kms-key-id arn:aws:kms:us-east-1:...:key/abc-123 \
  ...

# ECS: impor apenas HTTPS no ALB
aws elbv2 create-listener \
  --load-balancer-arn arn:... \
  --protocol HTTP \
  --port 80 \
  --default-actions \
    Type=redirect,RedirectConfig="{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}"

# Em Node.js: forçar TLS para conexões de banco de dados
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '?sslmode=require',
    },
  },
});
```

**Azure: padrões de criptografia:**

```bash
# Azure criptografa todo o storage em repouso por padrão (AES-256)
# Impor HTTPS-only na storage account
az storage account update \
  --name myappstorageprod \
  --resource-group myapp-prod-rg \
  --https-only true \
  --min-tls-version TLS1_2

# PostgreSQL: impor SSL
az postgres flexible-server update \
  --name myapp-db \
  --resource-group myapp-prod-rg \
  --ssl-enforcement Enabled
```

**GCP: criptografia e TLS:**

```bash
# Cloud Storage: criptografado por padrão (chaves gerenciadas pelo Google)
# Para chaves gerenciadas pelo cliente (CMEK):
gcloud kms keyrings create myapp-keyring --location us-central1
gcloud kms keys create myapp-storage-key \
  --keyring myapp-keyring \
  --location us-central1 \
  --purpose encryption

gcloud storage buckets update gs://myapp-uploads-prod \
  --default-encryption-key projects/myapp-prod/locations/us-central1/keyRings/myapp-keyring/cryptoKeys/myapp-storage-key

# Cloud SQL: SSL é obrigatório por padrão no Flexible Server
# Cloud Run: HTTPS por padrão, HTTP desabilitado
```

### Logging de Auditoria

Saiba o que aconteceu, quem fez e quando. Essencial para resposta a incidentes.

```bash
# AWS: CloudTrail captura todas as chamadas de API
aws cloudtrail create-trail \
  --name myapp-audit-trail \
  --s3-bucket-name myapp-cloudtrail-logs \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name myapp-audit-trail

# Consultar CloudTrail para ações específicas (usando CloudWatch Logs Insights)
aws logs start-query \
  --log-group-name aws-cloudtrail-logs \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, userIdentity.principalId, eventName, requestParameters.bucketName
    | filter eventName = "DeleteBucket" or eventName = "PutBucketPublicAccessBlock"
    | sort @timestamp desc
  '

# GCP: Cloud Audit Logs sempre ativos para acesso a dados
gcloud logging read \
  'logName="projects/myapp-prod/logs/cloudaudit.googleapis.com%2Factivity" AND
   protoPayload.methodName="storage.buckets.delete"' \
  --project myapp-prod --limit 50

# Azure: Activity Log para todas as ações de control plane
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --start-time 2024-01-01 \
  --caller admin@company.com \
  --output table
```

### Segurança de Containers

```dockerfile
# Dockerfile: boas práticas de segurança
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Imagem final distroless: sem shell, sem gerenciador de pacotes, superfície mínima de ataque
FROM gcr.io/distroless/nodejs22-debian12
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Rodar como usuário não-root (o padrão distroless é root — sobrescrever)
USER nonroot

EXPOSE 3000
CMD ["dist/index.js"]
```

```bash
# Escanear imagens de container em busca de vulnerabilidades
# Usando Trivy (open source)
docker pull aquasec/trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest \
  --exit-code 1 \
  --severity HIGH,CRITICAL

# AWS ECR: scan ao fazer push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Verificar resultados do scan
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=latest \
  --query 'imageScanFindings.findings[?severity==`CRITICAL` || severity==`HIGH`]'

# GCP Artifact Registry: varredura automática de vulnerabilidades
gcloud artifacts repositories update myapp \
  --project myapp-prod \
  --location us-central1 \
  --format docker \
  # scanning habilitado por padrão no Artifact Registry
```

---

## Padrões e Boas Práticas

### Checklist de Segurança para Novos Deploys

```
IAM:
[ ] Criar service accounts/roles dedicadas por serviço
[ ] Sem AdministratorAccess ou Owner para roles de aplicação
[ ] MFA habilitado para todos os usuários IAM humanos
[ ] Contas root/admin global com hardware MFA keys
[ ] Access keys rotacionadas a cada 90 dias (melhor ainda: usar roles, sem access keys)

Rede:
[ ] Nenhum banco de dados ou serviço interno em subnets públicas
[ ] Security groups seguem o menor privilégio (portas específicas, origens específicas)
[ ] VPC flow logs habilitados para investigação de incidentes
[ ] WAF na frente de load balancers públicos

Dados:
[ ] Todo storage criptografado em repouso (S3 SSE, criptografia do RDS, criptografia de disco)
[ ] Todas as conexões usam TLS (banco de dados, comunicação interna entre serviços, cliente-API)
[ ] Acesso público a S3/Blob bloqueado no nível da conta
[ ] Criptografia de backup habilitada

Secrets:
[ ] Sem secrets no código, variáveis de ambiente ou imagens Docker
[ ] Todos os secrets em secret stores gerenciados (Secrets Manager, Key Vault, Secret Manager)
[ ] Rotação de secrets automatizada ou agendada
[ ] Histórico do Git escaneado em busca de secrets vazados

Auditoria:
[ ] CloudTrail/Activity Log/Cloud Audit Logs habilitados em todas as regiões
[ ] Logs retidos por 90+ dias (compliance) em storage imutável
[ ] Alertas em ações críticas (login root, mudanças de política, mudanças de security group)
[ ] GuardDuty/Defender for Cloud/Security Command Center habilitados

Aplicação:
[ ] HTTPS imposto (HTTP redireciona para HTTPS)
[ ] Security headers configurados (HSTS, CSP, X-Frame-Options)
[ ] Validação de input com Zod ou similar no boundary da API
[ ] Rate limiting em endpoints de autenticação
[ ] Varredura de dependências (npm audit, Dependabot)
```

### MFA para Acesso ao Console Cloud

```bash
# AWS: impor MFA via política IAM
# Associar esta política a usuários IAM — eles não conseguem fazer nada até se autenticar com MFA
aws iam put-user-policy \
  --user-name myuser \
  --policy-name RequireMFA \
  --policy-document '{
    "Version": "2012-10-17",
    "Statement": [
      {
        "Sid": "AllowManageMFA",
        "Effect": "Allow",
        "Action": ["iam:GetMFADevice", "iam:CreateVirtualMFADevice",
                   "iam:EnableMFADevice", "iam:ListMFADevices"],
        "Resource": ["arn:aws:iam::*:mfa/${aws:username}",
                     "arn:aws:iam::*:user/${aws:username}"]
      },
      {
        "Sid": "DenyWithoutMFA",
        "Effect": "Deny",
        "NotAction": ["iam:GetMFADevice", "iam:CreateVirtualMFADevice",
                      "iam:EnableMFADevice", "iam:ListMFADevices",
                      "sts:GetSessionToken"],
        "Resource": "*",
        "Condition": {"BoolIfExists": {"aws:MultiFactorAuthPresent": "false"}}
      }
    ]
  }'
```

---

## Anti-Padrões a Evitar

### Permissões IAM com Wildcard

```json
// RUIM — concede toda e qualquer ação em todo e qualquer recurso
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// RUIM — erro comum de "próximo o suficiente": wildcard no resource
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}

// BOM — ações específicas em recurso específico
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::myapp-uploads-prod/*"
}
```

### Buckets S3 Públicos

```bash
# Verificar se algum bucket é público
aws s3api list-buckets --query 'Buckets[].Name' --output text | while read bucket; do
  PUBLIC=$(aws s3api get-public-access-block --bucket "$bucket" 2>/dev/null | \
    jq '.PublicAccessBlockConfiguration | to_entries | map(select(.value == false)) | length')
  if [ "$PUBLIC" -gt 0 ]; then
    echo "AVISO: $bucket tem configurações de acesso público desabilitadas"
  fi
done

# Correção: habilitar bloqueio de acesso público em todos os buckets
aws s3api put-public-access-block \
  --bucket BUCKET_NAME \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Access Keys de Longa Duração

```bash
# Encontrar access keys antigas (mais de 90 dias)
aws iam list-users --query 'Users[].UserName' --output text | while read user; do
  aws iam list-access-keys --user-name "$user" \
    --query "AccessKeyMetadata[?Status=='Active'].{User:'$user',Key:AccessKeyId,Created:CreateDate}" \
    --output table
done

# Rotacionar: criar nova key, atualizar a aplicação, desativar a key antiga
aws iam create-access-key --user-name myuser
# Atualizar onde quer que a key seja usada
aws iam update-access-key --access-key-id OLD_KEY --status Inactive --user-name myuser
# Após verificar que a nova key funciona:
aws iam delete-access-key --access-key-id OLD_KEY --user-name myuser
```

### Não Verificar Dependências em Busca de Vulnerabilidades

```bash
# No CI/CD, falhar o build em vulnerabilidades high/critical
npm audit --audit-level=high

# GitHub Dependabot: habilitar em .github/dependabot.yml
cat > .github/dependabot.yml << 'EOF'
version: 2
updates:
  - package-ecosystem: "npm"
    directory: "/"
    schedule:
      interval: "weekly"
    open-pull-requests-limit: 10
EOF
```

---

## Debugging e Troubleshooting

### "Access Denied" — Descobrindo o Porquê

**AWS:**
```bash
# Usar o IAM Policy Simulator para testar permissões sem realmente executar a ação
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/myapp-task-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::myapp-uploads-prod/test.txt

# Verificar o CloudTrail para a ação negada
aws logs start-query \
  --log-group-name aws-cloudtrail-logs \
  --start-time $(date -d '1 hour ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, errorCode, errorMessage, userIdentity.arn
    | filter errorCode = "AccessDenied"
    | sort @timestamp desc
    | limit 20
  '
```

**GCP:**
```bash
# Policy Troubleshooter
gcloud policy-troubleshoot iam \
  projects/myapp-prod \
  --principal-email myapp-api-sa@myapp-prod.iam.gserviceaccount.com \
  --permission storage.objects.create

# Verificar logs de auditoria para ações negadas
gcloud logging read \
  'protoPayload.status.code=7 AND
   protoPayload.authenticationInfo.principalEmail="myapp-api-sa@myapp-prod.iam.gserviceaccount.com"' \
  --project myapp-prod --limit 20
```

### Detecção de Configurações Incorretas de Segurança

```bash
# AWS Security Hub: agrega findings de GuardDuty, Inspector, Config
aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}' \
  --query 'Findings[].{Title:Title,Resource:Resources[0].Id}' \
  --output table

# Azure Defender for Cloud (anteriormente Security Center)
az security assessment list \
  --query "[?status.code=='Unhealthy'].{Title:displayName,Severity:metadata.severity}" \
  --output table

# GCP Security Command Center
gcloud scc findings list \
  --organization ORG_ID \
  --filter "state=ACTIVE AND severity=CRITICAL" \
  --format "table(name,finding.category,finding.createTime)"
```

---

## Cenários Reais

### Cenário 1: Respondendo a uma AWS Access Key Vazada

```bash
# 1. Desativar a key imediatamente
aws iam update-access-key \
  --access-key-id LEAKED_KEY_ID \
  --status Inactive \
  --user-name affected-user

# 2. Investigar o que aconteceu com ela
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=LEAKED_KEY_ID \
  --start-time $(date -d '30 days ago' --iso-8601) \
  --query 'Events[].{Time:EventTime,Action:EventName,Source:SourceIPAddress,Resource:Resources[0].ResourceName}' \
  --output table

# 3. Verificar recursos não autorizados
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Region:Placement.AvailabilityZone}'
aws s3api list-buckets
aws iam list-users  # verificar usuários não autorizados criados

# 4. Rotacionar todos os secrets que o principal comprometido poderia ter acessado
# 5. Deletar a key permanentemente
aws iam delete-access-key --access-key-id LEAKED_KEY_ID --user-name affected-user

# 6. Registrar um postmortem e melhorar:
#    - Migrar para IAM roles (sem access keys)
#    - Adicionar varredura de secrets aos hooks de pre-commit
#    - Adicionar GitHub Secret Scanning ao repositório
```

### Cenário 2: Hardening de um Deploy Existente

```bash
# Auditoria 1: encontrar buckets S3 públicos
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  xargs -I{} aws s3api get-bucket-acl --bucket {} --query 'Grants[?Grantee.URI==`http://acs.amazonaws.com/groups/global/AllUsers`]'

# Auditoria 2: encontrar security groups sem restrição
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`] && !starts_with(FromPort, `443`)]].{ID:GroupId,Name:GroupName}' \
  --output table

# Auditoria 3: encontrar instâncias RDS com acesso público
aws rds describe-db-instances \
  --query 'DBInstances[?PubliclyAccessible==`true`].{ID:DBInstanceIdentifier,Engine:Engine}' \
  --output table

# Auditoria 4: encontrar funções Lambda com permissões excessivas
aws lambda list-functions --query 'Functions[].FunctionArn' --output text | \
  xargs -I{} aws lambda get-function-configuration --function-name {} \
  --query '{Function:FunctionName,Role:Role}'
```

---

## Leitura Adicional

- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [AWS GuardDuty — detecção de ameaças](https://aws.amazon.com/guardduty/)
- [Azure Security Benchmark](https://learn.microsoft.com/security/benchmark/azure/)
- [GCP Security Best Practices](https://cloud.google.com/security/best-practices)
- [OWASP Cloud Security Guide](https://owasp.org/www-project-cloud-native-application-security-top-10/)
- [CIS Benchmarks para AWS/Azure/GCP](https://www.cisecurity.org/cis-benchmarks)
- [truffleHog — varredura de secrets](https://github.com/trufflesecurity/trufflehog)

---

## Resumo

| Controle | AWS | Azure | GCP | Prioridade |
|----------|-----|-------|-----|------------|
| IAM de menor privilégio | IAM roles | Managed Identity + RBAC | Service Accounts | Crítica |
| Gerenciamento de secrets | Secrets Manager | Key Vault | Secret Manager | Crítica |
| Segmentação de rede | VPC + Security Groups | VNet + NSG | VPC + Firewall Rules | Crítica |
| Criptografia em repouso | SSE-KMS | Padrão (gerenciado pelo Azure) | Padrão (gerenciado pelo Google) | Crítica |
| Criptografia em trânsito | ACM + HTTPS enforce | Enforce TLS | Enforce TLS | Crítica |
| Controle de acesso público | S3 Block Public Access | Storage account HTTPS | Uniform bucket-level access | Crítica |
| Logging de auditoria | CloudTrail | Activity Log | Cloud Audit Logs | Alta |
| Detecção de ameaças | GuardDuty | Defender for Cloud | Security Command Center | Alta |
| Varredura de vulnerabilidades | ECR Image Scanning | Defender for Containers | Artifact Registry scanning | Alta |
| Varredura de secrets | GitHub Secret Scanning | GitHub Secret Scanning | GitHub Secret Scanning | Alta |
| MFA | IAM virtual MFA | Azure AD MFA | Google Workspace MFA | Alta |

Segurança não é uma tarefa única. Habilite esses controles no primeiro dia. Execute auditorias de segurança mensalmente. Responda a findings em até 72 horas para críticos, 7 dias para altos. Revise permissões IAM trimestralmente — elas se acumulam e desviam para o excesso ao longo do tempo.
