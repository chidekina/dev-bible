# Cloud Security

## Overview

Cloud security failures almost always follow the same patterns: overly permissive IAM roles, exposed credentials, unencrypted data, and public resources that should be private. The tools to prevent all of these exist and are well-documented — the problem is teams that skip them under deadline pressure.

This chapter covers the security controls every cloud deployment needs. The principles apply across AWS, Azure, and GCP. We provide code examples and CLI commands for each where they differ.

---

## Prerequisites

- AWS Core Services (`aws-core-services.md`) or Azure/GCP equivalent
- Basic understanding of IAM concepts (users, roles, policies)
- Node.js/TypeScript application running on cloud infrastructure

---

## Core Concepts

### The Shared Responsibility Model

```
Cloud Provider responsible for:
  Physical security, hardware, hypervisor, network infrastructure,
  managed service patching (RDS OS, ECS control plane, etc.)

YOU are responsible for:
  IAM configuration, network configuration (security groups, firewall rules),
  data encryption, application security, secrets management,
  OS patching (for VMs you manage), compliance configuration
```

The most common misunderstanding: "It's in AWS so it's secure." AWS secures the infrastructure. You secure the configuration. A misconfigured S3 bucket with public read access is your problem, not AWS's.

### Defense in Depth

No single control is sufficient. Effective cloud security layers multiple controls:

```
Layer 1: Identity (IAM, MFA, short-lived credentials)
Layer 2: Network (VPC, security groups, private subnets)
Layer 3: Data encryption (at rest and in transit)
Layer 4: Secret management (no plaintext secrets in code/env/logs)
Layer 5: Audit and detection (CloudTrail, Defender for Cloud, Security Command Center)
Layer 6: Application security (input validation, auth, rate limiting)
```

If one layer is breached, the others limit the blast radius.

### Principle of Least Privilege

Every IAM principal (user, role, service account) should have exactly the permissions needed to do its job — no more.

```
BAD:  Lambda function has AdministratorAccess
GOOD: Lambda function has s3:GetObject on specific bucket + secretsmanager:GetSecretValue on specific secret

BAD:  RDS accessible from 0.0.0.0/0
GOOD: RDS accessible only from application security group on port 5432

BAD:  All engineers have production console access
GOOD: Production access requires separate approval + time-limited credentials
```

---

## Hands-On Examples

### Identity and Access Management

**AWS: Minimum permissions for a Node.js API on ECS Fargate:**

```bash
# Create task role with only what the app needs
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

# Grant specific permissions
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

**GCP: Minimum roles for a Cloud Run service:**

```bash
# Create dedicated service account
gcloud iam service-accounts create myapp-api-sa \
  --display-name "MyApp API Service Account"

# Grant only what the service needs
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

# Attach to service
gcloud run deploy myapp-api \
  --service-account myapp-api-sa@myapp-prod.iam.gserviceaccount.com \
  ...
```

**Azure: Managed Identity + RBAC:**

```bash
# Enable system-assigned managed identity
az containerapp identity assign \
  --name myapp-api \
  --resource-group myapp-prod-rg

PRINCIPAL_ID=$(az containerapp identity show \
  --name myapp-api \
  --resource-group myapp-prod-rg \
  --query principalId -o tsv)

# Grant Key Vault Secrets User (read secrets)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Key Vault Secrets User" \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.KeyVault/vaults/myapp-kv

# Grant Storage Blob Data Contributor (read/write blobs)
az role assignment create \
  --assignee $PRINCIPAL_ID \
  --role "Storage Blob Data Contributor" \
  --scope /subscriptions/SUB/resourceGroups/RG/providers/Microsoft.Storage/storageAccounts/myappstorage/blobServices/default/containers/uploads
```

### Network Security

**AWS: Security group rules — principle of least privilege:**

```bash
# ALB security group: only accept internet traffic on 80/443
aws ec2 create-security-group \
  --group-name alb-sg \
  --description "Internet-facing ALB" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 80 --cidr 0.0.0.0/0
aws ec2 authorize-security-group-ingress --group-id sg-alb --protocol tcp --port 443 --cidr 0.0.0.0/0

# App security group: only accept traffic from ALB (no direct internet access)
aws ec2 create-security-group \
  --group-name app-sg \
  --description "Application servers" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-app \
  --protocol tcp \
  --port 3000 \
  --source-group sg-alb   # only from ALB, not internet

