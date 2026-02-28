# AWS Architecture

## Overview

Knowing individual AWS services is not enough. The real skill is composing them into architectures that are reliable, secure, scalable, and cost-effective. AWS published the **Well-Architected Framework** — a set of design principles and best practices organized into six pillars — to guide exactly this.

This chapter covers architectural patterns for common use cases: the three-tier web application, serverless API, event-driven processing, and multi-region resilience. Each pattern is grounded in the Well-Architected Framework's principles and shows how the services from `aws-core-services.md` fit together.

---

## Prerequisites

- AWS Core Services (`aws-core-services.md`)
- Cloud Concepts (`cloud-concepts.md`)
- Basic networking knowledge (VPC, subnets, routing)

---

## Core Concepts

### The Well-Architected Framework

Six pillars that guide AWS architecture decisions:

| Pillar | Core Question |
|--------|--------------|
| **Operational Excellence** | How do we run and improve the system? |
| **Security** | How do we protect information and systems? |
| **Reliability** | How do we recover from failures? |
| **Performance Efficiency** | How do we use resources efficiently? |
| **Cost Optimization** | How do we avoid unnecessary costs? |
| **Sustainability** | How do we minimize environmental impact? |

### Reliability Design Principles

Reliable systems are built on these foundations:

**1. Multi-AZ deployment:** Deploy across at least two Availability Zones. AZ failures are rare but happen.

**2. Auto Scaling:** Automatically replace unhealthy instances and scale to demand.

**3. Health checks everywhere:** ALB health checks, ECS health checks, RDS multi-AZ automatic failover.

**4. Circuit breakers:** Stop calling a failing dependency to let it recover.

**5. Graceful degradation:** Return cached/partial results rather than failing completely.

```
Failure domain hierarchy:
Instance → AZ → Region → Provider

Architect for AZ failures (common).
Consider Region failures (rare) for critical systems.
Multi-provider is almost never worth the complexity.
```

### Security Architecture Principles

**Defense in depth:**
```
Internet
    ↓
WAF (block malicious requests)
    ↓
CloudFront (DDoS mitigation)
    ↓
Security Group (ports 80/443 only)
    ↓
ALB (HTTPS termination)
    ↓
Private subnet (no public IP on app servers)
Security Group (only from ALB)
    ↓
App (ECS Fargate, no root)
    ↓
Private subnet (database tier)
Security Group (only from app security group)
    ↓
RDS (encrypted, no public access)
```

No layer is sufficient alone. Each adds friction for an attacker.

---

## Architecture Patterns

### Pattern 1: Three-Tier Web Application

The canonical production web architecture for most applications.

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
              │    Balancer (ALB)        │ ← Public subnet
              └──────┬──────────┬────────┘
                     │          │
           ┌─────────▼──┐  ┌────▼────────┐
           │ ECS Fargate│  │ ECS Fargate │ ← Private subnet (2 AZs)
           │ Task (AZ-a)│  │ Task (AZ-b) │
           └─────────┬──┘  └────┬────────┘
                     │          │
              ┌──────▼──────────▼──────┐
              │     RDS PostgreSQL     │ ← Database subnet
              │  Primary   │  Standby  │
              │  (AZ-a)    │  (AZ-b)   │
              └────────────────────────┘
```

**VPC structure:**
```hcl
# Terraform example
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
  tags = { Name = "myapp-vpc" }
}

# Public subnets (ALB, NAT Gateway)
resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]
  map_public_ip_on_launch = true
}

# Private subnets (ECS tasks)
resource "aws_subnet" "private" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 10)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}

# Database subnets (RDS)
resource "aws_subnet" "database" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet("10.0.0.0/16", 8, count.index + 20)
  availability_zone = data.aws_availability_zones.available.names[count.index]
}
```

**Security groups:**
```bash
# ALB security group — accepts traffic from internet
aws ec2 create-security-group --group-name alb-sg --description "ALB" --vpc-id vpc-abc
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 443 --cidr 0.0.0.0/0

