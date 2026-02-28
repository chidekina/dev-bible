# AWS Cost Optimization

## Overview

AWS bills grow in predictable ways: compute that's oversized, databases with provisioned IOPS you don't use, S3 buckets with no lifecycle rules, and data transfer costs that sneak up as your application scales. The good news is that each of these has a clear remedy.

This chapter covers AWS-specific cost optimization techniques for Node.js backend deployments. It complements `cloud-cost-optimization.md` (cross-cloud strategies) with AWS-specific tools, pricing models, and CLI commands.

---

## Prerequisites

- AWS Core Services (`aws-core-services.md`)
- AWS account with Cost Explorer enabled
- Basic understanding of your application's traffic patterns

---

## Core Concepts

### AWS Pricing Models

Understanding the pricing models is essential before optimizing:

```
On-Demand:       Pay per hour/second. No commitment. Highest price.
                 Use for: variable workloads, development, new applications

Savings Plans:   Commit to $/hour spend across EC2, Fargate, Lambda.
                 Compute Savings Plans: 1yr/3yr, no-upfront/partial/all-upfront.
                 Up to 66% savings vs on-demand. Most flexible.

Reserved Instances: Commit to specific instance type/region.
                 Up to 72% savings vs on-demand. Less flexible than Savings Plans.
                 Use for: RDS, ElastiCache, Redshift (Savings Plans don't cover these).

Spot Instances:  Spare AWS capacity at up to 90% discount.
                 Can be interrupted with 2-minute warning.
                 Use for: batch jobs, CI/CD workers, stateless fault-tolerant workloads.

Free Tier:       Always-free: Lambda 1M requests/month, DynamoDB 25GB, S3 5GB.
                 12-month free: EC2 t2.micro 750h/month, RDS t2.micro 750h/month.
```

### The AWS Cost Optimization Toolkit

```
Cost Explorer:        Visualize and analyze spending. Historical and forecast.
AWS Budgets:          Alerts when you exceed thresholds. Set on day one.
Compute Optimizer:    ML-powered right-sizing recommendations for EC2, ECS, Lambda.
Trusted Advisor:      Best practice checks including cost optimization.
Cost Anomaly Detection: ML-based alerts when spending deviates from normal.
AWS Pricing Calculator: Estimate costs before deploying.
```

---

## Hands-On Examples

### Step 1: Set Up Billing Visibility

Do this before anything else. You can't optimize what you can't see.

```bash
# Enable Cost Explorer (one-time — takes 24h to populate)
aws ce update-cost-allocation-tags-status \
  --cost-allocation-tags-status TagKey=Project,Status=Active TagKey=Environment,Status=Active TagKey=Team,Status=Active

# Create budget with email alerts at 80% and 100%
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

# Enable Cost Anomaly Detection
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

### Step 2: Tag Everything from Day One

```bash
# Tag individual resources
aws ec2 create-tags \
  --resources i-abc123 \
  --tags \
    Key=Project,Value=myapp \
    Key=Environment,Value=production \
    Key=Team,Value=backend \
    Key=Service,Value=api

# Find all untagged resources (requires Resource Groups Tagging API)
aws resourcegroupstaggingapi get-resources \
  --resource-type-filters ec2:instance rds:db ecs:service elasticache:cluster \
  --tag-filters '[]' \
  --query 'ResourceTagMappingList[?Tags==`[]`].{ARN:ResourceARN}' \
  --output table

# Enforce tagging with AWS Config rule
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

### Step 3: Right-Size EC2 and ECS