# DB security group: only accept traffic from app servers
aws ec2 create-security-group \
  --group-name db-sg \
  --description "Database" \
  --vpc-id vpc-abc123

aws ec2 authorize-security-group-ingress \
  --group-id sg-db \
  --protocol tcp \
  --port 5432 \
  --source-group sg-app   # only from app servers

# NEVER do this:
# --cidr 0.0.0.0/0 on a database security group
```

**GCP: VPC firewall rules:**

```bash
# Deny all ingress by default (GCP's default policy)
# Add explicit allow rules only

# Allow HTTPS from internet to load balancer (health checks from Google ranges)
gcloud compute firewall-rules create allow-https-lb \
  --network myapp-vpc \
  --direction INGRESS \
  --action ALLOW \
  --rules tcp:443 \
  --source-ranges 0.0.0.0/0 \
  --target-tags load-balancer

# Allow app servers to access database
gcloud compute firewall-rules create allow-db-from-app \
  --network myapp-vpc \
  --direction INGRESS \
  --action ALLOW \
  --rules tcp:5432 \
  --source-tags app-server \
  --target-tags database
```

### Secret Management — Never in Code

The #1 cloud security mistake: secrets in source code, environment variables, or container images.

```typescript
// BAD — do any of these and the secret is compromised when code is shared/deployed
const DB_URL = 'postgres://admin:MyPassw0rd@rds-host.example.com:5432/app';
process.env.DATABASE_URL = 'postgres://admin:MyPassw0rd@rds-host.example.com:5432/app';

// GOOD — retrieve at runtime from a managed secret store
// The application has an IAM role/managed identity that grants access to the secret
// The secret itself is never in code or environment variables in plaintext

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

**Secret scanning in CI/CD:**

```bash
# Install git-secrets or truffleHog to prevent committing secrets
pip install truffleHog3

# Scan current repo
trufflehog git file://. --since-commit HEAD~10 --only-verified

# Add to pre-commit hook
cat > .git/hooks/pre-commit << 'EOF'
#!/bin/bash
trufflehog git file://. --since-commit HEAD --only-verified --fail
EOF
chmod +x .git/hooks/pre-commit

# Or use GitHub's secret scanning (free for public repos, paid for private)
# It will block pushes that contain detected secrets
```

**Secret rotation automation (AWS):**

```bash
# Enable automatic rotation for a database secret
aws secretsmanager rotate-secret \
  --secret-id myapp/database-url \
  --rotation-lambda-arn arn:aws:lambda:us-east-1:123456789012:function:SecretsManagerRDSRotation \
  --rotation-rules AutomaticallyAfterDays=30

# The Lambda rotates the password in both Secrets Manager and RDS
# Your application reads the secret at startup — no code changes needed
```

### Encryption at Rest and in Transit

**AWS: Enforce encryption everywhere:**

```bash
# S3: enforce server-side encryption
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

# S3: block public access at account level (do this on day one)
aws s3control put-public-access-block \
  --account-id 123456789012 \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

# RDS: enable encryption at rest (must set at creation time)
aws rds create-db-instance \
  --db-instance-identifier myapp-prod \
  --storage-encrypted \                       # AES-256 encryption
  --kms-key-id arn:aws:kms:us-east-1:...:key/abc-123 \
  ...

# ECS: enforce HTTPS only on ALB
aws elbv2 create-listener \
  --load-balancer-arn arn:... \
  --protocol HTTP \
  --port 80 \
  --default-actions \
    Type=redirect,RedirectConfig="{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}"

# In Node.js: force TLS for database connections
const prisma = new PrismaClient({
  datasources: {
    db: {
      url: process.env.DATABASE_URL + '?sslmode=require',
    },
  },
});
```

**Azure: encryption defaults:**

```bash
# Azure encrypts all storage at rest by default (AES-256)
# Enforce HTTPS-only on storage account
az storage account update \
  --name myappstorageprod \
  --resource-group myapp-prod-rg \
  --https-only true \
  --min-tls-version TLS1_2

# PostgreSQL: SSL enforcement
az postgres flexible-server update \
  --name myapp-db \
  --resource-group myapp-prod-rg \
  --ssl-enforcement Enabled
```

**GCP: encryption and TLS:**

