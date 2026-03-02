# Mapa de Serviços Cloud Cheatsheet

> Referência rápida — use Ctrl+F para encontrar o que precisa. Mapeia serviços equivalentes entre AWS, Azure e GCP.

---

## Computação

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Máquinas Virtuais | EC2 | Virtual Machines | Compute Engine |
| Imagem de VM | AMI | Managed Image / Gallery | Custom Image |
| VMs Spot / Preemptíveis | EC2 Spot Instances | Spot VMs | Preemptible / Spot VMs |
| Funções Serverless | Lambda | Azure Functions | Cloud Functions |
| Containers Serverless | Fargate | Container Instances (ACI) | Cloud Run |
| Kubernetes Gerenciado | EKS | AKS | GKE |
| Plataforma de Aplicação (PaaS) | Elastic Beanstalk | App Service | App Engine |
| Processamento em Lote | AWS Batch | Azure Batch | Cloud Batch |
| Computação de Alto Desempenho | AWS ParallelCluster | Azure CycleCloud | Bare Metal |
| Computação na Borda | AWS Wavelength / Outposts | Azure Edge Zones | Google Distributed Cloud |

---

## Armazenamento

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Armazenamento de Objetos | S3 | Blob Storage | Cloud Storage |
| Armazenamento em Bloco (discos de VM) | EBS | Managed Disks | Persistent Disk |
| Armazenamento de Arquivos (NFS/SMB) | EFS / FSx | Azure Files | Filestore |
| Armazenamento Frio / Arquivamento | S3 Glacier | Archive Blob Tier | Archive Storage |
| SSD Local (efêmero) | EC2 Instance Store | Local SSD | Local SSD |
| Backup | AWS Backup | Azure Backup | Backup and DR |
| Transferência / Migração | Snow Family | Data Box | Transfer Appliance |

---

## Bancos de Dados — Relacionais

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| PostgreSQL Gerenciado | RDS for PostgreSQL / Aurora PG | Azure Database for PostgreSQL | Cloud SQL (PostgreSQL) |
| MySQL Gerenciado | RDS for MySQL / Aurora MySQL | Azure Database for MySQL | Cloud SQL (MySQL) |
| SQL Server Gerenciado | RDS for SQL Server | Azure SQL Database | Cloud SQL (SQL Server) |
| Relacional Serverless | Aurora Serverless v2 | Azure SQL Serverless | Cloud Spanner / AlloyDB |
| SQL Distribuído Globalmente | Aurora Global DB | Azure SQL Hyperscale | Cloud Spanner |
| Data Warehouse | Redshift | Azure Synapse Analytics | BigQuery |

---

## Bancos de Dados — NoSQL e Cache

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Chave-Valor / Documentos | DynamoDB | Cosmos DB | Firestore / Datastore |
| Colunar | Keyspaces (Cassandra) | Cosmos DB (Cassandra API) | Bigtable |
| Grafos | Neptune | Cosmos DB (Gremlin API) | — |
| Cache em Memória (Redis) | ElastiCache (Redis) | Azure Cache for Redis | Memorystore (Redis) |
| Cache em Memória (Memcached) | ElastiCache (Memcached) | — | Memorystore (Memcached) |
| Séries Temporais | Timestream | Azure Data Explorer | BigQuery / Bigtable |
| Busca | OpenSearch Service | Azure AI Search | — (use Elasticsearch no GCE) |

---

