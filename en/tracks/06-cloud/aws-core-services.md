# AWS Core Services

## Overview

AWS (Amazon Web Services) is the world's largest cloud platform with over 200 services. Knowing all of them isn't the goal — knowing the 20 services that cover 90% of real-world use cases is.

This chapter covers the AWS services every backend developer needs to understand: compute (EC2, ECS, Lambda), storage (S3, EBS, EFS), databases (RDS, DynamoDB, ElastiCache), networking (VPC, Route 53, CloudFront, ALB), and operations (IAM, CloudWatch, SSM).

We focus on practical usage: when to pick each service, how to configure it correctly, and the gotchas that cost teams time and money.

---

## Prerequisites

- AWS account (free tier covers most examples)
- AWS CLI installed and configured
- Basic cloud concepts (see `cloud-concepts.md`)

```bash
# Install AWS CLI v2
curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
unzip awscliv2.zip && sudo ./aws/install

# Configure
aws configure
# AWS Access Key ID: ...
# AWS Secret Access Key: ...
# Default region name: us-east-1
# Default output format: json

# Verify
aws sts get-caller-identity
```

---

## Core Concepts

### Service Categories

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

## Compute Services

### EC2 (Elastic Compute Cloud)

Virtual machines. The building block of AWS compute.

**Instance naming:** `t3.medium`
- Family: `t` = burstable, `m` = general purpose, `c` = compute optimized, `r` = memory optimized
- Generation: `3` (higher = newer, better price/performance)
- Size: `nano` < `micro` < `small` < `medium` < `large` < `xlarge` < `2xlarge`

```bash
# Launch an instance
aws ec2 run-instances \
  --image-id ami-0c02fb55956c7d316 \   # Amazon Linux 2023
  --instance-type t3.small \
  --key-name my-key-pair \
  --security-group-ids sg-abc12345 \
  --subnet-id subnet-abc12345 \
  --iam-instance-profile Name=MyInstanceProfile \
  --user-data file://bootstrap.sh \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=myapp}]'

# List instances
aws ec2 describe-instances \
  --filters "Name=tag:Name,Values=myapp" \
  --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,IP:PublicIpAddress}'
```

`bootstrap.sh` (User Data — runs on first boot):
```bash
#!/bin/bash
yum update -y
curl -fsSL https://get.docker.com | sh
usermod -aG docker ec2-user
systemctl enable --now docker

# Install your application
docker pull ghcr.io/myorg/myapp:latest
docker run -d --restart=unless-stopped -p 3000:3000 ghcr.io/myorg/myapp:latest
```

**When to use EC2:**
- Need full OS control
- Running long-lived workloads
- Specific instance types (GPU, high memory)
- When ECS/Lambda don't fit the use case

### ECS (Elastic Container Service)

Run Docker containers on AWS. Two launch types:

**EC2 launch type:** Containers run on EC2 instances you manage.
**Fargate launch type:** Serverless containers — AWS manages the underlying infrastructure. You pay per task, not per EC2 instance.

Key concepts:
```
Cluster → logical grouping of compute capacity
Task Definition → blueprint (like docker-compose.yml for one service)
Task → running instance of a Task Definition
Service → maintains N tasks running, handles rolling deploys
```

```bash
# Register a task definition
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

# Create a service
aws ecs create-service \
  --cluster myapp-cluster \
  --service-name myapp-api \
  --task-definition myapp:1 \
  --desired-count 2 \
  --launch-type FARGATE \
  --network-configuration "awsvpcConfiguration={subnets=[subnet-abc,subnet-def],securityGroups=[sg-abc],assignPublicIp=DISABLED}" \
  --load-balancers "targetGroupArn=arn:aws:elasticloadbalancing:...,containerName=api,containerPort=3000"
```

**When to use Fargate (vs EC2 launch type):**
- Most applications — simpler, no EC2 management
- Irregular traffic patterns (pay per task)
- Security isolation requirements

### Lambda (Serverless Functions)

Run code without servers. Billed per 100ms of execution time and number of invocations.

