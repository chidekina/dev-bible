# Cloud Concepts

## Overview

Cloud computing is the on-demand delivery of computing resources — servers, storage, databases, networking, software — over the internet with pay-as-you-go pricing. Instead of buying physical hardware and managing data centers, you rent capacity from a cloud provider and pay only for what you use.

Cloud computing fundamentally changed how software is built and operated. Startups can launch globally on day one. Teams can provision a 100-node cluster in minutes and tear it down when done. Infrastructure becomes code.

This chapter builds the conceptual foundation for working in the cloud: service models, deployment models, shared responsibility, core design principles, and the economics that drive cloud decisions.

---

## Prerequisites

- Basic understanding of networking (HTTP, DNS, ports)
- Familiarity with Linux and Docker
- No prior cloud experience required

---

## Core Concepts

### Service Models

Cloud services are delivered at different levels of abstraction:

```
Abstraction spectrum (more control ← → more managed)

IaaS                    PaaS                    SaaS
(Infrastructure)        (Platform)              (Software)
    |                       |                       |
Virtual machines        Managed databases       Gmail
Object storage          App hosting             Slack
Load balancers          Serverless functions    Figma
Networks                Container services      GitHub
    |                       |                       |
You manage:             Provider manages:       Provider manages:
OS, runtime,            OS, runtime,            Everything
app, data               scaling, patching
```

**IaaS (Infrastructure as a Service):** You get raw compute, storage, and networking. You're responsible for everything above the hypervisor: OS, security patches, runtime, your application.
- Examples: AWS EC2, Google Compute Engine, Azure VMs, DigitalOcean Droplets

**PaaS (Platform as a Service):** The provider manages the underlying infrastructure. You deploy code or containers.
- Examples: AWS Elastic Beanstalk, Google App Engine, Heroku, Railway, Render

**SaaS (Software as a Service):** Fully managed applications. You use them, not run them.
- Examples: GitHub, Datadog, Stripe, Sendgrid

**FaaS (Function as a Service / Serverless):** Deploy individual functions. No servers to manage. Scales to zero.
- Examples: AWS Lambda, Google Cloud Functions, Cloudflare Workers

### Deployment Models

| Model | Description | Use case |
|-------|-------------|----------|
| **Public cloud** | Resources shared among multiple customers (isolated) | Most applications |
| **Private cloud** | Dedicated infrastructure for one organization | Regulated industries |
| **Hybrid cloud** | Combination of public + private | Gradual migration, compliance requirements |
| **Multi-cloud** | Multiple public cloud providers | Avoid vendor lock-in, best-of-breed |

### Shared Responsibility Model

A critical concept: cloud providers secure **of** the cloud, customers secure **in** the cloud.

```
Cloud Provider Responsibility:
  ✓ Physical data center security
  ✓ Hardware maintenance
  ✓ Hypervisor security
  ✓ Managed service availability
  ✓ Compliance certifications (SOC2, ISO27001, etc.)

Your Responsibility (varies by service model):
  IaaS: OS patches, firewall rules, app security, data encryption, access management
  PaaS: App security, data, access management, API keys
  SaaS: Access management, data you put in, user permissions
```

Misconfiguration is the leading cause of cloud data breaches. An S3 bucket set to public, an overly permissive IAM role, or a security group with `0.0.0.0/0` on port 5432 — these are your responsibility.

### Regions and Availability Zones

Cloud providers operate globally distributed infrastructure:

```
Region: us-east-1 (Northern Virginia)
  ├── Availability Zone: us-east-1a
  │     └── Multiple data centers
  ├── Availability Zone: us-east-1b
  │     └── Multiple data centers
  └── Availability Zone: us-east-1c
        └── Multiple data centers

Region: eu-west-1 (Ireland)
  ├── Availability Zone: eu-west-1a
  └── Availability Zone: eu-west-1b
```

**Region:** A geographic area with multiple data centers. Choosing a region affects latency (pick close to your users) and compliance (data sovereignty laws).

**Availability Zone (AZ):** An isolated, physically separate data center within a region. Deploying across multiple AZs protects against single data center failures.

**Edge locations:** Points of presence (PoPs) used by CDNs and services like AWS CloudFront. Much more numerous than regions.

### Cloud Economics

**CapEx vs OpEx:**
- Traditional IT: large upfront capital expenditure (CapEx) for hardware
- Cloud: ongoing operational expenditure (OpEx) — pay as you go

**Pricing models:**

| Model | Description | Best for |
|-------|-------------|----------|
| On-demand | Pay per hour/second. No commitment. | Variable workloads, dev/test |
| Reserved | 1 or 3-year commitment. 30-70% discount. | Steady, predictable workloads |
| Spot/Preemptible | Bid on unused capacity. 70-90% discount. Can be interrupted. | Batch jobs, CI, stateless workloads |
| Savings Plans | Flexible commitment to spend. 30-65% discount. | AWS-specific, flexible reserved |

**Common cost drivers:**
- Compute (vCPU hours)
- Storage (GB-months)
- Data transfer (egress is almost always paid; ingress is usually free)
- Requests (API calls, per-request services)
- Support plans

---

## Hands-On Examples

