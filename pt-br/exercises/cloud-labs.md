# Laboratórios de Cloud

> Labs práticos para aprender serviços de provedores cloud através de cenários práticos e realistas. Os labs são focados principalmente em AWS (com alguns labs em GCP onde indicado). Cada lab inclui notas de conscientização de custo — fique dentro dos limites do free tier onde possível, e sempre destrua os recursos ao terminar. Os pré-requisitos assumem AWS CLI configurado com as permissões IAM adequadas.

---

## Lab 1 — Deploy de App em Container no AWS ECS Fargate (Intermediário)

**Objetivo:** Fazer deploy da imagem Docker Node.js no AWS ECS Fargate — uma plataforma de container serverless — sem gerenciar instâncias EC2.

**Pré-requisitos:**
- AWS CLI configurado (`aws sts get-caller-identity` funciona).
- Imagem Docker enviada para o Amazon ECR (Elastic Container Registry).
- Conta AWS com permissões: ECS, ECR, IAM, VPC, CloudWatch.

**Tarefas:**

1. **Push para o ECR:**
   ```bash
   aws ecr create-repository --repository-name myapp --region us-east-1
   aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com
   docker tag myapp:local <account>.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
   docker push <account>.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
   ```

2. **Crie um Cluster ECS:** Tipo de launch Fargate, sem capacidade EC2.

3. **Defina uma Task Definition:**
   - Imagem do container: a URI do ECR.
   - CPU: `256` (.25 vCPU), Memória: `512` MB.
   - Mapeamento de porta: `3000`.
   - Variáveis de ambiente do AWS Secrets Manager (não em texto puro).
   - Configuração de log: driver `awslogs` → CloudWatch log group `/ecs/myapp`.

4. **Crie um ECS Service:**
   - Desired count: `2` tasks.
   - Launch type: `FARGATE`.
   - Networking: subnets públicas (ou privadas com NAT), security group permitindo porta `3000`.
   - Anexe um Application Load Balancer (ALB) apontando para a porta `3000`.

5. **Configure o ALB:**
   - Health check do target group: `GET /health`, threshold 2 healthy.
   - Listener: porta 80 (adicione HTTPS depois).

6. **Habilite auto-scaling:**
   - Scale out se CPU > 70% por 3 minutos (adicione 1 task).
   - Scale in se CPU < 20% por 10 minutos (remova 1 task).
   - Min tasks: 2, Max tasks: 10.

**Verificação:**
- [ ] `curl http://<alb-dns-name>/health` retorna 200.
- [ ] O console ECS mostra 2 tasks no estado `RUNNING`.
- [ ] O CloudWatch log group `/ecs/myapp` tem entradas de log de ambas as tasks.
- [ ] Parar uma task manualmente: o ECS inicia automaticamente uma substituição em 30 segundos.
- [ ] Gerar carga (Apache Bench) aciona a política de scale-out e adiciona uma task.
- [ ] `aws ecs describe-services` mostra `desiredCount` e `runningCount` iguais.

**Nota de custo:** 2 tasks Fargate (.25 vCPU, 512MB) + ALB ≈ $25/mês. Destrua após o lab. O free tier não cobre o Fargate.

---

## Lab 2 — Hospedagem de Site Estático com S3 e CloudFront (Iniciante)

**Objetivo:** Hospedar um site estático no S3 e servi-lo globalmente via CDN CloudFront com HTTPS e domínio customizado.

**Pré-requisitos:**
- Um build de site estático (HTML/CSS/JS em uma pasta `dist/` — use `vite build` em qualquer app React).
- Um domínio no Route 53 (ou um domínio existente onde você pode atualizar os NS records).
- Certificado ACM para seu domínio em `us-east-1` (necessário para o CloudFront).

**Tarefas:**

1. **Crie um bucket S3:**
   - Nome do bucket: `myapp-static-<account-id>`.
   - Bloqueie todo acesso público: ATIVADO (o CloudFront irá acessá-lo, não o público diretamente).
   - Habilite versionamento.

2. **Faça upload do build estático:**
   ```bash
   aws s3 sync dist/ s3://myapp-static-<account-id>/ --delete
   ```