```bash
# Cloud Storage: encrypted by default (Google-managed keys)
# For customer-managed keys (CMEK):
gcloud kms keyrings create myapp-keyring --location us-central1
gcloud kms keys create myapp-storage-key \
  --keyring myapp-keyring \
  --location us-central1 \
  --purpose encryption

gcloud storage buckets update gs://myapp-uploads-prod \
  --default-encryption-key projects/myapp-prod/locations/us-central1/keyRings/myapp-keyring/cryptoKeys/myapp-storage-key

# Cloud SQL: SSL is required by default for Flexible Server
# Cloud Run: HTTPS by default, HTTP disabled
```

### Audit Logging

Know what happened, who did it, and when. Essential for incident response.

```bash
# AWS: CloudTrail captures all API calls
aws cloudtrail create-trail \
  --name myapp-audit-trail \
  --s3-bucket-name myapp-cloudtrail-logs \
  --include-global-service-events \
  --is-multi-region-trail \
  --enable-log-file-validation

aws cloudtrail start-logging --name myapp-audit-trail

# Query CloudTrail for specific actions (using CloudWatch Logs Insights)
aws logs start-query \
  --log-group-name aws-cloudtrail-logs \
  --start-time $(date -d '24 hours ago' +%s) \
  --end-time $(date +%s) \
  --query-string '
    fields @timestamp, userIdentity.principalId, eventName, requestParameters.bucketName
    | filter eventName = "DeleteBucket" or eventName = "PutBucketPublicAccessBlock"
    | sort @timestamp desc
  '

# GCP: Cloud Audit Logs are always on for data access
gcloud logging read \
  'logName="projects/myapp-prod/logs/cloudaudit.googleapis.com%2Factivity" AND
   protoPayload.methodName="storage.buckets.delete"' \
  --project myapp-prod --limit 50

# Azure: Activity Log for all control plane actions
az monitor activity-log list \
  --resource-group myapp-prod-rg \
  --start-time 2024-01-01 \
  --caller admin@company.com \
  --output table
```

### Container Security

```dockerfile
# Dockerfile: security best practices
FROM node:22-alpine AS builder
WORKDIR /app
COPY package*.json ./
RUN npm ci --only=production
COPY . .
RUN npm run build

# Distroless final image: no shell, no package manager, minimal attack surface
FROM gcr.io/distroless/nodejs22-debian12
WORKDIR /app
COPY --from=builder /app/dist ./dist
COPY --from=builder /app/node_modules ./node_modules

# Run as non-root user (distroless default is root — override)
USER nonroot

EXPOSE 3000
CMD ["dist/index.js"]
```

```bash
# Scan container images for vulnerabilities
# Using Trivy (open source)
docker pull aquasec/trivy
docker run --rm -v /var/run/docker.sock:/var/run/docker.sock \
  aquasec/trivy image myapp:latest \
  --exit-code 1 \
  --severity HIGH,CRITICAL

# AWS ECR scan on push
aws ecr put-image-scanning-configuration \
  --repository-name myapp \
  --image-scanning-configuration scanOnPush=true

# Check scan results
aws ecr describe-image-scan-findings \
  --repository-name myapp \
  --image-id imageTag=latest \
  --query 'imageScanFindings.findings[?severity==`CRITICAL` || severity==`HIGH`]'

# GCP Artifact Registry: automatic vulnerability scanning
gcloud artifacts repositories update myapp \
  --project myapp-prod \
  --location us-central1 \
  --format docker \
  # scanning enabled by default on Artifact Registry
```

---

## Common Patterns & Best Practices

### Security Checklist for New Deployments