```bash
# Get EC2 right-sizing recommendations
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

# Get ECS/Fargate right-sizing recommendations
aws compute-optimizer get-ecs-service-recommendations \
  --query 'ecsServiceRecommendations[?finding!=`Optimized`].{
    Service: serviceArn,
    Finding: finding,
    CurrentCPU: currentServiceConfiguration.cpu,
    CurrentMemory: currentServiceConfiguration.memory,
    MonthlySavings: serviceRecommendationOptions[0].savingsOpportunity.estimatedMonthlySavings.value
  }' \
  --output table

# Get Lambda right-sizing recommendations
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

**Right-sizing a Fargate task:**
```bash
# Step 1: Review actual CPU and memory utilization over 2 weeks
aws cloudwatch get-metric-statistics \
  --namespace ECS/ContainerInsights \
  --metric-name CpuUtilized \
  --dimensions Name=ClusterName,Value=myapp-cluster Name=ServiceName,Value=myapp-api \
  --start-time $(date -d '14 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average Maximum

# Step 2: If p99 CPU < 30% → halve the CPU allocation
# If p99 Memory < 40% → reduce memory

# Step 3: Update the task definition
aws ecs register-task-definition \
  --family myapp \
  --cpu 256 \        # was 512, halving saves ~50%
  --memory 512 \     # was 1024
  ...

# Step 4: Update the service to use new task definition
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --task-definition myapp:NEW_REVISION
```

### Step 4: Savings Plans — The Highest-ROI Action

After 60-90 days of stable production usage, purchase Savings Plans. This is typically the single highest-ROI cost optimization action available.

```bash
# Step 1: Check your hourly on-demand commitment level
# (average of the last 30 days of on-demand EC2 + Fargate + Lambda spend)
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

# Step 2: Use AWS Savings Plans recommendations
aws savingsplans describe-savings-plans-purchase-recommendation \
  --savings-plans-type COMPUTE_SP \
  --term-in-years ONE_YEAR \
  --payment-option NO_UPFRONT \
  --lookback-period-in-days SIXTY_DAYS

# Step 3: Purchase (do this in the console for confirmation)
# Compute Savings Plans cover:
#   - EC2 (any instance type, any region)
#   - Fargate (any region)
#   - Lambda (any region)
# 1-year, no-upfront: ~27% savings
# 1-year, all-upfront: ~40% savings
# 3-year, all-upfront: ~66% savings
```

**Savings Plans purchase strategy:**
```
Conservative (recommended for first purchase):
  Commit to 70% of your current average hourly spend.
  On-demand covers the remaining 30% (variable workloads, spikes).

Aggressive:
  Commit to 90% of your minimum hourly spend.
  Maximizes savings, leaves less buffer for scale-up.

Never commit to 100%:
  If you scale down (team changes, new architecture), you still pay the commitment.
  You can sell unused Savings Plans on the Reserved Instance Marketplace.
```

### Step 5: RDS Cost Optimization

RDS is often the second-largest line item for Node.js applications.

```bash
# Check actual RDS utilization
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name CPUUtilization \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-prod \
  --start-time $(date -d '14 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Average Maximum \
  --output table

# Check connection count
aws cloudwatch get-metric-statistics \
  --namespace AWS/RDS \
  --metric-name DatabaseConnections \
  --dimensions Name=DBInstanceIdentifier,Value=myapp-prod \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics Maximum

# Resize RDS instance (low downtime with Multi-AZ — failover to standby during resize)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --db-instance-class db.t3.medium \     # down from db.m5.large
  --apply-immediately                     # triggers failover immediately, ~60s downtime

# Switch to gp3 storage (same performance, lower cost than gp2)
aws rds modify-db-instance \
  --db-instance-identifier myapp-prod \
  --storage-type gp3 \
  --apply-immediately
  # gp3: $0.115/GB vs gp2: $0.115/GB but gp3 includes 3000 IOPS free
  # gp2: additional IOPS cost $0.10/IOPS-month → can save significantly

# Purchase Reserved Instances for RDS (Savings Plans don't cover RDS)
# 1-year, all-upfront for db.t3.medium PostgreSQL: ~42% savings
```

**RDS vs Aurora Serverless v2 cost comparison:**
```
Use case: API with variable traffic, averaging 200 connections, peaks at 800

RDS db.t3.medium (Multi-AZ):
  db.t3.medium: $0.082/hr × 2 (Multi-AZ) × 720h = $118/mo
  Storage: 50GB gp3 = $5.75/mo
  Total: ~$124/mo

Aurora Serverless v2 (0.5-4 ACU, Multi-AZ):
  ACU: $0.12/ACU-hr × avg 1.5 ACU × 720h = $130/mo
  Storage: $0.10/GB × 50GB = $5/mo
  Total: ~$135/mo

Conclusion: Aurora Serverless v2 is ~9% more expensive for this workload
but auto-scales CPU/memory without maintenance windows.
Worth it when CPU is very spiky or team doesn't want to manage resize operations.
```

### Step 6: S3 Cost Optimization

```bash
# Find buckets with large storage (potential savings with lifecycle rules)
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

# Enable Intelligent-Tiering for buckets where access patterns are unknown
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
# Intelligent-Tiering automatically moves objects to cheaper tiers
# No retrieval fee penalty for Frequent/Infrequent Access tiers

# Clean up incomplete multipart uploads (common hidden cost)
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

# Enable request metrics to see if you're paying for too many API calls
aws s3api put-bucket-metrics-configuration \
  --bucket myapp-uploads \
  --id all-objects \
  --metrics-configuration '{"Id": "all-objects"}'
```

### Step 7: Lambda Cost Optimization

Lambda pricing:
- $0.0000002 per request (first 1M/month free)
- $0.0000166667 per GB-second (first 400,000 GB-seconds/month free)

```bash
# Check Lambda memory allocation vs actual usage
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name MaxMemoryUsed \
  --dimensions Name=FunctionName,Value=myapp-processor \
  --start-time $(date -d '14 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Maximum Average

# If MaxMemoryUsed p99 = 180MB and allocation is 512MB → reduce to 256MB
aws lambda update-function-configuration \
  --function-name myapp-processor \
  --memory-size 256   # ~50% cost reduction, no performance impact

# Check for Lambda cold starts (indicates over-scaling, or need for provisioned concurrency)
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name InitDuration \
  --dimensions Name=FunctionName,Value=myapp-processor \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 3600 \
  --statistics SampleCount Maximum Average

# Provisioned concurrency (for latency-sensitive functions): reduces cold starts
# but costs even when idle ($0.0000041 per provisioned concurrency-second)
aws lambda put-provisioned-concurrency-config \
  --function-name myapp-api \
  --qualifier ALIAS_OR_VERSION \
  --provisioned-concurrent-executions 5

# Lambda Power Tuning: find optimal memory allocation
# https://github.com/alexcasalboni/aws-lambda-power-tuning
# Runs your function with different memory sizes and finds the optimal price/performance point
```

### Step 8: Reduce Data Transfer Costs

```bash
# Check current data transfer costs
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

# Set up VPC endpoints for S3 and DynamoDB (free, eliminates NAT Gateway charges)
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

# Check CloudFront cache hit ratio (low ratio = more origin fetches = more cost)
aws cloudwatch get-metric-statistics \
  --namespace AWS/CloudFront \
  --metric-name CacheHitRate \
  --dimensions Name=DistributionId,Value=E1234567890ABC \
  --start-time $(date -d '7 days ago' --iso-8601) \
  --end-time $(date --iso-8601) \
  --period 86400 \
  --statistics Average

# If cache hit rate < 80%: review your Cache-Control headers
# Improve in Node.js:
# reply.header('Cache-Control', 'public, max-age=300, stale-while-revalidate=60');
```

### Step 9: NAT Gateway — The Hidden Cost

NAT Gateway is often the biggest surprise on AWS bills. It costs $0.045/hour + $0.045/GB of data processed.

```bash
# Check NAT Gateway costs
aws ce get-cost-and-usage \
  --time-period Start=2024-01-01,End=2024-02-01 \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["Amazon Virtual Private Cloud"]}}'

# Common optimization: use VPC endpoints instead of routing through NAT
# Services that have free VPC endpoints (Gateway type): S3, DynamoDB
# Services with Interface endpoints (cost $0.01/hr): Secrets Manager, SSM, ECR, CloudWatch

# For ECR pulls (large image layers through NAT = expensive):
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

# Cost calculation: Interface endpoint at $0.01/hr × 2 (HA) × 720h = $14.40/mo
# If ECR pulls cost >$14.40/mo in NAT charges → worth it
```

---

## Common Patterns & Best Practices

### Monthly Cost Review Script

```bash
#!/bin/bash
# Monthly cost review — run first week of each month
# Requires jq installed

ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
LAST_MONTH_START=$(date -d 'last month' +%Y-%m-01)
LAST_MONTH_END=$(date +%Y-%m-01)

echo "=== Cost by Service ($LAST_MONTH_START to $LAST_MONTH_END) ==="
aws ce get-cost-and-usage \
  --time-period Start=$LAST_MONTH_START,End=$LAST_MONTH_END \
  --granularity MONTHLY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'sort_by(ResultsByTime[0].Groups, &Metrics.UnblendedCost.Amount)[-10:] | [*].{Service:Keys[0], Cost:Metrics.UnblendedCost.Amount}' \
  --output table

echo "=== Unused Elastic IPs ==="
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].{IP:PublicIp,AllocationId:AllocationId}' \
  --output table