3. **Crie uma distribuição CloudFront:**
   - Origin: o bucket S3 usando um Origin Access Control (OAC) — não uma URL pública do bucket.
   - Default root object: `index.html`.
   - Custom error pages: `403` e `404` → `index.html` com status `200` (necessário para roteamento de SPA).
   - Price class: `PriceClass_100` (US, Canadá, Europa apenas — mais barato).
   - Alternate domain name: `app.seudominio.com`.
   - Certificado SSL: certificado ACM de `us-east-1`.

4. **Atualize a política do bucket S3** para permitir que apenas o CloudFront OAC leia os objetos.

5. **Configure o Route 53:** Crie um registro `A` alias para `app.seudominio.com` apontando para a distribuição CloudFront.

6. **Adicione comportamentos de cache:**
   - `/assets/*`: Cache-Control `max-age=31536000` (1 ano — assets têm nomes de arquivo com hash).
   - `index.html`: Cache-Control `no-cache` (sempre fresco).
   - Defina isso como políticas de headers de resposta do CloudFront, não nos metadados do S3.

7. **Invalide o cache após um deploy:**
   ```bash
   aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
   ```

**Verificação:**
- [ ] `curl https://app.seudominio.com` retorna o site com HTTP 200.
- [ ] `curl http://app.seudominio.com` redireciona para HTTPS (comportamento padrão do CloudFront).
- [ ] Teste do SSL Labs mostra nota A.
- [ ] `curl -I https://app.seudominio.com/rota-inexistente` retorna 200 (fallback SPA funcionando).
- [ ] Acesso direto pela URL do S3 é bloqueado: `curl https://myapp-static-<id>.s3.amazonaws.com/index.html` retorna 403.
- [ ] Após fazer re-deploy com `sync --delete`, uma invalidação do CloudFront limpa os assets desatualizados.
- [ ] `x-cache: Hit from cloudfront` aparece nos headers de resposta na segunda requisição.

**Nota de custo:** S3 + CloudFront está no free tier para tráfego baixo. Certificados ACM são gratuitos.

---

## Lab 3 — RDS PostgreSQL com Read Replica (Intermediário)

**Objetivo:** Provisionar uma instância primária Amazon RDS PostgreSQL, criar uma read replica, configurar a aplicação para usar ambas e testar failover.

**Pré-requisitos:**
- Permissões AWS: RDS, VPC, EC2 (para security groups).
- A aplicação Node.js dos labs anteriores, configurada com driver Prisma ou `pg`.
- Uma VPC com pelo menos duas subnets privadas em diferentes AZs.

**Tarefas:**

1. **Crie um RDS subnet group** abrangendo pelo menos duas AZs.

2. **Lance a instância RDS primária:**
   - Engine: PostgreSQL 15.
   - Instance class: `db.t3.micro` (elegível para free tier).
   - Storage: 20 GB gp2, com autoscaling de storage habilitado.
   - Multi-AZ: habilitado (para HA em produção — note: custa 2x).
   - Security group: permita a porta 5432 apenas do security group da aplicação.
   - Parameter group: defina `log_min_duration_statement = 1000` (logue queries lentas > 1s).

3. **Crie uma read replica** a partir do primário em uma AZ diferente.

4. **Configure a aplicação** com duas connection strings:
   - `DATABASE_URL`: primário (para escritas).
   - `DATABASE_READ_URL`: read replica (para queries de leitura pesada como relatórios e listagens).
   - Na camada Prisma ou de query: roteie queries de leitura explicitamente para a réplica.

5. **Teste a réplica:** Escreva um registro no primário e leia-o da réplica. Meça o lag de replicação com:
   ```sql
   SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
   ```

6. **Simule falha do primário:** Use o console AWS para "Reboot with failover". Observe o standby Multi-AZ ser promovido e o endpoint DNS ser trocado.

7. **Habilite o Performance Insights** no primário e identifique as principais queries SQL por carga.