```typescript
// Lambda function handler (Node.js/TypeScript)
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
# Deploy a Lambda function
zip function.zip index.js

aws lambda create-function \
  --function-name myapp-users \
  --runtime nodejs22.x \
  --role arn:aws:iam::123456789012:role/lambda-execution-role \
  --handler index.handler \
  --zip-file fileb://function.zip \
  --timeout 30 \
  --memory-size 256

# Update code
aws lambda update-function-code \
  --function-name myapp-users \
  --zip-file fileb://function.zip

# Invoke directly
aws lambda invoke \
  --function-name myapp-users \
  --payload '{"httpMethod":"GET","path":"/users"}' \
  response.json && cat response.json
```

**Lambda limits and gotchas:**
- Max execution time: 15 minutes
- Max memory: 10 GB
- Cold start: 100ms-1s (Java worse, Node.js/Python better)
- Concurrent executions per account: 1000 (soft limit)
- Package size: 50 MB zipped, 250 MB unzipped

**When to use Lambda:**
- Event-driven processing (S3 uploads, SQS messages, DynamoDB streams)
- APIs with unpredictable or low traffic
- Cron jobs / scheduled tasks
- Webhooks (pay per call, not idle time)

---

## Storage Services

### S3 (Simple Storage Service)

Object storage. Virtually unlimited capacity, 11 nines of durability. The most-used AWS service.

```bash
# Create a bucket
aws s3 mb s3://myapp-uploads-prod --region us-east-1

# Upload
aws s3 cp ./logo.png s3://myapp-uploads-prod/images/logo.png

# Upload entire directory
aws s3 sync ./public/ s3://myapp-static-prod/ --delete

# List
aws s3 ls s3://myapp-uploads-prod/

# Get presigned URL (for private files)
aws s3 presign s3://myapp-uploads-prod/documents/report.pdf --expires-in 3600

# Set bucket policy for public read
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

S3 from Node.js (AWS SDK v3):
```typescript
import { S3Client, PutObjectCommand, GetObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({ region: 'us-east-1' });

// Upload a file
async function uploadFile(key: string, body: Buffer, contentType: string): Promise<string> {
  await s3.send(new PutObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
    Body: body,
    ContentType: contentType,
  }));
  return `https://${process.env.S3_BUCKET}.s3.amazonaws.com/${key}`;
}

// Generate presigned URL for temporary access
async function getPresignedUrl(key: string, expiresIn = 3600): Promise<string> {
  const command = new GetObjectCommand({
    Bucket: process.env.S3_BUCKET!,
    Key: key,
  });
  return getSignedUrl(s3, command, { expiresIn });
}
```

**S3 storage classes:**
| Class | Use case | Retrieval | Cost |
|-------|----------|-----------|------|
| Standard | Frequently accessed data | Instant | $$$ |
| Standard-IA | Infrequently accessed | Instant | $$ |
| One Zone-IA | Non-critical infrequent | Instant | $ |
| Glacier Instant | Archive, accessed quarterly | Instant | $ |
| Glacier Flexible | Archive, accessed rarely | 1-12 hours | $$ |
| Glacier Deep Archive | Long-term archive | 12-48 hours | $ |

Use lifecycle rules to automatically transition objects between classes.

### EBS (Elastic Block Store)

Persistent block storage for EC2 instances. Think of it as a network-attached hard drive.

```bash
# Create a volume
aws ec2 create-volume \
  --availability-zone us-east-1a \
  --volume-type gp3 \
  --size 50 \
  --throughput 125 \
  --iops 3000

# Attach to an instance
aws ec2 attach-volume \
  --volume-id vol-abc123 \
  --instance-id i-abc123 \
  --device /dev/sdf
```

**Volume types:**
| Type | Use case | IOPS |
|------|----------|------|
| gp3 | General purpose (default) | 3000-16000 |
| io2 Block Express | High-performance databases | Up to 256,000 |
| sc1 | Cold HDD, throughput-optimized | 250 |
| st1 | Throughput-optimized HDD | 500 |

**Key EBS facts:**
- EBS volumes are AZ-specific (can't attach to instance in different AZ)
- Snapshots replicate across AZs within a region
- EBS volumes persist independently of the EC2 instance lifecycle

---

## Database Services

### RDS (Relational Database Service)

Managed relational databases: PostgreSQL, MySQL, MariaDB, Oracle, SQL Server, Aurora.

```bash
# Create a PostgreSQL RDS instance
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
  --multi-az \                         # deploy standby in another AZ
  --storage-encrypted \
  --backup-retention-period 7 \        # 7 days of automated backups
  --deletion-protection \              # prevent accidental deletion
  --no-publicly-accessible \           # private subnet only
  --vpc-security-group-ids sg-abc123
