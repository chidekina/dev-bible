# Cloud Cost Optimization

## Overview

Cloud costs have a way of growing quietly and then shocking you. A misconfigured Lambda, an S3 bucket collecting logs forever, a database with 4x the provisioned IOPS it needs — these accumulate into bills that dwarf your original estimates.

Cost optimization is not about being cheap. It is about paying for what you actually use and not paying for what you don't. The techniques in this chapter apply across AWS, Azure, and GCP and are grounded in real-world patterns for Node.js backend deployments.

---

## Prerequisites

- AWS Core Services (`aws-core-services.md`) or equivalent Azure/GCP chapter
- Understanding of your application's traffic patterns and resource usage

---

## Core Concepts

### The FinOps Mental Model

Cloud cost optimization follows a framework called FinOps:

```
Inform → Optimize → Operate

Inform:   Understand what you're spending and why
Optimize: Apply changes to reduce waste
Operate:  Continuously monitor and repeat
```

The biggest mistake teams make is skipping "Inform" and jumping straight to optimization. Without visibility, you optimize the wrong things.

### Where Cloud Bills Come From

For a typical Node.js SaaS, the major cost categories are:

```
1. Compute (EC2, ECS, App Service, Cloud Run)     40-60% of bill
2. Database (RDS, Cloud SQL, Cosmos DB)           20-35%
3. Data transfer / egress                         5-20%
4. Storage (S3, Blob, Cloud Storage)              2-10%
5. Networking (Load balancers, NAT, VPN)          3-10%
6. Managed services (ElastiCache, SQS, etc.)      2-10%
```

Start with compute and databases — that's where 70%+ of most bills live.

### Unit Economics

The right metric is not total cost but cost per unit of value:

```
Cost per active user per month
Cost per API request
Cost per GB processed
Cost per transaction
```

A $10,000/month bill is fine if you're serving 100,000 paying customers. A $500/month bill is a problem if you have 10 free users.

---

## Hands-On Examples

### Step 1: Get Visibility First

**AWS:**
```bash
# Cost by service for last 30 days
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[0].Groups[*].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}' \
  --output table

# Cost by tag (requires tagging all resources)
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=TAG,Key=Environment

# Enable Cost Explorer (do this today if you haven't)
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Environment,Status=Active
```

**Azure:**
```bash
# Cost by resource group
az consumption usage list \
  --start-date 2024-01-01 \
  --end-date 2024-01-31 \
  --query "sort_by([].{Name:instanceName,Cost:pretaxCost,Service:product}, &Cost)" \
  --output table

# Set up cost alerts before doing anything else
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
# Enable billing export to BigQuery (one-time setup)
# Then query:
# SELECT service.description, SUM(cost) as total_cost
# FROM `project.dataset.gcp_billing_export_v1_*`
# WHERE DATE(usage_start_time) >= DATE_SUB(CURRENT_DATE(), INTERVAL 30 DAY)
# GROUP BY 1 ORDER BY 2 DESC

# Recommendations
gcloud recommender recommendations list \
  --recommender google.compute.instance.MachineTypeRecommender \
  --location us-central1 \
  --project myapp-prod
```

### Step 2: Tag Everything

Tags/labels are the foundation of cost attribution. Without them, you cannot answer "how much does the API service cost?" or "what does the staging environment cost?"

**AWS tagging strategy:**
```bash
# Tag all resources consistently
aws resourcegroupstaggingapi tag-resources \
  --resource-arn-list arn:aws:ecs:us-east-1:123456789012:service/myapp-cluster/myapp-api \
  --tags \
    Project=myapp \
    Environment=production \
    Team=backend \
    Service=api \
    CostCenter=engineering

# Enable tag-based cost allocation
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status \
    TagKey=Project,Status=Active \
    TagKey=Environment,Status=Active \
    TagKey=Team,Status=Active

# Find untagged resources
aws resourcegroupstaggingapi get-resources \
  --tag-filters '[]' \
  --query 'ResourceTagMappingList[?Tags==`[]`].ResourceARN' \
  --output text
```

**Terraform: enforce tags by default:**
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

### Step 3: Right-Size Compute

Running oversized instances is one of the most common sources of waste.