**Verificação:**
- [ ] A aplicação conecta a tanto o primário quanto a réplica sem erro.
- [ ] O lag de replicação fica abaixo de 1 segundo em carga normal.
- [ ] Após um failover Multi-AZ, a aplicação reconecta automaticamente em 60 segundos (propagação DNS).
- [ ] `SHOW server_type;` — ou verificar via `pg_is_in_recovery()` — confirma que a réplica está em modo somente leitura.
- [ ] Um `INSERT` direto no endpoint da réplica retorna erro: "cannot execute INSERT in a read-only transaction".
- [ ] A métrica CloudWatch `ReadLatency` para a réplica fica abaixo de 10ms.
- [ ] O Enhanced Monitoring do RDS mostra CPU/memória/conexões por segundo no console.

**Nota de custo:** `db.t3.micro` single-AZ é free tier. Multi-AZ dobra o custo para ~$28/mês. A read replica é uma instância adicional (~$14/mês). Destrua após o lab.

---

## Lab 4 — Deploy no Google Cloud Run (Intermediário)

**Objetivo:** Fazer deploy da aplicação Node.js em container no Google Cloud Run — um serviço de container serverless totalmente gerenciado e com autoscaling — e configurá-lo para uso em produção.

**Pré-requisitos:**
- Conta Google Cloud com billing habilitado.
- CLI `gcloud` instalado e configurado (`gcloud auth login`).
- Imagem Docker construída e pronta para push.

**Tarefas:**

1. **Habilite as APIs necessárias:**
   ```bash
   gcloud services enable run.googleapis.com artifactregistry.googleapis.com secretmanager.googleapis.com
   ```

2. **Push para o Artifact Registry:**
   ```bash
   gcloud artifacts repositories create myapp --repository-format=docker --location=us-central1
   docker tag myapp:local us-central1-docker.pkg.dev/<project>/myapp/myapp:latest
   gcloud auth configure-docker us-central1-docker.pkg.dev
   docker push us-central1-docker.pkg.dev/<project>/myapp/myapp:latest
   ```

3. **Armazene segredos no Secret Manager:**
   ```bash
   echo -n "postgres://..." | gcloud secrets create DATABASE_URL --data-file=-
   ```

4. **Faça deploy no Cloud Run:**
   ```bash
   gcloud run deploy myapp \
     --image us-central1-docker.pkg.dev/<project>/myapp/myapp:latest \
     --region us-central1 \
     --platform managed \
     --set-secrets DATABASE_URL=DATABASE_URL:latest \
     --min-instances 1 \
     --max-instances 10 \
     --concurrency 100 \
     --cpu 1 \
     --memory 512Mi \
     --allow-unauthenticated
   ```

5. **Configure um domínio customizado:** Mapeie seu domínio para o serviço Cloud Run via UI de mapeamento de domínio do Cloud Run.

6. **Configure traffic splitting** para um deploy canary:
   ```bash
   gcloud run services update-traffic myapp \
     --to-revisions myapp-00002-xxx=10,myapp-00001-yyy=90
   ```

7. **Configure concorrência e alocação de CPU do Cloud Run:** Defina `--cpu-throttling` para alocar CPU apenas durante o tratamento de requisições (otimização de custo).

**Verificação:**
- [ ] `gcloud run services describe myapp --region us-central1` mostra status `READY`.
- [ ] `curl https://<cloud-run-url>/health` retorna 200.
- [ ] Segredos são montados como variáveis de ambiente — verifique com `gcloud run services describe` que nenhum segredo em texto puro aparece na configuração.
- [ ] Após fazer deploy de uma nova revisão, o traffic split mostra 10% indo para a nova revisão.
- [ ] `gcloud run revisions list` mostra ambas as revisões ativas.
- [ ] Escalonamento: 0 instâncias em repouso (min-instances=0 para testar cold start), escala para múltiplas sob carga.
- [ ] Cloud Logging mostra logs JSON estruturados da aplicação.

**Nota de custo:** O Cloud Run tem um free tier generoso: 2 milhões de requisições/mês, 360.000 vCPU-segundos, 180.000 GB-segundos. A maioria dos labs cabe no free tier.

---

## Lab 5 — Configurar Alertas de Billing e Gerenciamento de Custo AWS (Iniciante)

**Objetivo:** Configurar alertas de billing, budgets e tags de alocação de custo na AWS para nunca ser surpreendido por uma conta inesperada.

**Pré-requisitos:**
- Conta root AWS ou usuário IAM com permissões de billing.
- Alertas de billing devem ser habilitados nas preferências de billing da conta root.