echo "=== Unattached EBS Volumes ==="
aws ec2 describe-volumes \
  --filters Name=status,Values=available \
  --query 'Volumes[].{ID:VolumeId,Size:Size,Type:VolumeType,Created:CreateTime}' \
  --output table

echo "=== Stopped EC2 Instances ==="
aws ec2 describe-instances \
  --filters Name=instance-state-name,Values=stopped \
  --query 'Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,Stopped:StateTransitionReason}' \
  --output table

echo "=== Old EBS Snapshots (>90 days) ==="
aws ec2 describe-snapshots \
  --owner-ids self \
  --query "Snapshots[?StartTime<'$(date -d '90 days ago' --iso-8601)'].{ID:SnapshotId,Size:VolumeSize,Created:StartTime}" \
  --output table
```

### Cost Allocation Tags Enforcement

```bash
# AWS Config rule: non-compliant resources get flagged
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

# Send Config findings to SNS for notification
aws configservice put-delivery-channel \
  --delivery-channel '{
    "name": "default",
    "snsTopicARN": "arn:aws:sns:us-east-1:123456789012:config-alerts"
  }'
```

---

## Anti-Patterns to Avoid

### Elastic IPs That Aren't Attached

```bash
# Each unattached EIP costs $3.65/month (AWS charges for unused EIPs)
# Find and release them
aws ec2 describe-addresses \
  --query 'Addresses[?AssociationId==null].AllocationId' \
  --output text | while read alloc; do
    echo "Releasing $alloc"
    aws ec2 release-address --allocation-id "$alloc"