# App security group — accepts traffic only from ALB
aws ec2 create-security-group --group-name app-sg --description "App servers" --vpc-id vpc-abc
aws ec2 authorize-security-group-ingress --group-id sg-app --protocol tcp --port 3000 --source-group sg-alb

# DB security group — accepts traffic only from app servers
aws ec2 create-security-group --group-name db-sg --description "Database" --vpc-id vpc-abc
aws ec2 authorize-security-group-ingress --group-id sg-db --protocol tcp --port 5432 --source-group sg-app
```

**ECS Service with Auto Scaling:**
```bash
# Register auto scaling target
aws application-autoscaling register-scalable-target \
  --service-namespace ecs \
  --resource-id service/myapp-cluster/myapp-api \
  --scalable-dimension ecs:service:DesiredCount \
  --min-capacity 2 \
  --max-capacity 20

# Scale based on CPU utilization
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

### Pattern 2: Serverless API

For APIs with unpredictable traffic or low sustained load:

```
Route 53 → CloudFront → API Gateway → Lambda → RDS Proxy → RDS

Why RDS Proxy?
  Lambda scales from 0 to 1000+ concurrent executions instantly.
  Each Lambda opens a DB connection.
  Without RDS Proxy: 1000 connections hammer Postgres → OOM.
  With RDS Proxy: connections are pooled → Postgres sees 10-20 connections.
```

AWS CDK example (TypeScript):
```typescript
import * as cdk from 'aws-cdk-lib';
import * as lambda from 'aws-cdk-lib/aws-lambda';
import * as apigateway from 'aws-cdk-lib/aws-apigateway';
import * as rds from 'aws-cdk-lib/aws-rds';
import * as nodejs from 'aws-cdk-lib/aws-lambda-nodejs';

export class ServerlessApiStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    // Lambda function
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

    // Wire up routes
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

### Pattern 3: Event-Driven Processing

For async workloads: image processing, email sending, report generation:

```
S3 (upload) ──→ S3 Event Notification ──→ SQS Queue ──→ Lambda
                                                              ↓
                                               Process image, store result
                                                              ↓
                                                      SNS → email/push notification
```

```typescript
// Lambda triggered by SQS
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
      // Download file from S3
      const s3Response = await s3.send(new GetObjectCommand({ Bucket: bucket, Key: key }));
      const buffer = await streamToBuffer(s3Response.Body as NodeJS.ReadableStream);

      // Process the file (e.g., resize image)
      const processed = await processImage(buffer);

      // Store result
      await s3.send(new PutObjectCommand({
        Bucket: process.env.OUTPUT_BUCKET!,
        Key: `processed/${key}`,
        Body: processed,
      }));

      // Notify user
      await sns.send(new PublishCommand({
        TopicArn: process.env.NOTIFICATION_TOPIC_ARN!,
        Message: JSON.stringify({ userId, status: 'complete', outputKey: `processed/${key}` }),
      }));
    } catch (err) {
      console.error({ err, messageId: record.messageId }, 'Failed to process message');
      throw err;   // SQS will retry (up to maxReceiveCount, then send to DLQ)
    }
  }
};
```

SQS best practices:
```bash
# Create SQS queue with Dead Letter Queue
aws sqs create-queue --queue-name myapp-image-processing-dlq

aws sqs create-queue \
  --queue-name myapp-image-processing \
  --attributes '{
    "VisibilityTimeout": "300",
    "RedrivePolicy": "{\"deadLetterTargetArn\":\"arn:aws:sqs:us-east-1:123:myapp-image-processing-dlq\",\"maxReceiveCount\":\"3\"}"
  }'
```

### Pattern 4: Static Site + API (Jamstack)

```
CloudFront ──→ S3 (React/Next.js static export)
     └──────→ ALB / API Gateway (Node.js API)

Benefits:
- Static assets served from edge (global, fast, cheap)
- API scales independently
- S3 + CloudFront = virtually unlimited scale for frontend
- No server to manage for the frontend
```

```bash
# Build and deploy frontend
npm run build

# Sync to S3
aws s3 sync ./out s3://myapp-static/ --delete

# Invalidate CloudFront cache
aws cloudfront create-invalidation \
  --distribution-id E1234567890ABC \
  --paths "/*"
