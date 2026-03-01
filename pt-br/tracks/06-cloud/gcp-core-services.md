# Serviços Fundamentais do GCP

## Visão Geral

O Google Cloud Platform (GCP) é o terceiro maior provedor de cloud e a plataforma preferida para cargas de trabalho intensivas em dados, machine learning e organizações que usam o Google Workspace. Os pontos fortes do GCP incluem o melhor Kubernetes gerenciado da indústria (GKE), BigQuery para analytics e uma sólida backbone de rede global.

Este capítulo aborda os serviços GCP que todo desenvolvedor backend precisa conhecer: compute (Cloud Run, GKE, Cloud Functions, Compute Engine), storage (Cloud Storage, Filestore), bancos de dados (Cloud SQL, Firestore, Bigtable, Memorystore), redes (VPC, Cloud Load Balancing, Cloud CDN, Cloud DNS) e operações (IAM, Secret Manager, Cloud Monitoring, Cloud Logging).

---

## Pré-requisitos

- Conta GCP (trial gratuita de 90 dias com $300; nível always-free depois)
- Google Cloud CLI (`gcloud`) instalado e configurado
- Conceitos básicos de cloud (veja `cloud-concepts.md`)

```bash
# Instalar gcloud CLI (Linux/WSL)
curl https://sdk.cloud.google.com | bash
exec -l $SHELL
gcloud init

# Autenticar
gcloud auth login
gcloud auth application-default login   # para uso local do SDK

# Definir padrões de projeto e region
gcloud config set project my-project-id
gcloud config set compute/region us-central1
gcloud config set run/region us-central1

# Verificar
gcloud config list
gcloud projects list
```

---

## Conceitos Fundamentais

### Hierarquia de Recursos do GCP

```
Organization (company.com)
  └── Folders (opcional — times, departamentos)
        └── Projects (fronteira de billing, unidade de habilitação de APIs)
              └── Resources (VMs, buckets, bancos de dados, etc.)
```

**Projects** são a unidade organizacional principal. Cada projeto tem seu próprio billing, habilitação de APIs, políticas IAM e quota. Pense em um projeto por ambiente (dev, staging, prod) ou por produto.

```bash
# Criar um projeto
gcloud projects create myapp-prod \
  --name="MyApp Production" \
  --organization=123456789

# Definir como padrão
gcloud config set project myapp-prod

# Habilitar APIs necessárias (deve ser feito por projeto)
gcloud services enable \
  run.googleapis.com \
  sqladmin.googleapis.com \
  redis.googleapis.com \
  secretmanager.googleapis.com \
  artifactregistry.googleapis.com \
  cloudbuild.googleapis.com
```

### Categorias de Serviços

```
Compute:        Cloud Run, GKE, Cloud Functions, Compute Engine, Cloud Run Jobs
Storage:        Cloud Storage, Filestore, Persistent Disk
Databases:      Cloud SQL, Firestore, Bigtable, Spanner, AlloyDB
Caching:        Memorystore (Redis, Memcached)
Networking:     VPC, Cloud Load Balancing, Cloud CDN, Cloud DNS, Cloud Armor
Security:       IAM, Secret Manager, Certificate Manager, Cloud KMS
Operations:     Cloud Monitoring, Cloud Logging, Cloud Trace, Error Reporting
Messaging:      Pub/Sub, Cloud Tasks, Eventarc
Developer:      Artifact Registry, Cloud Build, Cloud Deploy
Data/ML:        BigQuery, Vertex AI, Dataflow, Dataproc
```

---

## Serviços de Compute

### Cloud Run

Containers serverless totalmente gerenciados. O equivalente GCP do AWS Fargate + App Runner. Escala até zero, cobrado por requisição/segundo de CPU, sem cluster para gerenciar.