```

**RDS vs Aurora:**
- Aurora is AWS's MySQL/Postgres-compatible engine
- Aurora is 3x-5x faster than standard MySQL
- Aurora Serverless v2: autoscales compute, great for variable workloads
- Aurora is ~20% more expensive but includes high availability by default

**RDS gotchas:**
- `db.t3.micro` and smaller are burstable — CPU credits can run out under sustained load
- Multi-AZ standby is not a read replica — it only activates on failover
- Point-in-time recovery only goes back to 5 minutes ago at best

### ElastiCache

Managed in-memory caching: Redis and Memcached.

```bash
# Create a Redis cluster
aws elasticache create-replication-group \
  --replication-group-id myapp-cache \
  --description "Application cache" \
  --cache-node-type cache.t3.micro \
  --engine redis \
  --engine-version 7.0 \
  --num-cache-clusters 2 \   # primary + one replica
  --security-group-ids sg-abc123 \
  --subnet-group-name myapp-cache-subnet-group \
  --at-rest-encryption-enabled \
  --transit-encryption-enabled
```

ElastiCache from Node.js:
```typescript
import { createClient } from 'redis';

const redis = createClient({
  url: `redis://${process.env.ELASTICACHE_ENDPOINT}:6379`,
  socket: {
    tls: process.env.NODE_ENV === 'production',   // ElastiCache in-transit encryption
  },
});

await redis.connect();

// Cache-aside pattern
async function getCachedUser(userId: string) {
  const cacheKey = `user:${userId}`;
  const cached = await redis.get(cacheKey);

  if (cached) return JSON.parse(cached);

  const user = await db.user.findUnique({ where: { id: userId } });
  if (user) {
    await redis.setEx(cacheKey, 300, JSON.stringify(user));  // TTL: 5 minutes
  }
  return user;
}
```

---

## Networking Services

### VPC (Virtual Private Cloud)

Your isolated private network in AWS. All resources run inside a VPC.

```
VPC: 10.0.0.0/16

  Public Subnets (have route to Internet Gateway)
    10.0.1.0/24 (us-east-1a) → Load Balancers, NAT Gateway
    10.0.2.0/24 (us-east-1b) → Load Balancers

  Private Subnets (no direct internet access)
    10.0.10.0/24 (us-east-1a) → ECS tasks, EC2 instances
    10.0.11.0/24 (us-east-1b) → ECS tasks, EC2 instances

  Database Subnets (even more restricted)
    10.0.20.0/24 (us-east-1a) → RDS, ElastiCache
    10.0.21.0/24 (us-east-1b) → RDS, ElastiCache
```

**Security Groups:** Stateful firewall at the resource level.
**Network ACLs (NACLs):** Stateless firewall at the subnet level.

### Application Load Balancer (ALB)

Layer 7 (HTTP/HTTPS) load balancer. Distributes traffic, handles SSL termination, supports path-based and host-based routing.

```bash
# Create target group
aws elbv2 create-target-group \
  --name myapp-tg \
  --protocol HTTP \
  --port 3000 \
  --vpc-id vpc-abc123 \
  --target-type ip \
  --health-check-path /health \
  --health-check-interval-seconds 30

# Create ALB
aws elbv2 create-load-balancer \
  --name myapp-alb \
  --subnets subnet-abc subnet-def \
  --security-groups sg-abc123 \
  --scheme internet-facing

# Add HTTPS listener
aws elbv2 create-listener \
  --load-balancer-arn arn:aws:elasticloadbalancing:... \
  --protocol HTTPS \
  --port 443 \
  --certificates CertificateArn=arn:aws:acm:us-east-1:...:certificate/abc \
  --default-actions Type=forward,TargetGroupArn=arn:aws:elasticloadbalancing:...
```

### Route 53

AWS's DNS service. Manages domain registration, DNS records, and routing policies.

```bash
# Create a record set
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

Global content delivery network. Caches content at edge locations worldwide, dramatically reducing latency.

```bash
# Create a CloudFront distribution for S3 static site
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

## Security Services

### IAM (Identity and Access Management)

Control who can access what in AWS. Everything in AWS is an IAM call.

```bash
# Create an IAM role for ECS tasks
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

# Attach a policy (S3 access example)
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

Store and rotate secrets (database passwords, API keys) securely.