```

CloudFront configuration for SPA with API:
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

## Operational Patterns

### Blue-Green Deployment

Run two identical environments, switch traffic between them:

```bash
# Create green target group (same as blue, points to new task definition)
aws elbv2 create-target-group --name myapp-green-tg ...

# Deploy new version to green ECS service
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-green \
  --task-definition myapp:new

# Wait for green to be healthy
aws ecs wait services-stable --cluster myapp-cluster --services myapp-green

# Switch ALB listener to green
aws elbv2 modify-listener \
  --listener-arn arn:... \
  --default-actions Type=forward,TargetGroupArn=arn:.../myapp-green-tg

# If anything goes wrong:
aws elbv2 modify-listener \
  --listener-arn arn:... \
  --default-actions Type=forward,TargetGroupArn=arn:.../myapp-blue-tg
```

### CI/CD Pipeline with CodePipeline

```yaml
# buildspec.yml — CodeBuild config
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

### Disaster Recovery Planning

RTO (Recovery Time Objective) vs RPO (Recovery Point Objective):

```
RPO = How much data can you afford to lose?
      → Set your backup frequency to be less than RPO
      → RDS: automated backups every 5 minutes (PITR)

RTO = How long can the system be down?
      → Multi-AZ RDS: ~60s failover
      → Multi-region: 5-30 minutes (if pre-warmed)
      → Single-region from backup: hours
```

DR strategies by tier:

| Strategy | RTO | RPO | Cost | Complexity |
|----------|-----|-----|------|------------|
| Backup & restore | Hours | Hours | $ | Low |
| Pilot light | 10-30 min | Minutes | $$ | Medium |
| Warm standby | 5-10 min | Seconds | $$$ | High |
| Active-Active | Near zero | Near zero | $$$$ | Very high |

---

## Cost Optimization Patterns

### Right-Sizing

```bash
# AWS Compute Optimizer gives recommendations based on actual usage
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

After running an application for 2-3 months with stable usage, purchase reserved capacity:

```
On-demand:   $0.0832/hr (t3.medium)
1yr Reserved: $0.0520/hr (37% savings)
3yr Reserved: $0.0336/hr (60% savings)

For 2 t3.medium instances running 24/7:
On-demand: 2 × $0.0832 × 8760h = $1,456/yr
3yr Reserved: 2 × $0.0336 × 8760h = $589/yr
Savings: $867/yr
```

### S3 Intelligent-Tiering

```bash
# Automatically move objects to cheaper storage classes based on access patterns
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

## Anti-Patterns to Avoid

### No Tagging Strategy

Without tags, you can't filter costs by team, environment, or project:

```bash
# Tag everything from the start
aws ec2 create-tags --resources i-abc123 --tags \
  Key=Project,Value=myapp \
  Key=Environment,Value=production \
  Key=Team,Value=backend \
  Key=CostCenter,Value=engineering
```

### Single Region, Single AZ

A single AZ deployment is equivalent to a single server — one hardware failure ends your service. Always deploy across at least two AZs for production.

### Not Setting Up Billing Alerts

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

Set this on day one. A misconfigured Lambda or forgotten EC2 instance can run up hundreds of dollars.

### Using Public Subnets for Databases

Databases should never be in public subnets, even if their security groups are restrictive. Defense in depth — no public IP means no exposure.

### Manual Console Configuration

```
Clicked through the console to create resources → can't reproduce → can't review changes
→ Use CloudFormation, CDK, or Terraform from the start
```

---

## Debugging & Troubleshooting

### Common Architecture Issues

**ECS task keeps restarting:**
```bash
# Check stopped task reason
aws ecs describe-tasks \
  --cluster myapp-cluster \
  --tasks TASK_ARN \
  --query 'tasks[].stoppedReason'

# Check CloudWatch logs
aws logs get-log-events \
  --log-group-name /ecs/myapp \
  --log-stream-name ecs/api/TASK_ID \
  --limit 50
```

**ALB 502 Bad Gateway:**
```bash
# Check target group health
aws elbv2 describe-target-health \
  --target-group-arn arn:aws:elasticloadbalancing:...

# Common causes:
# - Health check path returns non-2xx
# - Container not started on correct port
# - Security group blocks ALB → ECS communication
```

