# Cloud Comparison

## Overview

AWS, Azure, and GCP each solve the same problems differently. Choosing the wrong provider for your use case doesn't doom a project, but it creates friction — unfamiliar tooling, pricing surprises, and services that don't quite match the pattern you need.

This chapter gives you a practical basis for choosing a provider and for working across providers when you encounter them in different jobs or projects. The comparisons are anchored in real Node.js/TypeScript backend development: deploying APIs, storing data, managing secrets, and operating in production.

---

## Prerequisites

- AWS Core Services (`aws-core-services.md`)
- Azure Core Services (`azure-core-services.md`)
- GCP Core Services (`gcp-core-services.md`)

---

## Core Concepts

### Market Position (as of 2025)

```
AWS:   ~31% market share — broadest service catalog, most job listings
Azure: ~25% market share — dominant in enterprise/Microsoft shops
GCP:   ~12% market share — strongest for data/ML, Kubernetes, startups
```

None of them is objectively "better." They are meaningfully different in philosophy, pricing, and ecosystem. Your context drives the choice.

### Philosophy Differences

**AWS:** Explicit configuration. You control everything, but you must configure everything. Security is opt-in (open security groups don't warn you, they just allow traffic). Largest service catalog — there's an AWS service for almost any use case.

**Azure:** Integration-first. Deep integration with Microsoft ecosystem (Active Directory, Office 365, .NET, Windows Server). Managed Identity is the authentication superpower — no credentials anywhere. Bicep and ARM templates for IaC.

**GCP:** Developer experience and data. Application Default Credentials eliminate credential management. Global networking infrastructure is genuinely exceptional. Best-in-class Kubernetes (GKE), BigQuery for analytics, and Vertex AI for ML.

---

## Service Mapping

### Compute

| Use Case | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Managed containers (serverless) | ECS Fargate / App Runner | Container Apps | Cloud Run |
| Platform-as-a-Service | Elastic Beanstalk | App Service | App Engine (legacy) |
| Serverless functions | Lambda | Azure Functions | Cloud Functions (2nd gen) |
| Managed Kubernetes | EKS | AKS | GKE Autopilot |
| Virtual machines | EC2 | Azure VMs | Compute Engine |
| Batch / one-shot containers | ECS Fargate tasks | Container Apps Jobs | Cloud Run Jobs |

**Recommendation for most Node.js APIs:**
- AWS: ECS Fargate with an ALB
- Azure: Container Apps
- GCP: Cloud Run

All three approaches run Docker containers with auto-scaling. The main differences are operational model and pricing.

### Storage

| Use Case | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Object storage | S3 | Blob Storage | Cloud Storage |
| Block storage (VM disks) | EBS | Managed Disks | Persistent Disk |
| File storage (NFS) | EFS | Azure Files | Filestore |
| CDN | CloudFront | Azure Front Door / CDN | Cloud CDN |

**Object storage pricing comparison (approximate, us-east-1/us):**
| Provider | Storage (per GB/mo) | Egress (per GB) | PUT operations (per 1M) |
|----------|---------------------|-----------------|------------------------|
| AWS S3 Standard | $0.023 | $0.09 | $5.00 |
| Azure Blob (Hot) | $0.018 | $0.087 | $5.00 |
| GCP Cloud Storage | $0.020 | $0.12 | $5.00 |

AWS egress is notably cheaper for large data transfers. Azure has slightly lower storage rates. GCP has higher egress but is competitive for most workloads.

### Databases

| Use Case | AWS | Azure | GCP |
|----------|-----|-------|-----|
| Managed PostgreSQL | RDS PostgreSQL / Aurora | Azure Database for PostgreSQL Flexible | Cloud SQL PostgreSQL |
| Managed MySQL | RDS MySQL / Aurora | Azure Database for MySQL Flexible | Cloud SQL MySQL |
| NoSQL document | DynamoDB | Cosmos DB | Firestore |
| In-memory cache | ElastiCache (Redis/Memcached) | Azure Cache for Redis | Memorystore (Redis/Memcached) |
| Analytics / data warehouse | Redshift | Azure Synapse | BigQuery |
| Global distributed SQL | Aurora Global | Cosmos DB (multi-master) | Spanner |

**Managed PostgreSQL comparison:**
- **AWS RDS / Aurora:** Most mature. Aurora is faster (3-5x vs standard MySQL, significant vs Postgres) and expensive. Aurora Serverless v2 autoscales compute.
- **Azure Database for PostgreSQL Flexible:** Zone Redundant HA is solid. Good price/performance on General Purpose tier.
- **GCP Cloud SQL:** Connects via Auth Proxy (best developer experience for auth). Regional HA included. Slightly less performant than the others at same price point but ADC integration is seamless.

### Security and Identity

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Identity and access management | IAM | Azure AD / Entra ID | IAM |
| Service identity (no credentials) | IAM Roles | Managed Identity | Service Accounts + ADC |
| Secret storage | Secrets Manager | Key Vault | Secret Manager |
| Encryption keys | KMS | Key Vault | Cloud KMS |
| Web application firewall | WAF | Application Gateway WAF / Front Door WAF | Cloud Armor |
| DDoS protection | Shield | DDoS Protection | Cloud Armor |

**Identity philosophy:**
- **AWS IAM Roles:** Attach to EC2/ECS/Lambda. SDK discovers credentials via metadata service. Works well; configuration is more explicit.
- **Azure Managed Identity:** The gold standard. Zero credential management. `DefaultAzureCredential` discovers identity automatically in any Azure context. Works with Key Vault, Blob Storage, Service Bus — everything.
- **GCP ADC (Application Default Credentials):** Equivalent to Azure's approach. `gcloud auth application-default login` for local dev; service account attached to Cloud Run/GKE for production. Zero credentials in code or environment variables.

### Networking

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Private network | VPC | Virtual Network (VNet) | VPC |
| Load balancer (HTTP/L7) | Application Load Balancer (ALB) | Application Gateway | Cloud Load Balancing |
| Global load balancer | ALB + Route 53 | Azure Front Door | Cloud Load Balancing (global by default) |
| DNS | Route 53 | Azure DNS | Cloud DNS |
| Private DNS | Route 53 Resolver | Azure Private DNS | Cloud DNS (private zones) |
| VPN | Site-to-Site VPN | VPN Gateway | Cloud VPN |
| Private connectivity | Direct Connect | ExpressRoute | Cloud Interconnect |

**GCP's global load balancer is genuinely different:** AWS and Azure's load balancers are regional by default (you compose multiple regional LBs for global routing). GCP's Cloud Load Balancing is global by design — a single IP address, anycast routing, requests land at the nearest Google point of presence.

### Operations

| Concept | AWS | Azure | GCP |
|---------|-----|-------|-----|
| Metrics and monitoring | CloudWatch | Azure Monitor | Cloud Monitoring |
| Logging | CloudWatch Logs | Log Analytics / Azure Monitor Logs | Cloud Logging |
| Application performance monitoring | CloudWatch + X-Ray | Application Insights | Cloud Trace + Error Reporting |
| Infrastructure as Code | CloudFormation / CDK | ARM Templates / Bicep | Deployment Manager / Terraform |
| Managed CI/CD | CodePipeline + CodeBuild | Azure DevOps / GitHub Actions | Cloud Build + Cloud Deploy |
| Artifact registry | ECR | Azure Container Registry | Artifact Registry |

**IaC recommendation:**
- If using only one cloud: CDK (AWS), Bicep (Azure), Deployment Manager (GCP) — native tools with better first-class support
- If multi-cloud or already know it: Terraform works across all three

---

## Hands-On Examples

### Deploying the Same Node.js API to All Three Clouds

Assume a Node.js/TypeScript API in a Docker container, listening on port 3000, connecting to PostgreSQL.

**AWS (ECS Fargate):**
```bash
# Push to ECR
aws ecr get-login-password | docker login --username AWS --password-stdin $ECR_URI
docker push $ECR_URI/myapp:latest

# Update ECS service (assumes cluster and service already exist)
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --force-new-deployment

# Check rollout
aws ecs wait services-stable --cluster myapp-cluster --services myapp-api
```

**Azure (Container Apps):**
```bash
# Build in ACR
az acr build --registry myregistry --image myapp:$SHA .

# Update container app
az containerapp update \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --image myregistry.azurecr.io/myapp:$SHA
```

**GCP (Cloud Run):**
```bash
# Build with Cloud Build
gcloud builds submit --tag us-central1-docker.pkg.dev/myapp-prod/myapp/api:$SHA .

# Deploy
gcloud run deploy myapp-api \
  --image us-central1-docker.pkg.dev/myapp-prod/myapp/api:$SHA \
  --region us-central1
```

All three approaches deploy a container. The main differences are image registry (ECR, ACR, Artifact Registry) and the update mechanism.

### Reading a Secret in Node.js

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

All three use the cloud SDK with credentials auto-discovered from the environment. The pattern is identical in spirit.

---

## Common Patterns & Best Practices

### When to Choose AWS

- Team already has AWS expertise or certifications
- Need the widest service catalog (Kinesis, SageMaker, Rekognition, etc.)
- US-focused startup hiring developers — AWS is the default expectation
- Need the most mature managed Kubernetes (EKS is more battle-tested for large clusters than AKS or GKE in some workloads, though GKE is catching up)
- Heavy use of serverless: Lambda has the most mature ecosystem (layers, Lambda@Edge, custom runtimes)

### When to Choose Azure

- Organization is Microsoft-heavy: Active Directory, Office 365, Windows Server, .NET
- Enterprise compliance requirements: Azure has the most government and industry certifications
- Hybrid cloud: Azure Arc is the best story for on-premises + cloud workloads
- Using GitHub (Microsoft-owned): GitHub Actions + Azure integration is seamless with OIDC
- SQL Server workloads: Azure SQL is the obvious home

### When to Choose GCP

- Data and ML workloads: BigQuery, Vertex AI, Dataflow are industry-leading
- Strong Kubernetes usage: GKE invented Kubernetes; it's the most mature managed K8s
- Startup that values developer experience: ADC + Cloud Run + Cloud Build is the smoothest DX
- Google Workspace integration (Gmail, Drive, Sheets)
- Need best-in-class global network: GCP's private backbone genuinely reduces latency for global users

### When Provider Choice Doesn't Matter Much

For a typical Node.js SaaS (API + PostgreSQL + Redis + object storage + CDN), all three providers deliver identical outcomes at similar costs. The differentiator is team familiarity and ecosystem fit, not technical capability.

---

## Anti-Patterns to Avoid

### Multi-Cloud by Default

```
BAD: "We should use multiple clouds for vendor lock-in avoidance."

Reality:
- Multi-cloud increases operational complexity by 3-5x
- Each cloud has different networking, IAM, billing, and tooling
- Very few teams have the expertise to operate multiple clouds well
- Abstraction layers (Terraform, Crossplane) help but don't eliminate complexity

GOOD: Choose one cloud for your primary workload.
      Use a second cloud only when there's a strong technical reason
      (e.g., GCP BigQuery for analytics + AWS for the application).
```

### Ignoring Egress Costs

All three providers charge for data leaving their network:
- AWS: $0.09/GB after first 100GB/mo (free tier)
- Azure: $0.087/GB
- GCP: $0.12/GB (to internet)

For applications transferring terabytes of data monthly, egress costs can exceed compute costs. Measure before assuming it's negligible.

### Assuming "Managed" Means "Zero Ops"

Managed services reduce ops burden but don't eliminate it:
- RDS/Cloud SQL still needs storage monitoring and parameter tuning
- ElastiCache/Redis still needs memory monitoring and eviction policy
- EKS/GKE still needs node pool management and upgrade scheduling

"Managed" means the provider handles OS patching and hardware. You still own configuration, capacity, and performance.

---

## Debugging & Troubleshooting

### Provider-Specific Tooling Reference

**AWS:**
```bash
# Service health
aws health describe-events --filter eventStatusCodes=open

# Check service quotas
aws service-quotas list-service-quotas --service-code ecs

# Cost spike investigation
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE
```

**Azure:**
```bash
# Service health
az resource show --ids /subscriptions/SUB/providers/Microsoft.ResourceHealth/...

# Activity log (audit trail)
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --start-time 2024-01-01 \
  --output table

# Cost by service
az consumption usage list \
  --start-date 2024-01-01 --end-date 2024-01-31 \
  --query "[].{Service:instanceName,Cost:pretaxCost}" \
  --output table
```

**GCP:**
```bash
# Service health
gcloud alpha monitoring services list

# Audit logs
gcloud logging read \
  'logName="projects/myapp-prod/logs/cloudaudit.googleapis.com%2Factivity"' \
  --limit 50

# Cost export (requires BigQuery billing export enabled)
# Query in BigQuery console:
# SELECT service.description, SUM(cost) as total
# FROM `myapp-billing.billing_export.gcp_billing_export_v1_*`
# WHERE DATE(usage_start_time) BETWEEN '2024-01-01' AND '2024-01-31'
# GROUP BY 1 ORDER BY 2 DESC
```

---

## Real-World Scenarios

### Scenario 1: Startup Choosing Their First Cloud

```
Context:
  - 3 engineers, all have some AWS experience
  - Node.js API + React SPA + PostgreSQL
  - Budget: $200/month initially
  - Plan to raise and scale within 18 months

Recommendation: AWS

Reasons:
  - Team familiarity → fastest to production
  - Most job candidates have AWS experience
  - Free tier is generous for early-stage
  - ECS Fargate + RDS + S3 is well-documented

Cost estimate:
  ECS Fargate (0.5 vCPU, 1GB): ~$25/mo
  RDS db.t3.micro PostgreSQL: ~$13/mo
  S3 + CloudFront: ~$5/mo
  Route 53: ~$1/mo
  Total: ~$44/mo
```

### Scenario 2: Enterprise Moving from On-Premises

```
Context:
  - 500-person company, heavy Active Directory usage
  - Compliance requirements (SOC 2, ISO 27001)
  - IT team familiar with Windows Server and SQL Server
  - Mixed .NET and Node.js workloads

Recommendation: Azure

Reasons:
  - Azure AD integrates directly with on-premises AD (Azure AD Connect)
  - Azure has the most compliance certifications
  - Azure SQL is the natural home for SQL Server workloads
  - Managed Identity eliminates credential management across the estate
  - Azure DevOps + GitHub Actions + Azure is the best end-to-end story
```

### Scenario 3: Data-Heavy Analytical Platform

```
Context:
  - Processing 50GB+ of events per day
  - ML model training and serving
  - Analytics dashboards for customers

Recommendation: GCP

Reasons:
  - BigQuery is genuinely the best managed data warehouse
  - Vertex AI is strong for ML training and serving
  - Dataflow for streaming data processing
  - Cloud Run for the API layer
  - Competitive pricing for data transfer within GCP (BigQuery → Cloud Run is free)
```

---

## Further Reading

- [AWS vs Azure vs GCP pricing calculator comparison](https://cloudprice.net/)
- [Cloud Provider SLA comparison](https://uptime.is/)
- [Google Cloud Architecture Framework](https://cloud.google.com/architecture/framework)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Azure Well-Architected Framework](https://learn.microsoft.com/azure/well-architected/)
- [CNCF Landscape — cloud-native tooling across providers](https://landscape.cncf.io/)

---

## Summary

| Dimension | AWS | Azure | GCP |
|-----------|-----|-------|-----|
| Market share | Largest | Second | Third |
| Breadth of services | Most | Very broad | Focused |
| Developer experience | Good | Good | Excellent |
| Enterprise/compliance | Strong | Best | Growing |
| Kubernetes | EKS (solid) | AKS (good) | GKE (best) |
| Serverless containers | ECS Fargate | Container Apps | Cloud Run |
| Credential management | IAM Roles | Managed Identity | ADC + SA |
| Data/ML | SageMaker | Azure ML | BigQuery + Vertex AI |
| Global networking | Good | Good | Excellent |
| Free tier | Generous | Good | Good |
| Best fit | Most use cases, largest talent pool | Microsoft shops, enterprise | Data/ML, K8s, strong DX |

There is no universally correct choice. Choose the provider your team knows, then invest in learning it deeply. Shallow knowledge of multiple clouds is worse than deep knowledge of one.