```
IAM:
[ ] Create dedicated service accounts/roles per service
[ ] No AdministratorAccess or Owner for application roles
[ ] MFA enabled for all human IAM users
[ ] Root/global admin accounts have hardware MFA keys
[ ] Access keys rotated every 90 days (better: use roles, no access keys)

Network:
[ ] No databases or internal services in public subnets
[ ] Security groups follow least-privilege (specific ports, specific sources)
[ ] VPC flow logs enabled for incident investigation
[ ] WAF in front of public-facing load balancers

Data:
[ ] All storage encrypted at rest (S3 SSE, RDS encryption, disk encryption)
[ ] All connections use TLS (database, internal service-to-service, client-to-API)
[ ] S3/Blob public access blocked at account level
[ ] Backup encryption enabled

Secrets:
[ ] No secrets in code, environment variables, or Docker images
[ ] All secrets in managed secret stores (Secrets Manager, Key Vault, Secret Manager)
[ ] Secret rotation automated or scheduled
[ ] Git history scanned for leaked secrets

Audit:
[ ] CloudTrail/Activity Log/Cloud Audit Logs enabled in all regions
[ ] Logs retained for 90+ days (compliance) in immutable storage
[ ] Alerts on critical actions (root login, policy changes, security group changes)
[ ] GuardDuty/Defender for Cloud/Security Command Center enabled

Application:
[ ] HTTPS enforced (HTTP redirects to HTTPS)
[ ] Security headers set (HSTS, CSP, X-Frame-Options)
[ ] Input validation with Zod or similar at API boundary
[ ] Rate limiting on authentication endpoints
[ ] Dependency scanning (npm audit, Dependabot)
```

### MFA for Cloud Console Access

```bash
# AWS: enforce MFA via IAM policy
# Attach this policy to IAM users — they can do nothing until they authenticate with MFA
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

## Anti-Patterns to Avoid

### Wildcard IAM Permissions

```json
// BAD — grants every action on every resource
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// BAD — common "close enough" mistake: wildcard resource
{
  "Effect": "Allow",
  "Action": "s3:*",
  "Resource": "*"
}