```bash
# Fazer deploy de um container no Cloud Run
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
  --region us-central1 \
  --platform managed \
  --allow-unauthenticated \       # endpoint público
  --min-instances 1 \             # manter aquecido (evitar cold starts)
  --max-instances 10 \
  --concurrency 100 \             # requisições por instance antes de escalar
  --cpu 1 \
  --memory 512Mi \
  --timeout 30s \
  --set-env-vars NODE_ENV=production \
  --set-secrets DATABASE_URL=database-url:latest \   # do Secret Manager
  --service-account myapp-sa@myapp-prod.iam.gserviceaccount.com

# Obter a URL do serviço
gcloud run services describe myapp-api \
  --region us-central1 \
  --format "value(status.url)"

# Atualizar apenas a imagem (rolling deploy)
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:v2 \
  --region us-central1

# Traffic splitting (canary)
gcloud run services update-traffic myapp-api \
  --region us-central1 \
  --to-revisions myapp-api-00001-abc=90,myapp-api-00002-def=10
```

**Modelo de concorrência do Cloud Run:**
O padrão do Cloud Run é 80 requisições concorrentes por instance. Aumente para apps Node.js com I/O intensivo (até 1000). Diminua para cargas com uso intensivo de CPU.

```bash
gcloud run services update myapp-api \
  --region us-central1 \
  --concurrency 200   # Node.js lida com muitas requisições concorrentes de I/O
```

### Cloud Run Jobs

Para cargas batch (migrações de banco de dados, geração de relatórios, importações de dados) — não para serviços de longa duração.

```bash
# Criar um job
gcloud run jobs create db-migrate \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
  --region us-central1 \
  --command "node" \
  --args "dist/migrate.js" \
  --set-secrets DATABASE_URL=database-url:latest \
  --max-retries 3 \
  --task-timeout 600s

# Executar
gcloud run jobs execute db-migrate --region us-central1

# Agendar (Cloud Scheduler)
gcloud scheduler jobs create http weekly-report \
  --schedule "0 9 * * 1" \
  --uri "https://us-central1-run.googleapis.com/apis/run.googleapis.com/v1/namespaces/myapp-prod/jobs/generate-report:run" \
  --oauth-service-account-email myapp-sa@myapp-prod.iam.gserviceaccount.com \
  --location us-central1
```

### GKE (Google Kubernetes Engine)

Kubernetes gerenciado. A oferta de K8s gerenciado mais madura. Use quando precisar do ecossistema completo do Kubernetes (controllers customizados, service meshes, deploys complexos com múltiplos serviços).

```bash
# Criar um cluster Autopilot (GKE gerencia os nós — recomendado)
gcloud container clusters create-auto myapp-cluster \
  --region us-central1 \
  --release-channel regular

# Obter credenciais para kubectl
gcloud container clusters get-credentials myapp-cluster \
  --region us-central1

# Fazer deploy
kubectl apply -f k8s/

# Verificar status
kubectl get pods -n myapp
kubectl get services -n myapp
```

### Cloud Functions (2ª Geração)

Funções serverless orientadas a eventos. A 2ª geração é construída internamente sobre o Cloud Run — timeouts mais longos, mais memória.

```typescript
// src/index.ts
import { http, HttpFunction } from '@google-cloud/functions-framework';

const handleUsers: HttpFunction = async (req, res) => {
  if (req.method === 'GET') {
    const users = await db.user.findMany();
    res.json(users);
    return;
  }

  if (req.method === 'POST') {
    const user = await db.user.create({ data: req.body });
    res.status(201).json(user);
    return;
  }

  res.status(405).json({ error: 'Method not allowed' });
};

http('users', handleUsers);
```

```bash
# Deploy de uma Cloud Function (2ª geração)
gcloud functions deploy users \
  --gen2 \
  --runtime nodejs22 \
  --region us-central1 \
  --source . \
  --entry-point users \
  --trigger-http \
  --allow-unauthenticated \
  --set-secrets DATABASE_URL=database-url:latest \
  --memory 256MiB \
  --timeout 30s

# Trigger via Pub/Sub
gcloud functions deploy processMessage \
  --gen2 \
  --runtime nodejs22 \
  --region us-central1 \
  --trigger-topic my-topic \
  --entry-point processMessage
```

---

## Serviços de Storage

### Cloud Storage

Object storage equivalente ao S3. Replicado globalmente, 11 noves de durabilidade.

