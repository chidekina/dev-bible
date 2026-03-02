# Cloud Labs

> Hands-on labs to learn cloud provider services through practical, production-realistic scenarios. Labs are primarily AWS-focused (with some GCP where noted). Each lab includes cost-awareness notes — stay within free tier limits where possible, and always destroy resources when done. Prerequisites assume AWS CLI configured with appropriate IAM permissions.

---

## Lab 1 — Deploy a Containerized App to AWS ECS Fargate (Intermediate)

**Goal:** Deploy the Node.js Docker image to AWS ECS Fargate — a serverless container platform — without managing EC2 instances.

**Prerequisites:**
- AWS CLI configured (`aws sts get-caller-identity` works).
- Docker image pushed to Amazon ECR (Elastic Container Registry).
- AWS account with permissions: ECS, ECR, IAM, VPC, CloudWatch.

**Tasks:**

1. **Push to ECR:**
   ```bash
   aws ecr create-repository --repository-name myapp --region us-east-1
   aws ecr get-login-password | docker login --username AWS --password-stdin <account>.dkr.ecr.us-east-1.amazonaws.com
   docker tag myapp:local <account>.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
   docker push <account>.dkr.ecr.us-east-1.amazonaws.com/myapp:latest
   ```

2. **Create an ECS Cluster:** Fargate launch type, no EC2 capacity.

3. **Define a Task Definition:**
   - Container image: the ECR URI.
   - CPU: `256` (.25 vCPU), Memory: `512` MB.
   - Port mapping: `3000`.
   - Environment variables from AWS Secrets Manager (not plain text).
   - Log configuration: `awslogs` driver → CloudWatch log group `/ecs/myapp`.

4. **Create an ECS Service:**
   - Desired count: `2` tasks.
   - Launch type: `FARGATE`.
   - Networking: public subnets (or private with NAT), security group allowing port `3000`.
   - Attach an Application Load Balancer (ALB) targeting port `3000`.

5. **Configure the ALB:**
   - Target group health check: `GET /health`, threshold 2 healthy.
   - Listener: port 80 (add HTTPS later).

6. **Enable auto-scaling:**
   - Scale out if CPU > 70% for 3 minutes (add 1 task).
   - Scale in if CPU < 20% for 10 minutes (remove 1 task).
   - Min tasks: 2, Max tasks: 10.

**Verification:**
- [ ] `curl http://<alb-dns-name>/health` returns 200.
- [ ] ECS console shows 2 tasks in `RUNNING` state.
- [ ] CloudWatch log group `/ecs/myapp` has log entries from both tasks.
- [ ] Stopping one task manually: ECS automatically starts a replacement within 30 seconds.
- [ ] Generating load (Apache Bench) triggers the scale-out policy and adds a task.
- [ ] `aws ecs describe-services` shows `desiredCount` and `runningCount` equal.

**Cost note:** 2 Fargate tasks (.25 vCPU, 512MB) + ALB ≈ $25/month. Destroy after the lab. Free tier does not cover Fargate.

---

## Lab 2 — Static Site Hosting with S3 and CloudFront (Beginner)

**Goal:** Host a static website on S3 and serve it globally through CloudFront CDN with HTTPS and a custom domain.

**Prerequisites:**
- A static site build (HTML/CSS/JS in a `dist/` folder — use `vite build` on any React app).
- A domain in Route 53 (or an existing domain you can update NS records for).
- ACM certificate for your domain in `us-east-1` (required for CloudFront).

**Tasks:**

1. **Create an S3 bucket:**
   - Bucket name: `myapp-static-<account-id>`.
   - Block all public access: ON (CloudFront will access it, not the public directly).
   - Enable versioning.

2. **Upload the static build:**
   ```bash
   aws s3 sync dist/ s3://myapp-static-<account-id>/ --delete
   ```

3. **Create a CloudFront distribution:**
   - Origin: the S3 bucket using an Origin Access Control (OAC) — not a public bucket URL.
   - Default root object: `index.html`.
   - Custom error pages: `403` and `404` → `index.html` with status `200` (required for SPA routing).
   - Price class: `PriceClass_100` (US, Canada, Europe only — cheapest).
   - Alternate domain name: `app.yourdomain.com`.
   - SSL certificate: ACM certificate from `us-east-1`.