## Redes

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Rede Privada Virtual | VPC | Virtual Network (VNet) | Virtual Private Cloud (VPC) |
| Sub-redes | Subnets | Subnets | Subnets |
| Load Balancer (L7 / HTTP) | Application Load Balancer (ALB) | Application Gateway | Cloud Load Balancing (HTTP) |
| Load Balancer (L4 / TCP) | Network Load Balancer (NLB) | Azure Load Balancer | Cloud Load Balancing (TCP/UDP) |
| Load Balancer Global / Anycast | AWS Global Accelerator | Azure Front Door | Cloud Load Balancing |
| CDN | CloudFront | Azure CDN / Front Door | Cloud CDN |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| DNS Privado | Route 53 Private Zones | Private DNS Zones | Cloud DNS (privado) |
| Gateway VPN | AWS VPN | Azure VPN Gateway | Cloud VPN |
| Conexão Dedicada | Direct Connect | ExpressRoute | Cloud Interconnect |
| Peering de VPC | VPC Peering | VNet Peering | VPC Network Peering |
| Endpoint Privado | AWS PrivateLink | Private Endpoint | Private Service Connect |
| Firewall de Rede | Network Firewall | Azure Firewall | Cloud Armor / Firewall Policies |
| Proteção contra DDoS | AWS Shield | Azure DDoS Protection | Cloud Armor |
| NAT Gateway | NAT Gateway | NAT Gateway | Cloud NAT |
| API Gateway | API Gateway | API Management | Apigee / Cloud Endpoints |

---

## Mensageria e Streaming

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Fila de Mensagens Gerenciada | SQS | Service Bus (Queue) | Cloud Tasks / Pub/Sub |
| Mensageria Pub/Sub | SNS | Service Bus (Topic) | Pub/Sub |
| Streaming de Eventos (estilo Kafka) | Kinesis Data Streams / MSK | Event Hubs | Pub/Sub / Dataflow |
| Barramento de Eventos / Roteador | EventBridge | Event Grid | Eventarc |
| Orquestração de Workflows | Step Functions | Logic Apps / Durable Functions | Workflows |
| Agendador | EventBridge Scheduler | Azure Scheduler / Logic Apps | Cloud Scheduler |

---

## Identidade e Segurança

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| IAM — Usuários e Roles | IAM | Azure AD / Entra ID | Cloud IAM |
| Single Sign-On | AWS IAM Identity Center | Entra ID (Azure AD) | Cloud Identity |
| Gerenciamento de Secrets | Secrets Manager | Key Vault (Secrets) | Secret Manager |
| Gerenciamento de Chaves (KMS) | KMS | Key Vault (Keys) | Cloud KMS |
| Gerenciamento de Certificados | ACM | Key Vault (Certs) | Certificate Manager |
| Firewall de Aplicação Web | AWS WAF | Azure WAF | Cloud Armor |
| Postura de Segurança / CSPM | Security Hub + GuardDuty | Microsoft Defender for Cloud | Security Command Center |
| Detecção de Ameaças | GuardDuty | Microsoft Sentinel | Security Command Center |
| Varredura de Vulnerabilidades | Amazon Inspector | Microsoft Defender for Containers | Container Analysis |
| Relatórios de Conformidade | AWS Artifact | Microsoft Service Trust Portal | Compliance Reports |

---

## CI/CD e Ferramentas para Desenvolvedores

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Controle de Versão | CodeCommit | Azure Repos | Cloud Source Repositories |
| Pipelines de CI/CD | CodePipeline + CodeBuild | Azure Pipelines | Cloud Build |
| Registry de Artefatos | CodeArtifact / ECR | Azure Artifacts / ACR | Artifact Registry |
| Infraestrutura como Código | CloudFormation | ARM Templates / Bicep | Deployment Manager |
| IaC (multi-cloud) | Terraform / CDK | Terraform / Bicep | Terraform / Config Connector |
| IDE / Cloud Shell | CloudShell | Azure Cloud Shell | Cloud Shell |

---

## Monitoramento e Observabilidade

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Métricas | CloudWatch Metrics | Azure Monitor Metrics | Cloud Monitoring |
| Logs | CloudWatch Logs | Log Analytics (Azure Monitor) | Cloud Logging |
| Rastreamento Distribuído | X-Ray | Application Insights | Cloud Trace |
| Performance de Aplicação | CloudWatch + X-Ray | Application Insights | Cloud Profiler |
| Dashboards | CloudWatch Dashboards | Azure Monitor Workbooks / Grafana | Cloud Monitoring Dashboards |
| Alertas | CloudWatch Alarms | Azure Monitor Alerts | Cloud Monitoring Alerting |
| Análise de Logs | CloudWatch Logs Insights | Log Analytics KQL | Cloud Logging + BigQuery |
| Monitoramento Sintético | CloudWatch Synthetics | Application Insights Availability | Cloud Monitoring Uptime |