**Lambda timeout:**
```bash
# Check CloudWatch metrics
aws cloudwatch get-metric-statistics \
  --namespace AWS/Lambda \
  --metric-name Duration \
  --dimensions Name=FunctionName,Value=myapp-api \
  --start-time 2024-01-01T00:00:00Z \
  --end-time 2024-01-02T00:00:00Z \
  --period 3600 \
  --statistics p95 p99
```

**RDS connections exhausted:**
```bash
# Check active connections
aws rds describe-db-log-files \
  --db-instance-identifier myapp-prod

# Use RDS Proxy to pool connections for Lambda workloads
# Or increase max_connections in parameter group for ECS workloads
```

---

## Real-World Scenarios

### Scenario 1: Architecture for a SaaS MVP

```
Budget: $100-200/month
Users: 0-1000 (unknown growth)

Architecture:
  DNS:        Route 53 ($0.50/zone)
  CDN:        CloudFront (free tier for first 1TB/mo)
  Frontend:   S3 static hosting ($0.023/GB)
  API:        1× ECS Fargate task (t-equivalent, ~$15/mo)
  Database:   RDS db.t3.micro PostgreSQL ($13/mo)
  Cache:      ElastiCache cache.t3.micro ($13/mo) [optional at MVP]
  Secrets:    Secrets Manager ($0.40/secret)
  Monitoring: CloudWatch (free tier, then ~$5/mo)
  Total:      ~$50/mo

When to upgrade:
  - API: Add second task + ALB when uptime matters (~+$20/mo)
  - DB: Move to db.t3.small when CPU consistently >70%
  - Cache: Add when DB CPU spikes or P95 latency > 200ms
```

### Scenario 2: Preparing for a Traffic Spike

```bash
# Pre-warm ECS service before a known spike
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --desired-count 10

# Request temporary Lambda concurrency increase from AWS Support
# (if using serverless)

# Enable CloudFront caching for API responses that can be cached
# Cache-Control: public, max-age=60 for list endpoints

# After spike:
aws ecs update-service \
  --cluster myapp-cluster \
  --service myapp-api \
  --desired-count 2
```

### Scenario 3: Setting Up Infrastructure as Code

```bash
# Install AWS CDK
npm install -g aws-cdk

# Bootstrap CDK in your account
cdk bootstrap aws://ACCOUNT_ID/us-east-1

# Create new CDK app
mkdir myapp-infra && cd myapp-infra
cdk init app --language typescript

# Deploy
cdk deploy --all

# See what will change before deploying
cdk diff
```

---

## Further Reading

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [AWS Architecture Center](https://aws.amazon.com/architecture/)
- [AWS Solutions Library](https://aws.amazon.com/solutions/) — vetted reference architectures
- [CDK Patterns](https://cdkpatterns.com/) — open-source CDK architecture patterns
- [The Cloud Resume Challenge](https://cloudresumechallenge.dev/) — hands-on AWS learning
- [AWS Pricing Calculator](https://calculator.aws/)

---

## Summary

| Pattern | When to Use |
|---------|-------------|
| Three-tier web app | Most production applications with persistent state |
| Serverless API | Unpredictable traffic, event-driven, low sustained load |
| Event-driven processing | Async workloads: image processing, email, reports |
| Jamstack | Content-heavy sites, SPAs, frontends that can be statically exported |
| Blue-green deployment | Zero-downtime deploys with instant rollback |
| Multi-AZ | Always for production databases and services |
| RDS Proxy | Any Lambda function that connects to RDS |
| Infrastructure as Code | Always — use CDK, Terraform, or CloudFormation |
| Billing alerts | Day one — always |
| Resource tagging | Day one — enables cost attribution and filtering |

Architecture is the art of making the right trade-offs. Start simple (single region, two AZs, ECS Fargate, RDS multi-AZ), measure, and add complexity only when you have evidence it's needed. The Well-Architected Framework is a lens, not a checklist — use it to ask better questions, not to blindly implement every recommendation.