4. **Update the S3 bucket policy** to allow only the CloudFront OAC to read objects.

5. **Configure Route 53:** Create an `A` alias record for `app.yourdomain.com` pointing to the CloudFront distribution.

6. **Add cache behaviors:**
   - `/assets/*`: Cache-Control `max-age=31536000` (1 year — assets have hashed filenames).
   - `index.html`: Cache-Control `no-cache` (always fresh).
   - Set these as CloudFront response headers policies, not in S3 metadata.

7. **Invalidate the cache after a deploy:**
   ```bash
   aws cloudfront create-invalidation --distribution-id <id> --paths "/*"
   ```

**Verification:**
- [ ] `curl https://app.yourdomain.com` returns the site with HTTP 200.
- [ ] `curl http://app.yourdomain.com` redirects to HTTPS (CloudFront default behavior).
- [ ] SSL Labs test shows A rating.
- [ ] `curl -I https://app.yourdomain.com/nonexistent-route` returns 200 (SPA fallback working).
- [ ] Direct S3 URL access is blocked: `curl https://myapp-static-<id>.s3.amazonaws.com/index.html` returns 403.
- [ ] After re-deploying with `sync --delete`, a CloudFront invalidation clears stale assets.
- [ ] `x-cache: Hit from cloudfront` appears in response headers on the second request.

**Cost note:** S3 + CloudFront is in the free tier for low traffic. ACM certificates are free.

---

## Lab 3 — RDS PostgreSQL with Read Replica (Intermediate)

**Goal:** Provision an Amazon RDS PostgreSQL primary instance, create a read replica, configure the application to use both, and test failover.

**Prerequisites:**
- AWS permissions: RDS, VPC, EC2 (for security groups).
- The Node.js application from previous labs, configured with Prisma or `pg` driver.
- A VPC with at least two private subnets in different AZs.

**Tasks:**

1. **Create an RDS subnet group** spanning at least two AZs.

2. **Launch the primary RDS instance:**
   - Engine: PostgreSQL 15.
   - Instance class: `db.t3.micro` (free tier eligible).
   - Storage: 20 GB gp2, with storage autoscaling enabled.
   - Multi-AZ: enabled (for production HA — note: costs 2x).
   - Security group: allow port 5432 only from the application security group.
   - Parameter group: set `log_min_duration_statement = 1000` (log slow queries > 1s).

3. **Create a read replica** from the primary in a different AZ.

4. **Configure the application** with two connection strings:
   - `DATABASE_URL`: primary (for writes).
   - `DATABASE_READ_URL`: read replica (for read-heavy queries like reports and lists).
   - In the Prisma or query layer: route read queries to the replica explicitly.

5. **Test the replica:** Write a record to the primary and read it from the replica. Measure replication lag with:
   ```sql
   SELECT now() - pg_last_xact_replay_timestamp() AS replication_lag;
   ```

6. **Simulate primary failure:** Use the AWS console to "Reboot with failover". Observe the Multi-AZ standby promote and the DNS endpoint switch.

7. **Enable Performance Insights** on the primary and identify the top SQL statements by load.

**Verification:**
- [ ] Application connects to both primary and replica without error.
- [ ] Replication lag stays below 1 second under normal load.
- [ ] After a Multi-AZ failover, the application reconnects automatically within 60 seconds (DNS propagation).
- [ ] `SHOW server_type;` — or checking via `pg_is_in_recovery()` — confirms the replica is in read-only mode.
- [ ] A direct `INSERT` to the replica endpoint returns an error: "cannot execute INSERT in a read-only transaction".
- [ ] CloudWatch metric `ReadLatency` for the replica stays below 10ms.
- [ ] RDS Enhanced Monitoring shows CPU/memory/connections per second in the console.

**Cost note:** `db.t3.micro` single-AZ is free tier. Multi-AZ doubles the cost to ~$28/month. Read replica is an additional instance (~$14/month). Destroy after the lab.

---

## Lab 4 — Deploy to Google Cloud Run (Intermediate)