```bash
# Criar um bucket
gcloud storage buckets create gs://myapp-uploads-prod \
  --location us-central1 \
  --uniform-bucket-level-access   # recomendado: desabilitar ACLs por objeto

# Upload
gcloud storage cp ./logo.png gs://myapp-uploads-prod/images/logo.png

# Sincronizar diretório
gcloud storage rsync ./public/ gs://myapp-static-prod/ --delete-unmatched-destination-objects

# Gerar URL assinada
gcloud storage sign-url gs://myapp-uploads-prod/documents/report.pdf \
  --duration 1h \
  --private-key-file=/path/to/key.json
```

Cloud Storage a partir do Node.js (`@google-cloud/storage`):

```typescript
import { Storage } from '@google-cloud/storage';

// Quando rodando no GCP (Cloud Run, GKE), ADC (Application Default Credentials)
// usa automaticamente a service account. Sem credenciais no código.
const storage = new Storage({ projectId: process.env.GOOGLE_CLOUD_PROJECT });
const bucket = storage.bucket(process.env.GCS_BUCKET!);

// Upload
async function uploadFile(destination: string, data: Buffer, contentType: string): Promise<string> {
  const file = bucket.file(destination);
  await file.save(data, { contentType, resumable: false });
  return `https://storage.googleapis.com/${process.env.GCS_BUCKET}/${destination}`;
}

// Gerar URL assinada para arquivos privados
async function getSignedUrl(filePath: string, expiresInSeconds = 3600): Promise<string> {
  const [url] = await bucket.file(filePath).getSignedUrl({
    action: 'read',
    expires: Date.now() + expiresInSeconds * 1000,
  });
  return url;
}

// Upload em streaming (para arquivos grandes)
async function streamUpload(destination: string, readableStream: NodeJS.ReadableStream, contentType: string): Promise<void> {
  const file = bucket.file(destination);
  const writeStream = file.createWriteStream({ contentType });
  await new Promise<void>((resolve, reject) => {
    readableStream.pipe(writeStream)
      .on('finish', resolve)
      .on('error', reject);
  });
}
```

**Classes de storage:**
| Classe | Caso de uso | Duração mínima |
|--------|-------------|----------------|
| Standard | Acessado frequentemente | Nenhuma |
| Nearline | Acessado ~1x/mês | 30 dias |
| Coldline | Acessado ~1x/trimestre | 90 dias |
| Archive | Acessado ~1x/ano | 365 dias |

```bash
# Definir lifecycle policy: mover para Nearline após 30 dias, Archive após 365
gcloud storage buckets update gs://myapp-uploads-prod \
  --lifecycle-file=lifecycle.json
```

```json
{
  "lifecycle": {
    "rule": [
      {
        "action": { "type": "SetStorageClass", "storageClass": "NEARLINE" },
        "condition": { "age": 30, "matchesStorageClass": ["STANDARD"] }
      },
      {
        "action": { "type": "SetStorageClass", "storageClass": "ARCHIVE" },
        "condition": { "age": 365, "matchesStorageClass": ["NEARLINE"] }
      }
    ]
  }
}
```

---

## Serviços de Banco de Dados

### Cloud SQL

Bancos de dados relacionais gerenciados: PostgreSQL, MySQL, SQL Server.

```bash
# Criar uma instance Cloud SQL PostgreSQL
gcloud sql instances create myapp-db \
  --database-version POSTGRES_16 \
  --tier db-g1-small \           # ou db-n1-standard-2 para produção
  --region us-central1 \
  --availability-type REGIONAL \ # HA multi-zona
  --storage-size 20GB \
  --storage-type SSD \
  --storage-auto-increase \
  --backup-start-time 03:00 \
  --retained-backups-count 7 \
  --deletion-protection \
  --no-assign-ip \               # apenas IP privado
  --network projects/myapp-prod/global/networks/myapp-vpc

# Criar banco de dados e usuário
gcloud sql databases create app --instance myapp-db
gcloud sql users create appuser \
  --instance myapp-db \
  --password "$(openssl rand -base64 32)"

