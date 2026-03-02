# Cloud Services Map Cheatsheet

> Quick reference — use Ctrl+F to find what you need. Maps equivalent services across AWS, Azure, and GCP.

---

## Compute

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Virtual Machines | EC2 | Virtual Machines | Compute Engine |
| VM Image | AMI | Managed Image / Gallery | Custom Image |
| Spot / Preemptible VMs | EC2 Spot Instances | Spot VMs | Preemptible / Spot VMs |
| Serverless Functions | Lambda | Azure Functions | Cloud Functions |
| Container Instances (serverless) | Fargate | Container Instances (ACI) | Cloud Run |
| Managed Kubernetes | EKS | AKS | GKE |
| App Platform (PaaS) | Elastic Beanstalk | App Service | App Engine |
| Batch Processing | AWS Batch | Azure Batch | Cloud Batch |
| High-Performance Computing | AWS ParallelCluster | Azure CycleCloud | Bare Metal |
| Edge Compute | AWS Wavelength / Outposts | Azure Edge Zones | Google Distributed Cloud |

---

## Storage

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Object Storage | S3 | Blob Storage | Cloud Storage |
| Block Storage (VM disks) | EBS | Managed Disks | Persistent Disk |
| File Storage (NFS/SMB) | EFS / FSx | Azure Files | Filestore |
| Archive / Cold Storage | S3 Glacier | Archive Blob Tier | Archive Storage |
| Local (ephemeral) SSD | EC2 Instance Store | Local SSD | Local SSD |
| Backup | AWS Backup | Azure Backup | Backup and DR |
| Transfer / Migration | Snow Family | Data Box | Transfer Appliance |

---

## Databases — Relational

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Managed PostgreSQL | RDS for PostgreSQL / Aurora PG | Azure Database for PostgreSQL | Cloud SQL (PostgreSQL) |
| Managed MySQL | RDS for MySQL / Aurora MySQL | Azure Database for MySQL | Cloud SQL (MySQL) |
| Managed SQL Server | RDS for SQL Server | Azure SQL Database | Cloud SQL (SQL Server) |
| Serverless Relational | Aurora Serverless v2 | Azure SQL Serverless | Cloud Spanner / AlloyDB |
| Globally Distributed SQL | Aurora Global DB | Azure SQL Hyperscale | Cloud Spanner |
| Data Warehouse | Redshift | Azure Synapse Analytics | BigQuery |

---

## Databases — NoSQL & Caching

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Key-Value / Document | DynamoDB | Cosmos DB | Firestore / Datastore |
| Wide-Column | Keyspaces (Cassandra) | Cosmos DB (Cassandra API) | Bigtable |
| Graph | Neptune | Cosmos DB (Gremlin API) | — |
| In-Memory Cache (Redis) | ElastiCache (Redis) | Azure Cache for Redis | Memorystore (Redis) |
| In-Memory Cache (Memcached) | ElastiCache (Memcached) | — | Memorystore (Memcached) |
| Time-Series | Timestream | Azure Data Explorer | BigQuery / Bigtable |
| Search | OpenSearch Service | Azure AI Search | — (use Elasticsearch on GCE) |

---

## Networking

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Virtual Private Network | VPC | Virtual Network (VNet) | Virtual Private Cloud (VPC) |
| Subnets | Subnets | Subnets | Subnets |
| Load Balancer (L7 / HTTP) | Application Load Balancer (ALB) | Application Gateway | Cloud Load Balancing (HTTP) |
| Load Balancer (L4 / TCP) | Network Load Balancer (NLB) | Azure Load Balancer | Cloud Load Balancing (TCP/UDP) |
| Global Load Balancer / Anycast | AWS Global Accelerator | Azure Front Door | Cloud Load Balancing |
| CDN | CloudFront | Azure CDN / Front Door | Cloud CDN |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| Private DNS | Route 53 Private Zones | Private DNS Zones | Cloud DNS (private) |
| VPN Gateway | AWS VPN | Azure VPN Gateway | Cloud VPN |
| Dedicated Connection | Direct Connect | ExpressRoute | Cloud Interconnect |
| VPC Peering | VPC Peering | VNet Peering | VPC Network Peering |
| Private Endpoint | AWS PrivateLink | Private Endpoint | Private Service Connect |
| Network Firewall | Network Firewall | Azure Firewall | Cloud Armor / Firewall Policies |
| DDoS Protection | AWS Shield | Azure DDoS Protection | Cloud Armor |
| NAT Gateway | NAT Gateway | NAT Gateway | Cloud NAT |
| API Gateway | API Gateway | API Management | Apigee / Cloud Endpoints |

---

## Messaging & Streaming

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Managed Message Queue | SQS | Service Bus (Queue) | Cloud Tasks / Pub/Sub |
| Pub/Sub Messaging | SNS | Service Bus (Topic) | Pub/Sub |
| Event Streaming (Kafka-like) | Kinesis Data Streams / MSK | Event Hubs | Pub/Sub / Dataflow |
| Event Bus / Router | EventBridge | Event Grid | Eventarc |
| Workflow Orchestration | Step Functions | Logic Apps / Durable Functions | Workflows |
| Scheduler | EventBridge Scheduler | Azure Scheduler / Logic Apps | Cloud Scheduler |

---