**Goal:** Deploy the containerized Node.js application to Google Cloud Run — a fully managed, autoscaling serverless container service — and configure it for production use.

**Prerequisites:**
- Google Cloud account with billing enabled.
- `gcloud` CLI installed and configured (`gcloud auth login`).
- Docker image built and ready to push.

**Tasks:**

1. **Enable required APIs:**
   ```bash
   gcloud services enable run.googleapis.com artifactregistry.googleapis.com secretmanager.googleapis.com
   ```

2. **Push to Artifact Registry:**
   ```bash
   gcloud artifacts repositories create myapp --repository-format=docker --location=us-central1
   docker tag myapp:local us-central1-docker.pkg.dev/<project>/myapp/myapp:latest
   gcloud auth configure-docker us-central1-docker.pkg.dev
   docker push us-central1-docker.pkg.dev/<project>/myapp/myapp:latest
   ```

3. **Store secrets in Secret Manager:**
   ```bash
   echo -n "postgres://..." | gcloud secrets create DATABASE_URL --data-file=-
   ```

4. **Deploy to Cloud Run:**
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

5. **Configure a custom domain:** Map your domain to the Cloud Run service via the Cloud Run domain mapping UI.

6. **Set up traffic splitting** for a canary deployment:
   ```bash
   gcloud run services update-traffic myapp \
     --to-revisions myapp-00002-xxx=10,myapp-00001-yyy=90
   ```

7. **Configure Cloud Run concurrency and CPU allocation:** Set `--cpu-throttling` to allocate CPU only during request handling (cost optimization).

**Verification:**
- [ ] `gcloud run services describe myapp --region us-central1` shows status `READY`.
- [ ] `curl https://<cloud-run-url>/health` returns 200.
- [ ] Secrets are mounted as environment variables — verify with `gcloud run services describe` that no plaintext secrets appear in the config.
- [ ] After deploying a new revision, traffic split shows 10% going to the new revision.
- [ ] `gcloud run revisions list` shows both revisions active.
- [ ] Scaling: 0 instances at rest (min-instances=0 to test cold start), scales to multiple under load.
- [ ] Cloud Logging shows structured JSON logs from the application.

**Cost note:** Cloud Run has a generous free tier: 2 million requests/month, 360,000 vCPU-seconds, 180,000 GB-seconds. Most labs fit within free tier.

---

## Lab 5 — Set Up AWS Billing Alerts and Cost Management (Beginner)

**Goal:** Configure AWS billing alerts, budgets, and cost allocation tags so you are never surprised by an unexpected bill.

**Prerequisites:**
- AWS root account or an IAM user with billing permissions.
- Billing alerts must be enabled in the root account's billing preferences.

**Tasks:**

1. **Enable billing alerts** in AWS Billing → Billing Preferences → check "Receive Billing Alerts".

2. **Create CloudWatch billing alarms** (billing metrics are only in `us-east-1`):
   - Alarm 1: `EstimatedCharges` > $5 → SNS notification to your email.
   - Alarm 2: `EstimatedCharges` > $20 → second notification (escalation).
   - Alarm 3: `EstimatedCharges` > $50 → "panic" notification.

3. **Create an AWS Budget** via Budgets console:
   - Monthly cost budget: $10 limit.
   - Alert at 80% actual ($8) and 100% actual ($10).
   - Alert at 100% forecasted spend.
   - Notification: email.

4. **Enable Cost Allocation Tags:**
   - Tag all existing resources with `Project=dev-bible-labs` and `Environment=dev`.
   - In Cost Explorer, activate the `Project` tag as a cost allocation tag.

5. **Explore Cost Explorer:**
   - View last 30 days of spend grouped by service.
   - Identify the top 3 most expensive services.
   - Create a saved report for "Monthly EC2 spend by region".

6. **Set up AWS Cost Anomaly Detection:**
   - Create a monitor for all AWS services.
   - Alert threshold: $5 individual anomaly.
   - Notification via SNS email.

7. **Review Trusted Advisor** (basic tier):
   - Note any "Service Limits" warnings.
   - Note any underutilized EC2 instances.