**Tarefas:**

1. **Habilite alertas de billing** em AWS Billing → Billing Preferences → marque "Receive Billing Alerts".

2. **Crie alarmes de billing no CloudWatch** (métricas de billing estão apenas em `us-east-1`):
   - Alarme 1: `EstimatedCharges` > $5 → notificação SNS para seu email.
   - Alarme 2: `EstimatedCharges` > $20 → segunda notificação (escalação).
   - Alarme 3: `EstimatedCharges` > $50 → notificação de "pânico".

3. **Crie um AWS Budget** via console de Budgets:
   - Budget mensal de custo: limite de $10.
   - Alerta em 80% real ($8) e 100% real ($10).
   - Alerta em 100% de gasto previsto.
   - Notificação: email.

4. **Habilite Tags de Alocação de Custo:**
   - Tague todos os recursos existentes com `Project=dev-bible-labs` e `Environment=dev`.
   - No Cost Explorer, ative a tag `Project` como tag de alocação de custo.

5. **Explore o Cost Explorer:**
   - Veja os últimos 30 dias de gasto agrupados por serviço.
   - Identifique os 3 serviços mais caros.
   - Crie um relatório salvo para "Gasto mensal EC2 por região".

6. **Configure o AWS Cost Anomaly Detection:**
   - Crie um monitor para todos os serviços AWS.
   - Threshold de alerta: $5 de anomalia individual.
   - Notificação via SNS por email.

7. **Revise o Trusted Advisor** (tier básico):
   - Note quaisquer avisos de "Service Limits".
   - Note quaisquer instâncias EC2 subutilizadas.

**Verificação:**
- [ ] `aws cloudwatch describe-alarms --alarm-names "billing-alert-5"` mostra o alarme no estado `OK`.
- [ ] O budget está visível no console de Budgets com os thresholds corretos.
- [ ] O Cost Explorer mostra gastos agrupados pela tag `Project` (pode levar 24h para as tags aparecerem).
- [ ] O monitor de Anomaly Detection está ativo.
- [ ] Você recebe uma notificação de teste de alarme baixando temporariamente o threshold abaixo do gasto atual.

---

## Lab 6 — Auditoria de Menor Privilégio IAM (Intermediário)

**Objetivo:** Auditar a configuração IAM de uma conta AWS, identificar usuários e roles com privilégios excessivos e remediar aplicando políticas de menor privilégio.

**Pré-requisitos:**
- Permissões AWS IAM: `iam:*`, `access-analyzer:*`.
- Uma conta AWS com pelo menos alguns usuários e roles IAM para auditar.

**Tarefas:**

1. **Gere um Credential Report:**
   ```bash
   aws iam generate-credential-report
   aws iam get-credential-report --output text --query Content | base64 -d > cred-report.csv
   ```
   Identifique: usuários com access keys não utilizadas por mais de 90 dias, usuários sem MFA, existência de access key da conta root.

2. **Habilite o IAM Access Analyzer:**
   ```bash
   aws accessanalyzer create-analyzer --analyzer-name myaccount-analyzer --type ACCOUNT
   ```
   Revise os findings: acesso externo a buckets S3, chaves KMS, funções Lambda.

3. **Revise políticas inline vs. gerenciadas:** Identifique usuários com `AdministratorAccess` ou ações `*` em recursos `*`. Documente a lista.

4. **Remedie um usuário com privilégio excessivo:**
   - Identifique os serviços que ele realmente usa via `aws iam generate-service-last-accessed-details`.
   - Crie uma política gerenciada customizada concedendo apenas os serviços e ações acessados.
   - Substitua a política ampla pela customizada.
   - Verifique que o usuário ainda pode realizar suas tarefas.

5. **Aplique MFA para usuários do console:** Crie uma política IAM que nega todas as ações exceto `iam:CreateVirtualMFADevice` e `iam:EnableMFADevice` para usuários sem MFA habilitado. Anexe a todos os usuários do console.

6. **Rotacione uma access key com segurança:**
   - Crie uma nova access key para um usuário IAM.
   - Atualize a aplicação/CI usando a chave antiga para usar a nova.
   - Aguarde 24 horas (monitore por erros).
   - Desative a chave antiga. Aguarde 48 horas. Delete-a.