# Conectar via Cloud SQL Auth Proxy (dev local)
cloud-sql-proxy myapp-prod:us-central1:myapp-db &
# Depois conecte a localhost:5432 normalmente
```

**Padrão do Cloud SQL Auth Proxy** — o método de conexão recomendado. O proxy gerencia autenticação IAM e TLS automaticamente:

```bash
# Instalar Cloud SQL Auth Proxy
curl -o cloud-sql-proxy \
  https://storage.googleapis.com/cloud-sql-connectors/cloud-sql-proxy/v2.11.0/cloud-sql-proxy.linux.amd64
chmod +x cloud-sql-proxy

# Executar (em background para dev local)
./cloud-sql-proxy myapp-prod:us-central1:myapp-db --port 5432 &

# Conectar normalmente
DATABASE_URL=postgres://appuser:password@127.0.0.1:5432/app
```

Em produção (Cloud Run), use o socket Unix:

```typescript
// DATABASE_URL para Cloud Run (socket Cloud SQL)
// postgres://user:password@/dbname?host=/cloudsql/project:region:instance
const databaseUrl = process.env.DATABASE_URL;
// Cloud Run monta automaticamente o socket do Cloud SQL quando você adiciona
// a conexão Cloud SQL na configuração do serviço:
// gcloud run deploy ... --add-cloudsql-instances myapp-prod:us-central1:myapp-db
```

### Firestore

Banco de dados NoSQL de documentos serverless. Sem gerenciamento de servidor, scaling automático, sincronização em tempo real.

```typescript
import { Firestore, FieldValue } from '@google-cloud/firestore';

// ADC pega automaticamente as credenciais da service account no GCP
const db = new Firestore({ projectId: process.env.GOOGLE_CLOUD_PROJECT });

// Gravar um documento
async function createUser(user: { name: string; email: string }) {
  const ref = db.collection('users').doc();
  await ref.set({
    ...user,
    createdAt: FieldValue.serverTimestamp(),
  });
  return ref.id;
}

// Ler um documento
async function getUser(userId: string) {
  const doc = await db.collection('users').doc(userId).get();
  if (!doc.exists) return null;
  return { id: doc.id, ...doc.data() };
}

// Consulta com filtro
async function getActiveUsers() {
  const snapshot = await db.collection('users')
    .where('status', '==', 'active')
    .orderBy('createdAt', 'desc')
    .limit(100)
    .get();

  return snapshot.docs.map(doc => ({ id: doc.id, ...doc.data() }));
}

// Transações
async function transferCredits(fromId: string, toId: string, amount: number) {
  await db.runTransaction(async (transaction) => {
    const fromRef = db.collection('wallets').doc(fromId);
    const toRef = db.collection('wallets').doc(toId);

    const [from, to] = await transaction.getAll(fromRef, toRef);
    if (!from.exists || !to.exists) throw new Error('Carteira não encontrada');
    if (from.data()!.credits < amount) throw new Error('Saldo insuficiente');

    transaction.update(fromRef, { credits: FieldValue.increment(-amount) });
    transaction.update(toRef, { credits: FieldValue.increment(amount) });
  });
}
```

**Quando usar Firestore vs Cloud SQL:**
- Firestore: apps mobile/tempo real, schema flexível, active-active multi-region, auto-scaling
- Cloud SQL: dados relacionais, consultas complexas, expertise SQL existente, transações ACID — maioria dos apps server-side

### Memorystore for Redis

Redis gerenciado no GCP.

```bash
# Criar uma instance Redis
gcloud redis instances create myapp-cache \
  --size 1 \
  --region us-central1 \
  --redis-version redis_7_0 \
  --tier standard            # standard = HA com replica

# Obter detalhes de conexão
gcloud redis instances describe myapp-cache \
  --region us-central1 \
  --format "value(host,port)"
```

```typescript
import { createClient } from 'redis';

const redis = createClient({
  socket: {
    host: process.env.REDIS_HOST,
    port: Number(process.env.REDIS_PORT ?? 6379),
  },
});
await redis.connect();
```

---

## Serviços de Segurança

### IAM (Identity and Access Management)

O IAM do GCP usa um modelo de **principal + role + recurso**. Principals podem ser contas de usuário, service accounts ou grupos.

```bash
# Criar uma service account para sua aplicação
gcloud iam service-accounts create myapp-sa \
  --display-name "MyApp Service Account"