## Identity & Security

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| IAM — Users & Roles | IAM | Azure AD / Entra ID | Cloud IAM |
| Single Sign-On | AWS IAM Identity Center | Entra ID (Azure AD) | Cloud Identity |
| Secrets Management | Secrets Manager | Key Vault (Secrets) | Secret Manager |
| Key Management (KMS) | KMS | Key Vault (Keys) | Cloud KMS |
| Certificate Management | ACM | Key Vault (Certs) | Certificate Manager |
| Web App Firewall | AWS WAF | Azure WAF | Cloud Armor |
| Security Posture / CSPM | Security Hub + GuardDuty | Microsoft Defender for Cloud | Security Command Center |
| Threat Detection | GuardDuty | Microsoft Sentinel | Security Command Center |
| Vulnerability Scanning | Amazon Inspector | Microsoft Defender for Containers | Container Analysis |
| Compliance Reports | AWS Artifact | Microsoft Service Trust Portal | Compliance Reports |

---

## CI/CD & Developer Tools

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Source Control | CodeCommit | Azure Repos | Cloud Source Repositories |
| CI/CD Pipelines | CodePipeline + CodeBuild | Azure Pipelines | Cloud Build |
| Artifact Registry | CodeArtifact / ECR | Azure Artifacts / ACR | Artifact Registry |
| Infrastructure as Code | CloudFormation | ARM Templates / Bicep | Deployment Manager |
| IaC (multi-cloud) | Terraform / CDK | Terraform / Bicep | Terraform / Config Connector |
| IDE / Cloud Shell | CloudShell | Azure Cloud Shell | Cloud Shell |

---

## Monitoring & Observability

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Metrics | CloudWatch Metrics | Azure Monitor Metrics | Cloud Monitoring |
| Logs | CloudWatch Logs | Log Analytics (Azure Monitor) | Cloud Logging |
| Distributed Tracing | X-Ray | Application Insights | Cloud Trace |
| Application Performance | CloudWatch + X-Ray | Application Insights | Cloud Profiler |
| Dashboards | CloudWatch Dashboards | Azure Monitor Workbooks / Grafana | Cloud Monitoring Dashboards |
| Alerts | CloudWatch Alarms | Azure Monitor Alerts | Cloud Monitoring Alerting |
| Log Analytics | CloudWatch Logs Insights | Log Analytics KQL | Cloud Logging + BigQuery |
| Synthetic Monitoring | CloudWatch Synthetics | Application Insights Availability | Cloud Monitoring Uptime |

---

## Serverless & Functions Comparison

| Feature | Lambda | Azure Functions | Cloud Functions |
|---------|--------|-----------------|-----------------|
| Max duration | 15 min | 10 min (5 min Consumption) | 60 min (gen2) |
| Memory | 128 MB – 10 GB | 128 MB – 14 GB | 128 MB – 32 GB |
| Cold starts | Yes | Yes (Consumption plan) | Yes |
| Triggers | Events, HTTP, Queues, S3, etc. | HTTP, Queue, Blob, Timer, etc. | HTTP, Pub/Sub, Firestore, etc. |
| Pricing model | Per invocation + GB-s | Per invocation + GB-s | Per invocation + GB-s |
| Concurrency control | Reserved + provisioned | Premium / Dedicated plan | Min instances |

---

## Containers Comparison

| Feature | ECS/Fargate | ACI | Cloud Run |
|---------|-------------|-----|-----------|
| Type | Managed containers | Serverless containers | Serverless containers |
| Scale to zero | Yes (Fargate) | Yes | Yes |
| VPC integration | Yes | Yes | Yes |
| GPU support | Yes (EC2 launch type) | Yes | Yes (gen2) |
| Managed Kubernetes | EKS | AKS | GKE |
| Kubernetes version SLA | Yes | Yes | Yes (Autopilot) |

---

## AI / ML Services

| Category | AWS | Azure | GCP |
|----------|-----|-------|-----|
| ML Platform | SageMaker | Azure Machine Learning | Vertex AI |
| Managed LLM APIs | Amazon Bedrock | Azure OpenAI Service | Vertex AI (Gemini) |
| Vision API | Rekognition | Computer Vision | Cloud Vision AI |
| Speech-to-Text | Transcribe | Speech Service | Speech-to-Text |
| Text-to-Speech | Polly | Speech Service | Text-to-Speech |
| NLP / Language | Comprehend | Language Service | Natural Language AI |
| Translation | Translate | Translator | Cloud Translation |
| Document OCR | Textract | Form Recognizer / Document Intelligence | Document AI |
| Recommendation | Personalize | Personalizer | Recommendations AI |

---

## Pricing Model Patterns

| Model | When to Use | Example |
|-------|-------------|---------|
| On-Demand | Variable workloads, dev/test | EC2 On-Demand |
| Reserved (1-3yr) | Steady, predictable base load (save 30-70%) | EC2 Reserved, Azure Reserved VM |
| Spot / Preemptible | Fault-tolerant batch, ML training (save 60-90%) | EC2 Spot, GCP Spot VM |
| Savings Plans | Flexible commitment (vs. Reserved) | AWS Compute Savings Plans |
| Serverless | Low-traffic or spiky workloads | Lambda, Cloud Run |
| Committed Use | GCP equivalent of Reserved | CUDs for Compute Engine |

---

## Global Regions Quick Facts (as of 2025)

| Provider | Regions | Availability Zones |
|----------|---------|--------------------|
| AWS | 34+ | 108+ |
| Azure | 60+ | Multiple per region |
| GCP | 40+ | 121+ |

**Choosing a region:**
- Latency: pick closest to users
- Compliance: data residency requirements (GDPR → EU regions)
- Availability: pick regions with 3+ AZs for HA
- Cost: regions vary significantly in price (us-east-1 typically cheapest on AWS)