done
```

### NAT Gateway in Dev Environment

```bash
# Dev environments don't need NAT Gateway ($32/month)
# Options:
# 1. Use public subnets for dev (less secure but acceptable for non-production)
# 2. Use a NAT Instance (EC2 t3.nano at $3.50/month) instead of NAT Gateway
# 3. Use VPC endpoints for S3/DynamoDB and accept no other egress

# NAT Instance setup (not recommended for prod but fine for dev)
aws ec2 run-instances \
  --image-id ami-00a9d4a05375b2763 \    # Amazon NAT AMI
  --instance-type t3.nano \
  --subnet-id subnet-public-abc \
  --security-group-ids sg-nat \
  --source-dest-check false             # required for NAT to work
```

### Keeping Old Lambda Versions

```bash
# Lambda stores all deployed versions (each is billed for storage)
# Clean up old versions

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

# Or set a lifecycle policy on the function (new feature)
aws lambda put-function-event-invoke-config \
  --function-name myapp-processor \
  --maximum-retry-attempts 2
```

### CloudWatch Logs Without Expiry

Every Lambda function, ECS task, and EC2 instance creates CloudWatch log groups. By default they never expire and cost $0.50/GB stored.

```bash
# Set retention on all log groups (30 days is usually sufficient)
aws logs describe-log-groups \
  --query 'logGroups[?!retentionInDays].logGroupName' \
  --output text | tr '\t' '\n' | while read lg; do
    echo "Setting 30-day retention on $lg"
    aws logs put-retention-policy \
      --log-group-name "$lg" \
      --retention-in-days 30