**AWS — use Compute Optimizer:**
```bash
# Get recommendations for EC2 instances
aws compute-optimizer get-ec2-instance-recommendations \
  --query 'instanceRecommendations[].{
    Instance: instanceArn,
    CurrentType: currentInstanceType,
    Finding: finding,
    RecommendedType: recommendationOptions[0].instanceType,
    MonthlySavings: recommendationOptions[0].estimatedMonthlySavings.value
  }' \
  --output table

# Get recommendations for ECS services
aws compute-optimizer get-ecs-service-recommendations \
  --query 'ecsServiceRecommendations[].{
    Service: serviceArn,
    Finding: finding,
    CurrentCPU: currentServiceConfiguration.cpu,
    RecommendedCPU: serviceRecommendationOptions[0].containerRecommendations[0].cpu[0].upperBound,
    MonthlySavings: serviceRecommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value
  }'
```

**Practical right-sizing for Node.js:**
```
Node.js is single-threaded → rarely benefits from >2 vCPU per process
                             → scale out (more instances) not up (bigger instances)

ECS Fargate starting point:
  API (CRUD, low compute):    0.25-0.5 vCPU, 512MB-1GB memory
  API (heavy processing):     1-2 vCPU, 1-2GB memory

Signals you're over-provisioned:
  - CPU p95 < 20% consistently
  - Memory usage < 30% consistently
  - Scale events never trigger

Signals you're under-provisioned:
  - CPU p95 > 80%
  - OOM kills in logs
  - Frequent scale-out events, tasks never scale in
```

### Step 4: Purchase Reserved Capacity

After 2-3 months of stable production usage, reserved capacity saves 30-60% on compute.

**AWS Savings Plans (flexible, recommended over Reserved Instances):**
```bash
# Check your hourly on-demand spend to determine commitment amount
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-08 \
  --granularity DAILY \
  --metrics UsageQuantity UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Elastic Compute Cloud - Compute"]}}'

# Purchase via console (Compute Savings Plans covers EC2, Fargate, Lambda)
# Typical savings:
#   1-year, no upfront: 27% savings
#   1-year, all upfront: 40% savings
#   3-year, all upfront: 60% savings
```

**Azure Reserved VM Instances:**
```bash
# Buy via portal: Cost Management + Billing → Reservations
# Or via CLI:
az reservations reservation-order purchase \
  --reservation-order-id ORDER_ID \
  --sku Standard_D2s_v3 \
  --location eastus \
  --reserved-resource-type VirtualMachines \
  --term P1Y \                # 1 year
  --billing-scope /subscriptions/SUB \
  --quantity 2 \
  --applied-scope-type Shared
```

**GCP Committed Use Discounts:**
```bash
# 1-year commitment on Cloud Run CPU
gcloud compute commitments create myapp-commitment \
  --plan 12-month \
  --resources vcpu=4,memory=16GB \
  --region us-central1

# Typical Cloud Run savings:
#   On-demand:    $0.00002400/vCPU-second
#   1-yr commit:  $0.00001720/vCPU-second (28% savings)
#   3-yr commit:  $0.00001370/vCPU-second (43% savings)
```

### Step 5: Optimize Storage

**S3 / Cloud Storage lifecycle policies:**
```bash
# AWS — lifecycle rule: move to IA after 30 days, Glacier after 90
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

# Delete incomplete multipart uploads (they accumulate silently)
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

# GCP — lifecycle config
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

**Log retention:**
```bash
# AWS CloudWatch Logs: set retention on all log groups
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].logGroupName' \
  --output text | while read -r lg; do
    aws logs put-retention-policy \
      --log-group-name "$lg" \
      --retention-in-days 30
done

# Azure: Log Analytics workspace retention
az monitor log-analytics workspace update \
  --workspace-name myapp-workspace \
  --resource-group myapp-prod-rg \
  --retention-time 30
```

### Step 6: Reduce Data Transfer / Egress

Data transfer is often an overlooked cost, especially as applications grow.

**Put your CDN in front of everything:**
```bash
# AWS — CloudFront in front of S3 and ALB
# CloudFront → S3: $0 transfer (same network)
# CloudFront → Users: $0.0085/GB (much cheaper than ALB egress at $0.08/GB)

# Enable caching for API responses that can be cached
# In your Node.js API:
app.get('/api/products', async (req, reply) => {
  const products = await getProducts();
  reply
    .header('Cache-Control', 'public, max-age=300, stale-while-revalidate=60')
    .send(products);
});

# For user-specific data: use short TTLs or private cache
reply.header('Cache-Control', 'private, max-age=60');
```

**Keep traffic within the cloud network:**
```bash
# BAD: RDS in us-east-1, app calling external API in eu-west-1 → cross-region egress
# GOOD: Use VPC endpoints to keep traffic internal

# AWS: S3 VPC endpoint (free, avoids internet egress)
aws ec2 create-vpc-endpoint \
  --vpc-id vpc-abc123 \
  --service-name com.amazonaws.us-east-1.s3 \
  --vpc-endpoint-type Gateway \
  --route-table-ids rtb-abc123