### Thinking in Cloud-Native Terms

Traditional deployment:
```
Server → runs Node.js process → connects to local Postgres → serves traffic
```

Cloud-native equivalent:
```
ECS/EKS task → containerized Node.js → RDS Postgres (managed) → behind ALB
     |                                        |                       |
Scales horizontally              Automatic failover           Health checks
Auto-replace on failure          Automated backups            SSL termination
Deploy without downtime          Point-in-time recovery       WAF integration
```

### Calculating Cloud Costs

Example: small production web application on AWS

```
Component             | Size           | Monthly Cost (approx.)
----------------------|----------------|----------------------
EC2 t3.small          | 2 vCPU, 2 GB   | $15/mo (on-demand)
EBS gp3 volume        | 20 GB          | $1.60/mo
RDS db.t3.micro       | 1 vCPU, 1 GB   | $13/mo
Application Load Balancer | 1 ALB      | $16/mo + usage
Data transfer out     | ~50 GB/mo      | $4.50/mo
Route 53              | 1 hosted zone  | $0.50/mo
----------------------|----------------|----------------------
Total                 |                | ~$50/mo
```

Compare to Hetzner VPS (CX21: 2 vCPU, 4 GB): $7/mo — but no managed database, no load balancer, no auto-scaling, no managed backups.

Cloud costs money, but management time costs more. The economics favor cloud for teams that value developer velocity over infrastructure frugality.

### The 5 Rs of Cloud Migration

When moving existing applications to the cloud:

| Strategy | Description | When to use |
|----------|-------------|-------------|
| **Rehost** (Lift & Shift) | Move VMs as-is to cloud VMs | Fastest migration, minimal change |
| **Replatform** | Minor optimizations (managed DB, PaaS) | Quick wins with some cloud benefits |
| **Repurchase** | Switch to SaaS | Replace self-hosted with commercial SaaS |
| **Refactor** | Redesign as cloud-native | Maximum benefit, most work |
| **Retire** | Decommission unused systems | Cost reduction |

---

## Common Patterns & Best Practices

### Design for Failure

Cloud instances fail. Networks partition. Services become temporarily unavailable. Design for it:

```typescript
// Retry with exponential backoff
async function callExternalService<T>(fn: () => Promise<T>, maxRetries = 3): Promise<T> {
  for (let attempt = 1; attempt <= maxRetries; attempt++) {
    try {
      return await fn();
    } catch (err) {
      if (attempt === maxRetries) throw err;
      const delay = Math.min(1000 * 2 ** attempt, 30000) + Math.random() * 1000;
      await new Promise((resolve) => setTimeout(resolve, delay));
    }
  }
  throw new Error('Max retries exceeded');
}
```

### Principle of Least Privilege

Every cloud resource should have only the permissions it needs:

```json
// BAD — wildcard permissions
{
  "Effect": "Allow",
  "Action": "*",
  "Resource": "*"
}

// GOOD — specific permissions on specific resources
{
  "Effect": "Allow",
  "Action": ["s3:GetObject", "s3:PutObject"],
  "Resource": "arn:aws:s3:::my-app-uploads/*"
}
```

### Infrastructure as Code (IaC)

Never click through cloud consoles to create production infrastructure. Define it as code:

```typescript
// AWS CDK example
import * as cdk from 'aws-cdk-lib';
import * as ec2 from 'aws-cdk-lib/aws-ec2';
import * as ecs from 'aws-cdk-lib/aws-ecs';

export class AppStack extends cdk.Stack {
  constructor(scope: cdk.App, id: string, props?: cdk.StackProps) {
    super(scope, id, props);

    const vpc = new ec2.Vpc(this, 'AppVpc', { maxAzs: 2 });

    const cluster = new ecs.Cluster(this, 'Cluster', { vpc });

    new ecs.FargateService(this, 'ApiService', {
      cluster,
      taskDefinition: new ecs.FargateTaskDefinition(this, 'Task'),
      desiredCount: 2,
    });
  }
}
```

IaC benefits:
- Reproducible environments (staging = production)
- Version-controlled infrastructure changes
- Peer review for infrastructure modifications
- Disaster recovery (recreate entire stack from code)

### Immutable Infrastructure

Don't patch running servers — replace them:

```
Traditional: push code → SSH → git pull → restart app
Cloud-native: build image → push → deploy new containers → kill old containers
```

Old containers are never modified. If something breaks, roll back to the previous image.

### The 12-Factor App

Methodology for building cloud-native applications:

1. **Codebase:** One repo per app, many deploys
2. **Dependencies:** Explicitly declare (package.json), don't rely on system packages
3. **Config:** Store in environment variables, not code
4. **Backing services:** Treat databases, queues as attached resources (URL-configured)
5. **Build/release/run:** Strictly separate stages
6. **Processes:** Stateless processes — no local state
7. **Port binding:** Export services via port
8. **Concurrency:** Scale via process model
9. **Disposability:** Fast startup, graceful shutdown
10. **Dev/prod parity:** Keep environments as similar as possible
11. **Logs:** Treat logs as event streams (stdout only)
12. **Admin processes:** Run one-off tasks as one-off processes