# Conceder roles à service account
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/cloudsql.client"

gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"

gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/storage.objectAdmin"

# Anexar service account ao serviço Cloud Run
gcloud run deploy myapp-api \
  --service-account myapp-sa@myapp-prod.iam.gserviceaccount.com \
  ...
```

**Workload Identity (GKE):** permite que pods usem uma Kubernetes service account mapeada para uma GCP service account — sem arquivos de chave JSON.

### Secret Manager

```bash
# Criar um secret
echo -n "postgres://appuser:password@/app?host=/cloudsql/..." | \
  gcloud secrets create database-url --data-file=-

# Adicionar nova versão (rotação)
echo -n "nova-connection-string" | \
  gcloud secrets versions add database-url --data-file=-

# Acessar a versão mais recente
gcloud secrets versions access latest --secret database-url
```

A partir do Node.js:

```typescript
import { SecretManagerServiceClient } from '@google-cloud/secret-manager';

const client = new SecretManagerServiceClient();

async function getSecret(secretId: string): Promise<string> {
  const [version] = await client.accessSecretVersion({
    name: `projects/${process.env.GOOGLE_CLOUD_PROJECT}/secrets/${secretId}/versions/latest`,
  });
  return version.payload!.data!.toString();
}

const DATABASE_URL = await getSecret('database-url');
```

### Cloud Monitoring e Cloud Logging

```bash
# Criar uma política de alertas (taxa de erros)
gcloud alpha monitoring policies create \
  --policy-from-file=alert-policy.json

# Consultar logs com gcloud
gcloud logging read \
  'resource.type="cloud_run_revision" AND severity>=ERROR' \
  --project myapp-prod \
  --limit 50 \
  --format json
```

---

## Anti-Padrões a Evitar

### Usar Chaves JSON de Service Account

```bash
# RUIM — baixar uma chave JSON cria uma credencial de longa duração que pode ser vazada
gcloud iam service-accounts keys create key.json \
  --iam-account myapp-sa@myapp-prod.iam.gserviceaccount.com

# BOM — use Workload Identity (GKE), Metadata Server (Compute Engine),
# ou anexe uma service account ao Cloud Run. O ADC cuida de tudo.
```

### Deixar a Service Account Padrão do Compute com Role Editor

A service account padrão do Compute tem permissões amplas em todo o projeto. Crie service accounts dedicadas com roles mínimas para cada serviço.

### Instances Cloud SQL Públicas

```bash
# Sempre use --no-assign-ip (apenas IP privado)
# Conecte via Cloud SQL Auth Proxy ou IP privado direto dentro da VPC
gcloud sql instances patch myapp-db \
  --no-assign-ip   # remover IP público se foi definido
```

### Não Habilitar Versionamento de Objetos em Buckets Críticos

```bash
gcloud storage buckets update gs://myapp-critical-data \
  --versioning
```

---

## Debugging e Solução de Problemas

### Serviço Cloud Run Não Inicia

```bash
# Verificar status do deploy
gcloud run services describe myapp-api --region us-central1

# Listar revisões e seus status
gcloud run revisions list --service myapp-api --region us-central1

# Stream de logs
gcloud run services logs tail myapp-api --region us-central1

# Logs de revisão específica
gcloud logging read \
  'resource.type="cloud_run_revision" AND resource.labels.service_name="myapp-api"' \
  --project myapp-prod --limit 100
```

### Permissão Negada no Secret Manager

```bash
# Verificar se a service account tem role secretAccessor
gcloud projects get-iam-policy myapp-prod \
  --flatten="bindings[].members" \
  --filter="bindings.members:myapp-sa@" \
  --format="table(bindings.role)"
```

### Timeout de Conexão com Cloud SQL

```bash
# Verificar se Cloud SQL Auth Proxy está configurado para Cloud Run
gcloud run services describe myapp-api \
  --region us-central1 \
  --format "value(spec.template.metadata.annotations)"
# Procure por: run.googleapis.com/cloudsql-instances