7. **Defina uma política de senha:** mínimo de 14 caracteres, exigir maiúscula, minúscula, número, símbolo, sem reutilização das últimas 12 senhas, idade máxima de 90 dias.

**Verificação:**
- [ ] O Credential Report não mostra usuários com access keys não utilizadas por mais de 90 dias (ou você as desabilitou).
- [ ] O IAM Access Analyzer não mostra findings de acesso externo não intencional.
- [ ] O usuário remediado pode realizar as ações necessárias com a nova política de menor privilégio.
- [ ] O usuário remediado não pode realizar ações fora de seu escopo (teste uma explicitamente).
- [ ] `aws iam get-account-password-policy` retorna a política estrita configurada.
- [ ] A conta root não tem access keys ativas (coluna `CredentialReport`: `<root_account>` — key1_active = false).

---

## Lab 7 — Função Serverless com AWS Lambda e API Gateway (Intermediário)

**Objetivo:** Fazer deploy de uma função Node.js no AWS Lambda, expô-la via API Gateway e conectá-la ao DynamoDB — a stack serverless completa.

**Pré-requisitos:**
- Permissões AWS: Lambda, API Gateway, DynamoDB, IAM.
- Conhecimento do runtime Node.js 20.
- AWS SAM CLI ou Serverless Framework instalado.

**Tarefas:**

1. **Crie uma tabela DynamoDB:**
   - Nome da tabela: `todos`
   - Partition key: `userId` (String)
   - Sort key: `todoId` (String)
   - Billing: on-demand (pay-per-request — elegível para free tier).

2. **Escreva a função Lambda** (`src/handler.ts`):
   - `GET /todos` → `Query` no DynamoDB por todos os todos do usuário autenticado.
   - `POST /todos` → `PutItem` com UUID gerado.
   - `DELETE /todos/{todoId}` → `DeleteItem`.

3. **Crie uma role IAM** para o Lambda com permissões mínimas:
   - `dynamodb:Query`, `dynamodb:PutItem`, `dynamodb:DeleteItem` em `arn:aws:dynamodb:*:*:table/todos`.
   - `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` para o CloudWatch.

4. **Faça deploy usando AWS SAM** (`template.yaml`):
   - Defina a função Lambda, o trigger do API Gateway e a tabela DynamoDB no template SAM.
   - `sam build && sam deploy --guided`

5. **Configure um Lambda layer** para utilitários compartilhados (ex.: a biblioteca de validação) para reduzir o tamanho do pacote e permitir reuso do layer.

6. **Teste localmente com SAM local:**
   ```bash
   sam local start-api
   curl http://localhost:3000/todos
   ```

7. **Adicione um Lambda Powertools layer** para logging estruturado e rastreamento X-Ray.

**Verificação:**
- [ ] `curl https://<api-id>.execute-api.us-east-1.amazonaws.com/Prod/todos` retorna um array vazio `[]`.
- [ ] `POST /todos` com `{ "title": "test" }` cria um registro — verifique no console DynamoDB.
- [ ] Os CloudWatch Logs mostram entradas de log JSON estruturadas de cada invocação.
- [ ] Tempo de cold start do Lambda < 2 segundos (meça via métrica `Init Duration` do CloudWatch).
- [ ] A role IAM do Lambda não pode escrever em nenhuma outra tabela DynamoDB (teste com `aws dynamodb put-item --table-name outra-tabela`).
- [ ] O trace X-Ray mostra a chamada DynamoDB como um segmento dentro da execução Lambda.

**Nota de custo:** Lambda + API Gateway + DynamoDB são todos elegíveis para free tier. 1M requisições Lambda/mês gratuitas.

---

## Lab 8 — Ciclo de Vida de Object Storage e Arquivamento no Glacier (Iniciante)

**Objetivo:** Configurar políticas de ciclo de vida do S3 para transicionar automaticamente objetos para tiers de armazenamento mais baratos e expirar arquivos antigos — uma habilidade fundamental de otimização de custo.

**Pré-requisitos:**
- Um bucket S3 com alguns objetos de teste de idades variadas.
- Conhecimento básico de S3.

**Tarefas:**