---

## Anti-Patterns to Avoid

### Snowflake Servers

Servers that were manually configured and can't be reproduced. If the server dies, you can't recreate it. Use IaC and treat servers as cattle, not pets.

### Storing State in Application Servers

```
BAD:  User uploads file → saved to /app/uploads on EC2 instance
      When EC2 is replaced, uploads are gone

GOOD: User uploads file → saved to S3 bucket
      Application servers are stateless and replaceable
```

### Opening All Ports "Temporarily"

```
Security Group Inbound: 0.0.0.0/0 ALL TRAFFIC
"Just for debugging" → never removed → data breach
```

### Ignoring Egress Costs

Data transfer out of cloud is expensive. Architectures that move large amounts of data out of AWS (to users, to other clouds) accumulate significant costs. CDNs reduce egress costs significantly.

### Single-AZ Deployment for Production

```
BAD:  All resources in us-east-1a only
      Data center failure → complete outage

GOOD: Resources spread across us-east-1a, us-east-1b, us-east-1c
      Data center failure → transparent failover
```

---

## Debugging & Troubleshooting

### General Cloud Debugging Strategy

```
1. Check service health:
   → AWS Status: status.aws.amazon.com
   → GCP Status: status.cloud.google.com
   → Azure Status: status.azure.com

2. Check your resource health:
   → AWS: CloudWatch, Health Dashboard
   → Cost anomalies: billing console

3. Check your application logs:
   → CloudWatch Logs, Cloud Logging, Azure Monitor

4. Check network connectivity:
   → Security groups / firewall rules
   → VPC routing tables
   → DNS resolution (dig, nslookup)

5. Check IAM permissions:
   → IAM Policy Simulator (AWS)
   → "Access denied" errors usually mean missing permissions
```

### Common Cloud Errors

**Access Denied / 403:** Missing IAM permissions or incorrect resource policy. Check the action and resource in the error — look up which permission is needed.

**Connection timeout to database:** Security group missing the inbound rule for port 5432/3306 from your application's security group.

**Out of capacity in AZ:** AWS can't provision the requested instance type in that AZ. Try a different AZ or instance type.

**Service limit exceeded:** AWS has default limits (e.g., 5 VPCs per region). Request a limit increase in Service Quotas.

---

## Real-World Scenarios

### Scenario 1: Choosing Between Cloud and VPS

```
VPS is better when:
- Monthly budget < $50
- Single application, single developer
- Don't need managed services
- Full control over infrastructure
- Learning/experimentation

Cloud is better when:
- Team of 2+ developers
- Need to scale unpredictably
- Require managed databases, CDN, global edge
- Compliance requirements (SOC2, HIPAA need cloud certifications)
- Building products that sell cloud services
```

### Scenario 2: Estimating Cloud Costs Before Committing

```
1. Use cloud provider calculators:
   - AWS Pricing Calculator: calculator.aws
   - GCP Pricing Calculator: cloud.google.com/products/calculator
   - Azure Pricing Calculator: azure.microsoft.com/pricing/calculator

2. Start with minimal resources (you can scale up)

3. Set billing alerts immediately:
   AWS → Billing → Budgets → Create budget → alert at $X

4. Reserve instances after 2-3 months of stable usage
   (typically saves 30-50% vs on-demand)
```

### Scenario 3: Multi-Region for Low Latency

```
Users in Brazil: route to sa-east-1 (São Paulo)
Users in US:     route to us-east-1 (Virginia)
Users in Europe: route to eu-west-1 (Ireland)

Using: Route 53 latency-based routing
       CloudFront with multiple origins
       Global Accelerator
```

---

## Further Reading

- [AWS Well-Architected Framework](https://aws.amazon.com/architecture/well-architected/)
- [Google Cloud Architecture Center](https://cloud.google.com/architecture)
- [The 12-Factor App](https://12factor.net/)
- [Cloud Native Computing Foundation](https://www.cncf.io/)
- [AWS Pricing Calculator](https://calculator.aws/)
- [Infrastructure as Code — HashiCorp Learn](https://developer.hashicorp.com/terraform/tutorials)

---

## Summary

| Concept | Key Takeaway |
|---------|-------------|
| IaaS/PaaS/FaaS | Abstraction spectrum — more managed = less control, less ops work |
| Shared responsibility | Provider secures the cloud; you secure what's in it |
| Regions/AZs | Deploy across AZs for resilience; choose regions for latency + compliance |
| On-demand vs Reserved | On-demand for flexibility; Reserved for 30-70% savings on steady workloads |
| Least privilege | Every resource gets only the permissions it needs |
| IaC | Infrastructure defined as code — reproducible, reviewable, versioned |
| Immutable infra | Replace, don't patch — enables reliable rollbacks |
| 12-Factor | Blueprint for cloud-native application design |
| Design for failure | Assume anything can fail; build retry, circuit breakers, multi-AZ |
| Egress costs | Data out of cloud costs money — factor into architecture |

The cloud is not just "someone else's computer." It's a programmable infrastructure platform with APIs, managed services, and global reach. Understanding the economics, the responsibility model, and the design principles separates cloud practitioners from cloud users.