# Adicionar se estiver faltando
gcloud run services update myapp-api \
  --region us-central1 \
  --add-cloudsql-instances myapp-prod:us-central1:myapp-db
```

---

## Cenários do Mundo Real

### Cenário 1: Deploy de uma API Node.js no Cloud Run

```bash
# 1. Build e push da imagem para o Artifact Registry
gcloud artifacts repositories create myapp \
  --repository-format docker \
  --location us-central1

gcloud builds submit \
  --tag us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest .

# 2. Armazenar DATABASE_URL no Secret Manager
echo -n "postgres://..." | gcloud secrets create database-url --data-file=-

# 3. Criar service account
gcloud iam service-accounts create myapp-sa
gcloud projects add-iam-policy-binding myapp-prod \
  --member "serviceAccount:myapp-sa@myapp-prod.iam.gserviceaccount.com" \
  --role "roles/secretmanager.secretAccessor"

# 4. Fazer deploy
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:latest \
  --region us-central1 \
  --allow-unauthenticated \
  --min-instances 1 \
  --set-secrets DATABASE_URL=database-url:latest \
  --service-account myapp-sa@myapp-prod.iam.gserviceaccount.com \
  --add-cloudsql-instances myapp-prod:us-central1:myapp-db
```

### Cenário 2: Fan-Out com Pub/Sub

```typescript
// Publisher
import { PubSub } from '@google-cloud/pubsub';
const pubsub = new PubSub({ projectId: process.env.GOOGLE_CLOUD_PROJECT });

async function publishEvent(event: { type: string; payload: unknown }) {
  const topic = pubsub.topic('user-events');
  await topic.publishMessage({
    json: event,
    attributes: { eventType: event.type },
  });
}

// Subscriber (Cloud Function ou Cloud Run)
app.post('/pubsub-handler', async (req, res) => {
  const message = Buffer.from(req.body.message.data, 'base64').toString();
  const event = JSON.parse(message);
  await handleEvent(event);
  res.status(204).send();
});
```

---

## Leitura Adicional

- [Documentação do GCP](https://cloud.google.com/docs)
- [Documentação do Cloud Run](https://cloud.google.com/run/docs)
- [Bibliotecas cliente Node.js do Google Cloud](https://cloud.google.com/nodejs/docs/reference)
- [Referência do gcloud CLI](https://cloud.google.com/sdk/gcloud/reference)
- [GCP Architecture Framework](https://cloud.google.com/architecture/framework)
- [Boas práticas do Cloud SQL](https://cloud.google.com/sql/docs/postgres/best-practices)

---

## Resumo

| Serviço | Caso de Uso | Equivalente AWS | Equivalente Azure |
|---------|-------------|-----------------|-------------------|
| Cloud Run | Containers serverless | ECS Fargate / App Runner | Container Apps |
| Cloud Run Jobs | Cargas batch | Tasks ECS Fargate (one-shot) | Container Apps Jobs |
| GKE Autopilot | Kubernetes gerenciado | EKS | AKS |
| Cloud Functions | Serverless orientado a eventos | Lambda | Azure Functions |
| Cloud Storage | Object storage | S3 | Blob Storage |
| Cloud SQL | PostgreSQL/MySQL gerenciado | RDS | Azure Database for PostgreSQL |
| Firestore | NoSQL serverless | DynamoDB | Cosmos DB |
| Memorystore | Redis gerenciado | ElastiCache | Azure Cache for Redis |
| Cloud Load Balancing | LB HTTP(S) global | ALB | Application Gateway |
| Cloud CDN | Rede de entrega de conteúdo | CloudFront | Azure Front Door |
| IAM + Service Accounts | Controle de acesso | Roles IAM | Azure AD + Managed Identity |
| Secret Manager | Armazenamento de secrets | Secrets Manager | Key Vault |
| Cloud Monitoring | Métricas + alertas | CloudWatch | Azure Monitor |

Os recursos matadores do GCP para desenvolvedores backend: a simplicidade do Cloud Run, o Cloud SQL Auth Proxy para conexões seguras ao banco sem chaves, e Application Default Credentials tornando a autenticação transparente entre dev local e produção.