# Same principle: use private endpoints for Azure/GCP managed services
```

### Step 7: Scale to Zero for Non-Production

Development and staging environments should not run 24/7 at production scale.

```bash
# AWS: scale ECS service to 0 outside business hours
# Using CloudWatch Events (EventBridge) + Lambda

# Create Lambda to scale service
aws lambda create-function \
  --function-name scale-nonprod \
  --runtime nodejs22.x \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --role arn:aws:iam::123456789012:role/lambda-role

# Schedule: scale down at 8pm, scale up at 8am weekdays
aws events put-rule \
  --name scale-down-nonprod \
  --schedule-expression "cron(0 23 ? * MON-FRI *)"    # 8pm EST = 23:00 UTC

aws events put-rule \
  --name scale-up-nonprod \
  --schedule-expression "cron(0 12 ? * MON-FRI *)"    # 8am EST = 12:00 UTC

# Savings: ~65% reduction in non-prod compute costs
# (running 12h on weekdays vs 24/7 = 36h vs 168h per week)
```

```bash
# Azure: Container Apps can scale to 0 with min-replicas=0
az containerapp update \
  --name myapp-api-staging \
  --resource-group myapp-staging-rg \
  --min-replicas 0   # scales to zero when no traffic

# GCP: Cloud Run scales to zero by default
# Set min-instances=0 for staging (accept cold start penalty)
gcloud run services update myapp-api-staging \
  --region us-central1 \
  --min-instances 0
```

---

## Common Patterns & Best Practices

### The Optimization Checklist

Run through this monthly:

```
[ ] Unused resources        — stopped EC2 instances, unattached EBS/disks
[ ] Orphaned snapshots      — EBS/disk snapshots older than 90 days
[ ] Empty load balancers    — LBs with no healthy targets (still billed)
[ ] Unattached EIPs         — Elastic IPs not associated with an instance ($3.65/mo each)
[ ] Idle databases          — RDS with <1 connection/hour for a week
[ ] Log retention           — CloudWatch logs with no expiry set
[ ] S3 storage classes      — objects untouched for 90+ days in Standard tier
[ ] Lambda timeouts         — functions with timeout >function p99 (overpaying for compute)
[ ] Unused ECR images       — hundreds of old container image versions
[ ] Data transfer           — is egress unexpectedly high? Added a new external dependency?
```

**Find unused resources with AWS scripts:**
```bash
# Unattached EBS volumes
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Age:CreateTime}' \
  --output table

# EC2 instances with < 5% average CPU (candidates for downsizing)
aws cloudwatch get-metric-statistics \
  --namespace AWS/EC2 \
  --metric-name CPUUtilization \
  --dimensions Name=InstanceId,Value=i-abc123 \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average

# Unused NAT Gateways (common source of waste in dev environments)
aws ec2 describe-nat-gateways \
  --filter Name=state,Values=available \
  --query 'NatGateways[].{ID:NatGatewayId,VPC:VpcId,Created:CreateTime}'
```

### Database Cost Optimization

Databases are often over-provisioned because teams fear downtime during resizing.

```bash
# AWS RDS: resize with minimal downtime (Multi-AZ failover)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --db-instance-class db.t3.medium \    # down from db.m5.large
  --apply-immediately                   # or --no-apply-immediately for next maintenance window