**Verification:**
- [ ] `aws cloudwatch describe-alarms --alarm-names "billing-alert-5"` shows the alarm in `OK` state.
- [ ] Budget is visible in the Budgets console with the correct thresholds.
- [ ] Cost Explorer shows spend grouped by `Project` tag (may take 24h for tags to appear).
- [ ] Anomaly Detection monitor is active.
- [ ] You receive a test alarm notification by temporarily lowering the threshold below current spend.

---

## Lab 6 — IAM Least Privilege Audit (Intermediate)

**Goal:** Audit an AWS account's IAM configuration, identify over-privileged users and roles, and remediate by applying least-privilege policies.

**Prerequisites:**
- AWS IAM permissions: `iam:*`, `access-analyzer:*`.
- An AWS account with at least a few IAM users and roles to audit.

**Tasks:**

1. **Generate a Credential Report:**
   ```bash
   aws iam generate-credential-report
   aws iam get-credential-report --output text --query Content | base64 -d > cred-report.csv
   ```
   Identify: users with access keys unused for > 90 days, users with no MFA, root account access key existence.

2. **Enable IAM Access Analyzer:**
   ```bash
   aws accessanalyzer create-analyzer --analyzer-name myaccount-analyzer --type ACCOUNT
   ```
   Review findings: external access to S3 buckets, KMS keys, Lambda functions.

3. **Review inline vs. managed policies:** Identify any users with `AdministratorAccess` or `*` actions on `*` resources. Document the list.

4. **Remediate one over-privileged user:**
   - Identify the services they actually use via `aws iam generate-service-last-accessed-details`.
   - Create a custom managed policy granting only the accessed services and actions.
   - Replace the broad policy with the custom one.
   - Verify the user can still perform their job.

5. **Enforce MFA for console users:** Create an IAM policy that denies all actions except `iam:CreateVirtualMFADevice` and `iam:EnableMFADevice` for users without MFA enabled. Attach to all console users.

6. **Rotate an access key safely:**
   - Create a new access key for an IAM user.
   - Update the application/CI using the old key to use the new key.
   - Wait 24 hours (monitor for errors).
   - Deactivate the old key. Wait 48 hours. Delete it.

7. **Set a password policy:** minimum 14 characters, require uppercase, lowercase, number, symbol, no reuse of last 12 passwords, max age 90 days.