---

## Comparativo de Serverless e Functions

| Funcionalidade | Lambda | Azure Functions | Cloud Functions |
|----------------|--------|-----------------|-----------------|
| Duração máxima | 15 min | 10 min (5 min no plano Consumption) | 60 min (gen2) |
| Memória | 128 MB – 10 GB | 128 MB – 14 GB | 128 MB – 32 GB |
| Cold starts | Sim | Sim (plano Consumption) | Sim |
| Triggers | Events, HTTP, Queues, S3, etc. | HTTP, Queue, Blob, Timer, etc. | HTTP, Pub/Sub, Firestore, etc. |
| Modelo de preço | Por invocação + GB-s | Por invocação + GB-s | Por invocação + GB-s |
| Controle de concorrência | Reserved + provisioned | Plano Premium / Dedicated | Instâncias mínimas |

---

## Comparativo de Containers

| Funcionalidade | ECS/Fargate | ACI | Cloud Run |
|----------------|-------------|-----|-----------|
| Tipo | Containers gerenciados | Containers serverless | Containers serverless |
| Escalar para zero | Sim (Fargate) | Sim | Sim |
| Integração com VPC | Sim | Sim | Sim |
| Suporte a GPU | Sim (launch type EC2) | Sim | Sim (gen2) |
| Kubernetes Gerenciado | EKS | AKS | GKE |
| SLA de versão do Kubernetes | Sim | Sim | Sim (Autopilot) |

---

## Serviços de IA / ML

| Categoria | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Plataforma de ML | SageMaker | Azure Machine Learning | Vertex AI |
| APIs de LLM Gerenciadas | Amazon Bedrock | Azure OpenAI Service | Vertex AI (Gemini) |
| API de Visão Computacional | Rekognition | Computer Vision | Cloud Vision AI |
| Speech-to-Text | Transcribe | Speech Service | Speech-to-Text |
| Text-to-Speech | Polly | Speech Service | Text-to-Speech |
| NLP / Linguagem | Comprehend | Language Service | Natural Language AI |
| Tradução | Translate | Translator | Cloud Translation |
| OCR de Documentos | Textract | Form Recognizer / Document Intelligence | Document AI |
| Recomendação | Personalize | Personalizer | Recommendations AI |

---

## Modelos de Precificação

| Modelo | Quando Usar | Exemplo |
|--------|-------------|---------|
| Sob Demanda | Cargas variáveis, dev/teste | EC2 On-Demand |
| Reservado (1-3 anos) | Carga base constante e previsível (economize 30-70%) | EC2 Reserved, Azure Reserved VM |
| Spot / Preemptível | Batch tolerante a falhas, treinamento de ML (economize 60-90%) | EC2 Spot, GCP Spot VM |
| Savings Plans | Compromisso flexível (vs. Reservado) | AWS Compute Savings Plans |
| Serverless | Cargas com baixo tráfego ou picos | Lambda, Cloud Run |
| Uso Comprometido | Equivalente GCP do Reservado | CUDs para Compute Engine |

---

## Dados Rápidos sobre Regiões Globais (em 2025)

| Provedor | Regiões | Zonas de Disponibilidade |
|----------|---------|--------------------------|
| AWS | 34+ | 108+ |
| Azure | 60+ | Múltiplas por região |
| GCP | 40+ | 121+ |

**Como escolher uma região:**
- Latência: escolha a mais próxima dos usuários
- Conformidade: requisitos de residência de dados (LGPD/GDPR → regiões locais)
- Disponibilidade: prefira regiões com 3+ AZs para alta disponibilidade
- Custo: os preços variam significativamente por região (us-east-1 costuma ser a mais barata na AWS)