// GOOD — specific actions on specific resource
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::myapp-uploads-prod/*"
}
```

### Public S3 Buckets

```bash
# Check if any buckets are public
aws s3api list-buckets --query 'Buckets[].Name' --output text | while read bucket; do
  PUBLIC=$(aws s3api get-public-access-block --bucket "$bucket" 2>/dev/null | \
    jq '.PublicAccessBlockConfiguration | to_entries | map(select(.value == false)) | length')
  if [ "$PUBLIC" -gt 0 ]; then
    echo "WARNING: $bucket has public access settings disabled"
  fi
done

# Fix: enable public access block on all buckets
aws s3api put-public-access-block \
  --bucket BUCKET_NAME \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true
```

### Long-Lived Access Keys

```bash
# Find old access keys (older than 90 days)
aws iam list-users --query 'Users[].UserName' --output text | while read user; do
  aws iam list-access-keys --user-name "$user" \
    --query "AccessKeyMetadata[?Status=='Active'].{User:'$user',Key:AccessKeyId,Created:CreateDate}" \
    --output table
done

# Rotate: create new key, update application, deactivate old key
aws iam create-access-key --user-name myuser
# Update wherever the key is used
aws iam update-access-key --access-key-id OLD_KEY --status Inactive --user-name myuser
# After verifying new key works:
aws iam delete-access-key --access-key-id OLD_KEY --user-name myuser
```

### Not Checking Dependencies for Vulnerabilities

```bash
# In CI/CD, fail the build on high/critical vulnerabilities
npm audit --audit-level=high

# GitHub Dependabot: enable in .github/dependabot.yml
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

## Debugging & Troubleshooting

### "Access Denied" — Finding Why

**AWS:**
```bash
# Use IAM Policy Simulator to test permissions without actually doing the action
aws iam simulate-principal-policy \
  --policy-source-arn arn:aws:iam::123456789012:role/myapp-task-role \
  --action-names s3:GetObject \
  --resource-arns arn:aws:s3:::myapp-uploads-prod/test.txt

# Check CloudTrail for the denied action
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

# Check audit logs for denied actions
gcloud logging read \
  'protoPayload.status.code=7 AND
   protoPayload.authenticationInfo.principalEmail="myapp-api-sa@myapp-prod.iam.gserviceaccount.com"' \
  --project myapp-prod --limit 20
```

### Security Misconfiguration Detection

```bash
# AWS Security Hub: aggregates findings from GuardDuty, Inspector, Config
aws securityhub get-findings \
  --filters '{"SeverityLabel":[{"Value":"CRITICAL","Comparison":"EQUALS"}]}' \
  --query 'Findings[].{Title:Title,Resource:Resources[0].Id}' \
  --output table

# Azure Defender for Cloud (formerly Security Center)
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

## Real-World Scenarios

### Scenario 1: Responding to a Leaked AWS Access Key

```bash
# 1. Immediately deactivate the key
aws iam update-access-key \
  --access-key-id LEAKED_KEY_ID \
  --status Inactive \
  --user-name affected-user

# 2. Investigate what happened with it
aws cloudtrail lookup-events \
  --lookup-attributes AttributeKey=AccessKeyId,AttributeValue=LEAKED_KEY_ID \
  --start-time $(date -d '30 days ago' --iso-8601) \
  --query 'Events[].{Time:EventTime,Action:EventName,Source:SourceIPAddress,Resource:Resources[0].ResourceName}' \
  --output table

# 3. Check for unauthorized resources
aws ec2 describe-instances --query 'Reservations[].Instances[].{ID:InstanceId,State:State.Name,Region:Placement.AvailabilityZone}'
aws s3api list-buckets
aws iam list-users  # check for unauthorized users created

# 4. Rotate all secrets that the compromised principal could have accessed
# 5. Delete the key permanently
aws iam delete-access-key --access-key-id LEAKED_KEY_ID --user-name affected-user

# 6. File a postmortem and improve:
#    - Switch to IAM roles (no access keys at all)
#    - Add git secret scanning to pre-commit hooks
#    - Add GitHub Secret Scanning for the repo
```

### Scenario 2: Hardening an Existing Deployment

```bash
# Audit 1: find public S3 buckets
aws s3api list-buckets --query 'Buckets[].Name' --output text | \
  xargs -I{} aws s3api get-bucket-acl --bucket {} --query 'Grants[?Grantee.URI==`http://acs.amazonaws.com/groups/global/AllUsers`]'

# Audit 2: find unrestricted security groups
aws ec2 describe-security-groups \
  --query 'SecurityGroups[?IpPermissions[?IpRanges[?CidrIp==`0.0.0.0/0`] && !starts_with(FromPort, `443`)]].{ID:GroupId,Name:GroupName}' \
  --output table

# Audit 3: find RDS instances with public access
aws rds describe-db-instances \
  --query 'DBInstances[?PubliclyAccessible==`true`].{ID:DBInstanceIdentifier,Engine:Engine}' \
  --output table

# Audit 4: find Lambda functions with excessive permissions
aws lambda list-functions --query 'Functions[].FunctionArn' --output text | \
  xargs -I{} aws lambda get-function-configuration --function-name {} \
  --query '{Function:FunctionName,Role:Role}'
```

---

## Further Reading

- [AWS Security Best Practices](https://aws.amazon.com/architecture/security-identity-compliance/)
- [AWS GuardDuty — threat detection](https://aws.amazon.com/guardduty/)
- [Azure Security Benchmark](https://learn.microsoft.com/security/benchmark/azure/)
- [GCP Security Best Practices](https://cloud.google.com/security/best-practices)
- [OWASP Cloud Security Guide](https://owasp.org/www-project-cloud-native-application-security-top-10/)
- [CIS Benchmarks for AWS/Azure/GCP](https://www.cisecurity.org/cis-benchmarks)
- [truffleHog — secret scanning](https://github.com/trufflesecurity/trufflehog)

---

## Summary

| Control | AWS | Azure | GCP | Priority |
|---------|-----|-------|-----|----------|
| Least-privilege IAM | IAM roles | Managed Identity + RBAC | Service Accounts | Critical |
| Secret management | Secrets Manager | Key Vault | Secret Manager | Critical |
| Network segmentation | VPC + Security Groups | VNet + NSG | VPC + Firewall Rules | Critical |
| Encryption at rest | SSE-KMS | Default (Azure-managed) | Default (Google-managed) | Critical |
| Encryption in transit | ACM + HTTPS enforce | TLS enforce | TLS enforce | Critical |
| Public access control | S3 Block Public Access | Storage account HTTPS | Uniform bucket-level access | Critical |
| Audit logging | CloudTrail | Activity Log | Cloud Audit Logs | High |
| Threat detection | GuardDuty | Defender for Cloud | Security Command Center | High |
| Vulnerability scanning | ECR Image Scanning | Defender for Containers | Artifact Registry scanning | High |
| Secret scanning | GitHub Secret Scanning | GitHub Secret Scanning | GitHub Secret Scanning | High |
| MFA | IAM virtual MFA | Azure AD MFA | Google Workspace MFA | High |

Security is not a one-time task. Enable these controls on day one. Run security audits monthly. Respond to findings within 72 hours for critical, 7 days for high. Review IAM permissions quarterly — they accumulate and drift toward excess over time.