1. **Faça upload de objetos de teste** com diferentes datas `LastModified` (simule fazendo upload com metadados):
   ```bash
   aws s3 cp testfile.log s3://mybucket/logs/2024/testfile.log
   ```

2. **Crie uma configuração de ciclo de vida** (`lifecycle.json`):
   ```json
   {
     "Rules": [
       {
         "ID": "log-archival",
         "Status": "Enabled",
         "Filter": { "Prefix": "logs/" },
         "Transitions": [
           { "Days": 30, "StorageClass": "STANDARD_IA" },
           { "Days": 90, "StorageClass": "GLACIER_IR" },
           { "Days": 365, "StorageClass": "DEEP_ARCHIVE" }
         ],
         "Expiration": { "Days": 2555 }
       }
     ]
   }
   ```
   Aplique: `aws s3api put-bucket-lifecycle-configuration --bucket mybucket --lifecycle-configuration file://lifecycle.json`

3. **Crie uma segunda regra** para uploads multipart incompletos:
   - Aborte uploads multipart incompletos após 7 dias. (Eles se acumulam silenciosamente e custam dinheiro.)

4. **Habilite o S3 Intelligent-Tiering** para o prefixo `assets/` — deixe a AWS otimizar automaticamente a classe de armazenamento com base nos padrões de acesso.

5. **Calcule as economias de custo de armazenamento:** Use o dashboard do S3 Storage Lens para estimar quanto seus objetos atuais custariam em Standard vs. Intelligent-Tiering.

6. **Teste uma restauração do Glacier:** Faça a transição de um objeto para o Glacier, depois inicie uma restauração (`Expedited`, 1–5 minutos). Baixe-o após a restauração.

7. **Configure o S3 Storage Lens** para monitorar métricas de buckets em toda a conta.

**Verificação:**
- [ ] `aws s3api get-bucket-lifecycle-configuration --bucket mybucket` retorna as regras configuradas.
- [ ] O console S3 mostra a regra de ciclo de vida como `Active`.
- [ ] Após iniciar uma restauração do Glacier, os metadados do objeto mostram `Restore: ongoing-request="true"`.
- [ ] Após a restauração ser concluída, `Restore: ongoing-request="false", expiry-date="..."`.
- [ ] O dashboard do S3 Storage Lens mostra o tamanho total de armazenamento e contagem de objetos.
- [ ] A regra de abortar uploads multipart está visível na configuração de ciclo de vida.

---

## Lab 9 — Networking VPC e Security Groups (Intermediário)

**Objetivo:** Projetar e construir uma VPC AWS segura com subnets públicas e privadas, um NAT Gateway e security groups devidamente configurados — a fundação de rede para qualquer deploy em produção.

**Pré-requisitos:**
- Permissões AWS: VPC, EC2, NAT Gateway.
- Conhecimento básico de redes (CIDR, tabelas de roteamento, subnets).

**Tarefas:**

1. **Crie uma VPC** com CIDR `10.0.0.0/16`.

2. **Crie subnets:**
   - Subnet pública A: `10.0.1.0/24` em `us-east-1a`
   - Subnet pública B: `10.0.2.0/24` em `us-east-1b`
   - Subnet privada A: `10.0.10.0/24` em `us-east-1a`
   - Subnet privada B: `10.0.11.0/24` em `us-east-1b`

3. **Crie e anexe um Internet Gateway.** Atualize a tabela de roteamento da subnet pública: `0.0.0.0/0 → igw-xxx`.

4. **Crie um NAT Gateway** em uma subnet pública (requer um Elastic IP). Atualize a tabela de roteamento da subnet privada: `0.0.0.0/0 → nat-xxx`.

5. **Crie security groups:**
   - `alb-sg`: permita inbound 80, 443 de `0.0.0.0/0`.
   - `app-sg`: permita inbound 3000 apenas de `alb-sg`.
   - `db-sg`: permita inbound 5432 apenas de `app-sg`.

6. **Lance uma instância EC2** em uma subnet privada. Verifique que ela pode acessar a internet (via NAT) mas não é acessível diretamente da internet.

7. **Use VPC Flow Logs** para capturar todo o tráfego e armazenar no CloudWatch Logs. Query: encontre todo o tráfego rejeitado para a porta 22.