done
```

---

## Debugging & Troubleshooting

### Bill Spike Investigation

```bash
# 1. Find the service causing the spike
aws ce get-cost-and-usage \
  --time-period Start=2024-01-15,End=2024-01-22 \
  --granularity DAILY \
  --metrics UnblendedCost \
  --group-by Type=DIMENSION,Key=SERVICE \
  --query 'ResultsByTime[].{Date:TimePeriod.Start, Groups:Groups[*].{Service:Keys[0],Cost:Metrics.UnblendedCost.Amount}}' \
  --output json | jq '.[] | .Date as $date | .Groups[] | {date: $date, service: .Service, cost: (.Cost | tonumber)} | select(.cost > 5)'

# 2. Drill into the specific service
aws ce get-cost-and-usage \
  --time-period Start=2024-01-15,End=2024-01-22 \
  --granularity DAILY \
  --metrics UnblendedCost UsageQuantity \
  --filter '{"Dimensions":{"Key":"SERVICE","Values":["AWS Lambda"]}}' \
  --group-by Type=DIMENSION,Key=USAGE_TYPE

# Common causes:
# Lambda: function stuck in infinite retry loop → check SQS DLQ and Lambda error metrics
# EC2: Auto Scaling launched many instances and didn't scale back → check ASG activity history
# Data Transfer: added external API call returning large payloads
# S3 Requests: bug writing to S3 in a loop
# RDS IOPS: migration or data load caused IOPS spike
```

### Understanding Fargate Pricing

```bash
# Fargate pricing:
# vCPU: $0.04048 per vCPU-hour
# Memory: $0.004445 per GB-hour

# Calculate monthly cost for a task definition
VCPU=0.5
MEMORY_GB=1
HOURS_PER_MONTH=720

VCPU_COST=$(echo "$VCPU * 0.04048 * $HOURS_PER_MONTH" | bc)
MEMORY_COST=$(echo "$MEMORY_GB * 0.004445 * $HOURS_PER_MONTH" | bc)
TOTAL=$(echo "$VCPU_COST + $MEMORY_COST" | bc)

echo "vCPU cost: \$$VCPU_COST/month"
echo "Memory cost: \$$MEMORY_COST/month"
echo "Total per task: \$$TOTAL/month"

# For 2 tasks (minimum for HA): ~$20/month
# For 10 tasks (under load): ~$100/month
# Cost scales linearly with task count
```

---

## Real-World Scenarios

### Scenario 1: $800/Month Bill → $480/Month in 4 Weeks

```
Initial bill breakdown:
  ECS Fargate (4 tasks, 1 vCPU, 2GB each, 24/7): $232/mo
  RDS db.m5.large PostgreSQL Multi-AZ:            $280/mo
  NAT Gateway (2 AZs) + data processing:          $120/mo
  CloudWatch Logs (no retention):                  $45/mo
  S3 (no lifecycle rules):                         $78/mo
  ElastiCache cache.t3.small:                      $28/mo
  Misc:                                            $17/mo
  Total:                                          $800/mo

Actions taken (4 weeks):
  Week 1: Right-sized ECS tasks to 0.5 vCPU, 1GB
          → Fargate: $232 → $116/mo
  Week 1: Set 30-day CloudWatch log retention
          → Logs: $45 → $8/mo
  Week 2: Added VPC endpoints for S3, ECR, Secrets Manager
          → NAT: $120 → $68/mo (reduced ECR pull traffic)
  Week 2: Set S3 lifecycle rules (Intelligent-Tiering)
          → S3: $78 → $45/mo
  Week 3: Resized RDS to db.t3.medium (CPU p99 was 15%)
          → RDS: $280 → $124/mo
  Week 4: Purchased 1-year Compute Savings Plans at 70% of new spend
          → Fargate discount: ~$30/mo