```bash
# Store a secret
aws secretsmanager create-secret \
  --name myapp/database-url \
  --secret-string "postgres://app:mysecretpassword@rds-host:5432/app"

# Retrieve a secret
aws secretsmanager get-secret-value \
  --secret-id myapp/database-url \
  --query SecretString \
  --output text
```

From Node.js:
```typescript
import { SecretsManagerClient, GetSecretValueCommand } from '@aws-sdk/client-secrets-manager';

const secretsManager = new SecretsManagerClient({ region: 'us-east-1' });

async function getSecret(secretId: string): Promise<string> {
  const { SecretString } = await secretsManager.send(
    new GetSecretValueCommand({ SecretId: secretId })
  );
  if (!SecretString) throw new Error(`Secret ${secretId} has no string value`);
  return SecretString;
}

// At startup
const DATABASE_URL = await getSecret('myapp/database-url');
```

### CloudWatch

AWS's monitoring, logging, and alerting service.

```bash
# Create a metric alarm
aws cloudwatch put-metric-alarm \
  --alarm-name myapp-error-rate-high \
  --alarm-description "Error rate above 5%" \
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

# Query application logs
aws logs filter-log-events \
  --log-group-name /ecs/myapp \
  --start-time $(date -d '1 hour ago' +%s000) \
  --filter-pattern '"level":50'   # Pino error level
```

---

## Anti-Patterns to Avoid

### Hardcoding AWS Credentials

```typescript
// BAD — never put credentials in code
const s3 = new S3Client({
  credentials: {
    accessKeyId: 'AKIAIOSFODNN7EXAMPLE',
    secretAccessKey: 'wJalrXUtnFEMI/K7MDENG/bPxRfiCYEXAMPLEKEY',
  },
});

// GOOD — use IAM roles (EC2/ECS), or environment variables for local dev
// AWS SDK automatically discovers credentials from IAM role, env vars, ~/.aws/credentials
const s3 = new S3Client({ region: 'us-east-1' });
```

### Using Root Account Credentials

Create IAM users and roles. Never use root account API keys for anything.

### Forgetting to Enable Versioning on S3 Buckets

```bash
aws s3api put-bucket-versioning \
  --bucket myapp-critical-data \
  --versioning-configuration Status=Enabled
```

Without versioning, deleting or overwriting an object is unrecoverable.

### Wide-Open Security Groups

```bash
# BAD — 0.0.0.0/0 on all ports
aws ec2 authorize-security-group-ingress \
  --group-id sg-abc \
  --protocol -1 \
  --port -1 \
  --cidr 0.0.0.0/0

# GOOD — only what's needed, from where it's needed
aws ec2 authorize-security-group-ingress \
  --group-id sg-rds \
  --protocol tcp \
  --port 5432 \
  --source-group sg-ecs-tasks   # only from ECS tasks security group
```

---

## Further Reading

- [AWS Documentation](https://docs.aws.amazon.com/)
- [AWS Free Tier](https://aws.amazon.com/free/)
- [AWS SDK for JavaScript v3](https://docs.aws.amazon.com/sdk-for-javascript/v3/developer-guide/)
- [AWS CLI Reference](https://awscli.amazonaws.com/v2/documentation/api/latest/index.html)
- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [CloudFormation / CDK](https://docs.aws.amazon.com/cdk/v2/guide/home.html)

---

## Summary

| Service | Use Case |
|---------|----------|
| EC2 | Long-lived VMs, full OS control |
| ECS Fargate | Containerized workloads, serverless containers |
| Lambda | Event-driven, short-lived functions, webhooks |
| S3 | Object storage, static sites, backups, file uploads |
| EBS | Persistent block storage for EC2 |
| RDS | Managed relational databases (Postgres, MySQL) |
| ElastiCache | Managed Redis/Memcached for caching and sessions |
| VPC | Isolated private network — foundation for everything |
| ALB | Layer 7 load balancing, HTTPS termination, routing |
| Route 53 | DNS management, health routing |
| CloudFront | CDN — edge caching for static assets and APIs |
| IAM | Authentication and authorization for all AWS resources |
| Secrets Manager | Secure secret storage with automatic rotation |
| CloudWatch | Metrics, logs, alarms, and dashboards |

AWS is vast, but these services form the core of most production architectures. Master them before exploring the specialized services at the edges.