**Verificação:**
- [ ] EC2 na subnet privada: `curl https://checkip.amazonaws.com` retorna o Elastic IP do NAT Gateway.
- [ ] SSH direto para o IP público da EC2 privada (ela não tem nenhum) é impossível.
- [ ] SSH para a EC2 privada via um bastion host na subnet pública funciona.
- [ ] RDS no `db-sg` rejeita conexões de fora da VPC (teste do seu laptop — deve dar timeout).
- [ ] VPC Flow Logs aparecem no CloudWatch Logs em 10 minutos após atividade de rede.
- [ ] Uma query do CloudWatch Logs Insights para `action = "REJECT" and dstPort = 22` retorna quaisquer tentativas de port scan.

**Nota de custo:** O NAT Gateway custa ~$0,045/hora + transferência de dados. Remova após o lab ou ele acumulará cobranças.

---

## Lab 10 — CI/CD Multi-Ambiente com Terraform e GitHub Actions (Avançado)

**Objetivo:** Construir um pipeline completo de Infrastructure-as-Code + CI/CD que provisiona ambientes separados `staging` e `production` na AWS usando workspaces do Terraform, acionados automaticamente por branches Git.

**Pré-requisitos:**
- Labs 3 (CI/CD com GitHub Actions) e 9 (Terraform) concluídos.
- Conta AWS com permissões para VPC, ECS, RDS, ALB.
- Um repositório GitHub com o código da aplicação e configuração Terraform.

**Tarefas:**

1. **Estruture a configuração Terraform** com workspaces:
   ```
   infra/
     main.tf      # recursos ECS, ALB, RDS, VPC
     variables.tf
     outputs.tf
     backend.tf   # backend S3 + tabela de lock DynamoDB
   ```
   Use `terraform.workspace` para variar tamanhos de instância:
   ```hcl
   instance_class = terraform.workspace == "production" ? "db.t3.small" : "db.t3.micro"
   ```

2. **Pipeline CI/CD** (`.github/workflows/infra.yml`):
   - No push para `main` → `terraform workspace select staging` → `terraform plan` → (após aprovação) `terraform apply`.
   - No push de tag `release` → `terraform workspace select production` → `terraform plan` → exigir aprovação manual (GitHub Environment com reviewer).
   - No PR → apenas `terraform plan`, poste o plano como comentário no PR.

3. **Pipeline da aplicação** (`.github/workflows/deploy.yml`):
   - No push para `main` → construa imagem → faça deploy para o serviço ECS de **staging**.
   - No push de tag `release` → faça deploy para o serviço ECS de **production** após o health check do staging passar.

4. **Configure GitHub Environments:**
   - `staging`: sem aprovações necessárias.
   - `production`: exigir aprovação de 1 reviewer antes do deploy.

5. **Passo de smoke test:** Após cada deploy, execute `curl https://<env-url>/health` com lógica de retry (5 tentativas, espera de 10s). Falhe o pipeline se o health check nunca passar.

6. **Implemente detecção de drift:** Adicione um GitHub Action agendado (cron: diário) que executa `terraform plan` em ambos os workspaces e falha se houver qualquer drift (infraestrutura alterada fora do Terraform).

**Verificação:**
- [ ] Push para `main` faz deploy automaticamente para staging. Nenhuma ação humana necessária.
- [ ] Push de uma tag `v1.0.0` aciona o workflow de produção, que aguarda aprovação do reviewer.
- [ ] Após aprovação, o deploy de produção roda e o smoke test passa.
- [ ] O workflow de detecção de drift detecta uma regra de security group adicionada manualmente e falha (esperado: "1 to add").
- [ ] `terraform workspace list` mostra os workspaces `staging` e `production` com arquivos de state separados.
- [ ] Um smoke test falhando (app retorna 500) faz o pipeline falhar e nenhum tráfego é redirecionado para a versão quebrada.
- [ ] O comentário no PR do GitHub mostra o diff do plano Terraform para mudanças de infraestrutura.

**Nota de custo:** Dois ambientes completos (ECS + RDS + ALB) custam aproximadamente $50–$80/mês. Destrua o `staging` à noite usando um `terraform destroy` agendado ou defina o desired count do ECS para 0.