**Verification:**
- [ ] Credential report shows no users with unused access keys > 90 days (or you've disabled them).
- [ ] IAM Access Analyzer shows no unintended external access findings.
- [ ] The remediated user can perform their required actions with the new least-privilege policy.
- [ ] The remediated user cannot perform actions outside their scope (test one explicitly).
- [ ] `aws iam get-account-password-policy` returns the configured strict policy.
- [ ] Root account has no active access keys (`CredentialReport` column: `<root_account>` — key1_active = false).

---

## Lab 7 — Serverless Function with AWS Lambda and API Gateway (Intermediate)

**Goal:** Deploy a Node.js function to AWS Lambda, expose it via API Gateway, and connect it to DynamoDB — the full serverless stack.

**Prerequisites:**
- AWS permissions: Lambda, API Gateway, DynamoDB, IAM.
- Node.js 20 runtime knowledge.
- AWS SAM CLI or the Serverless Framework installed.

**Tasks:**

1. **Create a DynamoDB table:**
   - Table name: `todos`
   - Partition key: `userId` (String)
   - Sort key: `todoId` (String)
   - Billing: on-demand (pay-per-request — free tier eligible).

2. **Write the Lambda function** (`src/handler.ts`):
   - `GET /todos` → `Query` DynamoDB for all todos of the authenticated user.
   - `POST /todos` → `PutItem` with a generated UUID.
   - `DELETE /todos/{todoId}` → `DeleteItem`.

3. **Create an IAM role** for the Lambda with minimum permissions:
   - `dynamodb:Query`, `dynamodb:PutItem`, `dynamodb:DeleteItem` on `arn:aws:dynamodb:*:*:table/todos`.
   - `logs:CreateLogGroup`, `logs:CreateLogStream`, `logs:PutLogEvents` for CloudWatch.

4. **Deploy using AWS SAM** (`template.yaml`):
   - Define the Lambda function, API Gateway trigger, and DynamoDB table in the SAM template.
   - `sam build && sam deploy --guided`

5. **Configure a Lambda layer** for shared utilities (e.g., the validation library) to reduce package size and enable layer reuse.

6. **Test locally with SAM local:**
   ```bash
   sam local start-api
   curl http://localhost:3000/todos
   ```

7. **Add a Lambda Powertools layer** for structured logging and X-Ray tracing.

**Verification:**
- [ ] `curl https://<api-id>.execute-api.us-east-1.amazonaws.com/Prod/todos` returns an empty array `[]`.
- [ ] `POST /todos` with `{ "title": "test" }` creates a record — verify in the DynamoDB console.
- [ ] CloudWatch Logs show structured JSON log entries from each invocation.
- [ ] Lambda cold start time < 2 seconds (measure via CloudWatch `Init Duration` metric).
- [ ] The Lambda IAM role cannot write to any other DynamoDB table (test with `aws dynamodb put-item --table-name other-table`).
- [ ] X-Ray trace shows the DynamoDB call as a segment within the Lambda execution.

**Cost note:** Lambda + API Gateway + DynamoDB are all free tier eligible. 1M Lambda requests/month free.

---

## Lab 8 — Object Storage Lifecycle and Glacier Archival (Beginner)

**Goal:** Configure S3 lifecycle policies to automatically transition objects to cheaper storage tiers and expire old files — a key cost optimization skill.

**Prerequisites:**
- An S3 bucket with some test objects of varying ages.
- Basic S3 knowledge.

**Tasks:**

1. **Upload test objects** with different `LastModified` dates (simulate by uploading with metadata):
   ```bash
   aws s3 cp testfile.log s3://mybucket/logs/2024/testfile.log
   ```

2. **Create a lifecycle configuration** (`lifecycle.json`):
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
   Apply: `aws s3api put-bucket-lifecycle-configuration --bucket mybucket --lifecycle-configuration file://lifecycle.json`

3. **Create a second rule** for incomplete multipart uploads:
   - Abort incomplete multipart uploads after 7 days. (These silently accumulate and cost money.)

4. **Enable S3 Intelligent-Tiering** for the `assets/` prefix — let AWS automatically optimize storage class based on access patterns.

5. **Calculate storage cost savings:** Use the S3 Storage Lens dashboard to estimate what your current objects would cost in Standard vs. Intelligent-Tiering.

6. **Test a Glacier restore:** Transition an object to Glacier, then initiate a restore (`Expedited`, 1–5 minutes). Download it after restoration.

7. **Set up S3 Storage Lens** to monitor bucket metrics across the account.

**Verification:**
- [ ] `aws s3api get-bucket-lifecycle-configuration --bucket mybucket` returns the configured rules.
- [ ] S3 console shows the lifecycle rule as `Active`.
- [ ] After initiating a Glacier restore, object metadata shows `Restore: ongoing-request="true"`.
- [ ] After restoration completes, `Restore: ongoing-request="false", expiry-date="..."`.
- [ ] S3 Storage Lens dashboard shows total storage size and object count.
- [ ] The multipart upload abort rule is visible in the lifecycle configuration.

---

## Lab 9 — VPC Networking and Security Groups (Intermediate)

**Goal:** Design and build a secure AWS VPC with public and private subnets, a NAT Gateway, and properly configured security groups — the networking foundation for any production deployment.

**Prerequisites:**
- AWS permissions: VPC, EC2, NAT Gateway.
- Basic networking knowledge (CIDR, routing tables, subnets).

**Tasks:**

1. **Create a VPC** with CIDR `10.0.0.0/16`.

2. **Create subnets:**
   - Public subnet A: `10.0.1.0/24` in `us-east-1a`
   - Public subnet B: `10.0.2.0/24` in `us-east-1b`
   - Private subnet A: `10.0.10.0/24` in `us-east-1a`
   - Private subnet B: `10.0.11.0/24` in `us-east-1b`

3. **Create and attach an Internet Gateway.** Update the public subnet route table: `0.0.0.0/0 → igw-xxx`.

4. **Create a NAT Gateway** in a public subnet (requires an Elastic IP). Update the private subnet route table: `0.0.0.0/0 → nat-xxx`.

5. **Create security groups:**
   - `alb-sg`: allow inbound 80, 443 from `0.0.0.0/0`.
   - `app-sg`: allow inbound 3000 only from `alb-sg`.
   - `db-sg`: allow inbound 5432 only from `app-sg`.

6. **Launch an EC2 instance** in a private subnet. Verify it can reach the internet (via NAT) but is not reachable directly from the internet.

7. **Use VPC Flow Logs** to capture all traffic and store in CloudWatch Logs. Query: find all rejected traffic to port 22.

**Verification:**
- [ ] EC2 in private subnet: `curl https://checkip.amazonaws.com` returns the NAT Gateway's Elastic IP.
- [ ] Direct SSH to the private EC2 public IP (it has none) is impossible.
- [ ] SSH to the private EC2 via a bastion host in the public subnet works.
- [ ] RDS in `db-sg` rejects connections from outside the VPC (test from your laptop — should time out).
- [ ] VPC Flow Logs appear in CloudWatch Logs within 10 minutes of network activity.
- [ ] A CloudWatch Logs Insights query for `action = "REJECT" and dstPort = 22` returns any port-scan attempts.

**Cost note:** NAT Gateway costs ~$0.045/hour + data transfer. Remove after the lab or it will accumulate charges.

---

## Lab 10 — Multi-Environment CI/CD with Terraform and GitHub Actions (Advanced)

**Goal:** Build a complete Infrastructure-as-Code + CI/CD pipeline that provisions separate `staging` and `production` environments on AWS using Terraform workspaces, triggered automatically by Git branches.

**Prerequisites:**
- Labs 3 (CI/CD with GitHub Actions) and 9 (Terraform) completed.
- AWS account with permissions for VPC, ECS, RDS, ALB.
- A GitHub repository with the application code and Terraform configuration.

**Tasks:**

1. **Structure the Terraform config** with workspaces:
   ```
   infra/
     main.tf      # ECS, ALB, RDS, VPC resources
     variables.tf
     outputs.tf
     backend.tf   # S3 backend + DynamoDB lock table
   ```
   Use `terraform.workspace` to vary instance sizes:
   ```hcl
   instance_class = terraform.workspace == "production" ? "db.t3.small" : "db.t3.micro"
   ```

2. **CI/CD pipeline** (`.github/workflows/infra.yml`):
   - On push to `main` → `terraform workspace select staging` → `terraform plan` → (after approval) `terraform apply`.
   - On push to `release` tag → `terraform workspace select production` → `terraform plan` → require manual approval (GitHub Environment with reviewer).
   - On PR → `terraform plan` only, post the plan as a PR comment.

3. **Application pipeline** (`.github/workflows/deploy.yml`):
   - On push to `main` → build image → deploy to **staging** ECS service.
   - On push to `release` tag → deploy to **production** ECS service after staging health check passes.

4. **Configure GitHub Environments:**
   - `staging`: no approvals required.
   - `production`: require 1 reviewer approval before deploy.

5. **Smoke test step:** After each deploy, run `curl https://<env-url>/health` with retry logic (5 retries, 10s wait). Fail the pipeline if health check never passes.

6. **Implement drift detection:** Add a scheduled GitHub Action (cron: daily) that runs `terraform plan` on both workspaces and fails if there is any drift (infrastructure changed outside Terraform).

**Verification:**
- [ ] Pushing to `main` deploys automatically to staging. No human action required.
- [ ] Pushing a `v1.0.0` tag triggers the production workflow, which waits for reviewer approval.
- [ ] After approval, the production deploy runs and the smoke test passes.
- [ ] The drift detection workflow detects a manually added security group rule and fails (expected: "1 to add").
- [ ] `terraform workspace list` shows `staging` and `production` workspaces with separate state files.
- [ ] A failed smoke test (app returns 500) causes the pipeline to fail and no traffic is shifted to the broken version.
- [ ] GitHub PR comment shows the Terraform plan diff for infrastructure changes.

**Cost note:** Two full environments (ECS + RDS + ALB) cost approximately $50–$80/month. Destroy `staging` at night using a scheduled `terraform destroy` or set ECS desired count to 0.