New total: $116 + $8 + $68 + $45 + $124 + $28 + $17 - $30 = $376/mo
Savings: $424/mo (53% reduction)
```

### Scenario 2: Cost Model for a Growing SaaS

```
Traffic: 10,000 users, 50 requests/user/day = 500,000 requests/day = 15M requests/month
API: 2× Fargate tasks (0.5 vCPU, 1GB), autoscaling 2-8 tasks
Database: RDS db.t3.medium PostgreSQL Multi-AZ
Cache: ElastiCache cache.t3.micro Redis
Storage: S3 (~100GB uploaded files, CDN via CloudFront)

Compute:
  Fargate avg 3 tasks × $14.54/task/mo = $43.62/mo
  1yr Savings Plans (40% discount): $26.17/mo

Database:
  RDS db.t3.medium Multi-AZ: $124/mo
  1yr Reserved Instance (42% discount): $72/mo

Cache:
  ElastiCache cache.t3.micro: $12.24/mo

Storage + CDN:
  S3: 100GB × $0.023 = $2.30/mo
  CloudFront: 15M requests × $0.0075/10k + 1TB × $0.0085 = $11.25 + $8.70 = $19.95/mo

Networking:
  NAT Gateway (2 AZs): $32/mo

Misc:
  Route 53: $1/mo
  Secrets Manager: $1/mo
  CloudWatch: $5/mo

Total: $26.17 + $72 + $12.24 + $19.95 + $32 + $1 + $1 + $5 + $2.30 = $171.66/mo

Unit cost: $171.66 / 10,000 users = $0.017/user/month
If users pay $10/month: COGS = 0.17% of revenue → very healthy
```

---

## Further Reading

- [AWS Cost Explorer](https://aws.amazon.com/aws-cost-management/aws-cost-explorer/)
- [AWS Compute Optimizer](https://aws.amazon.com/compute-optimizer/)
- [AWS Savings Plans](https://aws.amazon.com/savingsplans/)
- [AWS Trusted Advisor](https://aws.amazon.com/premiumsupport/technology/trusted-advisor/)
- [AWS Pricing Calculator](https://calculator.aws/pricing/2/home)
- [Lambda Power Tuning](https://github.com/alexcasalboni/aws-lambda-power-tuning) — find optimal Lambda memory
- [infracost](https://www.infracost.io/) — estimate Terraform changes cost before applying

---

## Summary

| Action | Effort | Potential Savings | Priority |
|--------|--------|-------------------|----------|
| Set billing alerts + budget | 30 min | Prevents surprises | Do today |
| Tag all resources | 1-2 days | Enables attribution | Do this week |
| Enable Cost Anomaly Detection | 15 min | Early spike detection | Do today |
| Right-size EC2/ECS tasks | 2-4 hr | 20-50% on compute | High |
| Purchase Compute Savings Plans | 1 hr | 27-40% on EC2/Fargate/Lambda | High (after 60 days) |
| Purchase RDS Reserved Instances | 1 hr | 42% on RDS | High (after 60 days) |
| S3 lifecycle rules + Intelligent-Tiering | 1 hr | 30-70% on storage | Medium |
| Set CloudWatch log retention | 30 min | 80%+ on logging | Easy win |
| S3 + DynamoDB VPC endpoints | 1 hr | Eliminates NAT costs for those services | Medium |
| Clean up unattached EIPs | 15 min | $3.65/EIP/month | Easy win |
| Remove old CloudWatch log groups | 30 min | Varies | Easy win |
| Lambda memory right-sizing | 1-2 hr | 20-40% on Lambda | Medium |

AWS cost optimization is a continuous practice, not a one-time project. Set billing alerts on day one, review costs monthly, right-size after 60 days of production data, and purchase reserved capacity after 90 days of stable usage.