# Monitor to validate the resize worked
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-prod \
  --start-time $(date -d '24 hours ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Average Maximum

# ElastiCache: check memory usage before downsizing
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

## Anti-Patterns to Avoid

### Optimizing Without Measuring

```
BAD: "I think Lambda is too expensive, let's switch to ECS"
     → Changed to ECS, costs increased (had forgotten about Fargate idle time)

GOOD: Pull actual Lambda costs → compare to ECS Fargate cost model for same workload
      → Then decide
```

### Premature Optimization

Don't optimize at $50/month. At $50/month, 20% savings is $10/month — less than one hour of engineering time. Optimize when:
- Monthly bill exceeds $500 (even then, start with visibility)
- A specific service has unexpectedly high costs
- You're planning reserved capacity purchases

### Deleting Without Snapshots

```bash
# Always snapshot before deleting
aws ec2 create-snapshot \
  --volume-id vol-abc123 \
  --description "Backup before deletion $(date)"

# Then delete
aws ec2 delete-volume --volume-id vol-abc123
```

### Not Setting Lifecycle Policies Before Storage Grows

Storage is easy to ignore at $0.023/GB. At 1TB, that's $23/month. At 10TB (not unusual for an application collecting images/logs for a few years), it's $230/month — plus retrieval costs if you moved to cold storage without a plan.

---

## Debugging & Troubleshooting

### Unexpected Cost Spike Investigation

```bash
# AWS: find the spike in Cost Explorer
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-01-31 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --output table

# Common culprits:
# - Data transfer: did you add an external API call that returns large payloads?
# - EC2: did Auto Scaling spin up many instances and not scale back down?
# - RDS: did a migration or data load cause a spike in IOPS?
# - S3 PUT requests: did you introduce a bug that writes to S3 in a loop?
# - Lambda: did a Lambda get stuck in an infinite retry loop?

# Check Lambda error metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Errors \
  --dimensions Name=FunctionName,Value=myapp-processor \
  --start-time $(date -d '3 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Sum
```

### Egress Costs Higher Than Expected

```bash
# Check CloudFront data transfer
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name BytesDownloaded \
  --dimensions Name=DistributionId,Value=E1234567890ABC \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Sum

# Check if cache hit ratio is low (low hit = more origin fetches = more compute + egress)
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

## Real-World Scenarios

### Scenario 1: Reducing a $2,000/Month AWS Bill by 40%

```
Initial bill breakdown:
  EC2 (m5.large × 4, 24/7):      $560/mo
  RDS (db.m5.large Multi-AZ):    $380/mo
  ElastiCache (r6g.large):       $180/mo
  Data transfer:                  $340/mo
  S3:                             $220/mo
  Misc:                           $320/mo
  Total:                         $2,000/mo

Actions taken:
  1. Right-sized EC2 to m5.medium (CPU avg was 18%): -$280/mo
  2. Resized RDS to db.t3.large (same performance, burstable): -$120/mo
  3. Bought 1-year Compute Savings Plans on EC2: -$90/mo
  4. Added CloudFront → cut data transfer 60%: -$200/mo
  5. Set S3 lifecycle to move old objects to IA/Glacier: -$80/mo
  6. Set 30-day retention on CloudWatch logs: -$40/mo

Result: $1,190/mo (41% reduction in 3 weeks of work)
```

### Scenario 2: Cloud Run Cost Model for a SaaS API

```typescript
// Cloud Run pricing:
//   CPU: $0.00002400/vCPU-second (when processing requests)
//   Memory: $0.00000250/GiB-second
//   Requests: $0.40 per million

// For an API handling 1M requests/month, 200ms avg response time:

const requestsPerMonth = 1_000_000;
const avgResponseTimeSeconds = 0.2;
const vCPU = 1;
const memoryGiB = 0.5;

const cpuCost = requestsPerMonth * avgResponseTimeSeconds * vCPU * 0.000024;
// = 1,000,000 * 0.2 * 1 * 0.000024 = $4.80

const memoryCost = requestsPerMonth * avgResponseTimeSeconds * memoryGiB * 0.0000025;
// = 1,000,000 * 0.2 * 0.5 * 0.0000025 = $0.25

const requestCost = (requestsPerMonth / 1_000_000) * 0.40;
// = 1 * 0.40 = $0.40

const totalCost = cpuCost + memoryCost + requestCost;
// = $5.45/month

// At 10M requests: ~$55/month
// At 100M requests: ~$550/month
// Predictable, linear scaling — no reserved capacity needed
```

---

## Further Reading

- [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/)
- [AWS Savings Plans](https://aws.amazon.com/savingsplans/)
- [Azure Cost Management](https://azure.microsoft.com/services/cost-management/)
- [GCP Cost Management](https://cloud.google.com/cost-management)
- [infracost — estimate IaC costs before deploying](https://www.infracost.io/)
- [The FinOps Framework](https://www.finops.org/framework/)

---

## Summary

| Action | Effort | Typical Savings |
|--------|--------|-----------------|
| Enable cost alerts + budgets | 30 min | Prevents bill shock |
| Tag all resources | 1-2 days | Enables cost attribution |
| Review and right-size compute | 2-4 hours | 20-40% on compute |
| Purchase reserved capacity (1yr) | 1 hour | 27-40% on committed services |
| Set S3/GCS lifecycle policies | 1 hour | 30-70% on storage |
| Set log retention policies | 30 min | 20-60% on logging |
| Scale non-prod to zero | 2-4 hours | 60-80% on non-prod compute |
| Add CDN caching | 2-4 hours | 40-80% on data transfer |
| Clean up orphaned resources | 2-3 hours | Varies |

Start with visibility (tagging + cost alerts). Then right-size. Then buy reserved capacity only after your usage is stable and measured. You cannot optimize what you cannot measure.
