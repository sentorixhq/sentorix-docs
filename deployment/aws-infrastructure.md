> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# AWS Infrastructure Reference

**AWS Account:** ${AWS_ACCOUNT_ID}
**AWS Region:** us-east-1
**Infrastructure as Code:** Terraform 1.6+ in `sentorixhq/sentorix-infra`

---

## 1. AWS Account and Region

All Sentorix infrastructure lives in `us-east-1` (N. Virginia). This region was chosen because it has the lowest latency to most US-based customers, hosts all AWS Bedrock model endpoints used by the gateway, and provides the most complete set of AWS services.

The account is managed under the `sentorix` AWS profile locally. In CI/CD, access is via OIDC (see [cicd-pipeline.md](cicd-pipeline.md)).

---

## 2. Networking

### VPC

| Resource | Name | CIDR | Terraform resource |
|---|---|---|---|
| VPC | `sentorix-vpc` | `10.0.0.0/16` | `aws_vpc.main` |

### Subnets

| Name | Type | AZ | CIDR | Contents |
|---|---|---|---|---|
| `sentorix-public-1a` | Public | us-east-1a | `10.0.1.0/24` | ALB, NAT Gateway |
| `sentorix-public-1b` | Public | us-east-1b | `10.0.2.0/24` | ALB, NAT Gateway |
| `sentorix-private-1a` | Private | us-east-1a | `10.0.10.0/24` | ECS tasks, Lambda, VPC endpoints |
| `sentorix-private-1b` | Private | us-east-1b | `10.0.11.0/24` | ECS tasks, Lambda, VPC endpoints |

### Gateways and routing

| Resource | Name | Purpose |
|---|---|---|
| Internet Gateway | `sentorix-igw` | Provides internet access for public subnets |
| NAT Gateway (1a) | `sentorix-nat-1a` | Provides outbound-only internet access for private subnet 1a |
| NAT Gateway (1b) | `sentorix-nat-1b` | Provides outbound-only internet access for private subnet 1b |
| Public Route Table | `sentorix-public-rt` | Routes `0.0.0.0/0` to Internet Gateway |
| Private Route Table 1a | `sentorix-private-rt-1a` | Routes `0.0.0.0/0` to NAT Gateway 1a |
| Private Route Table 1b | `sentorix-private-rt-1b` | Routes `0.0.0.0/0` to NAT Gateway 1b |

**Why two NAT Gateways:** If a single NAT Gateway fails (AZ-level failure), ECS tasks in the other AZ lose internet access. Two NAT Gateways (one per AZ) provide resilience at the cost of approximately $33/month extra.

**What uses the NAT Gateway:** Only outbound calls to OpenAI and Anthropic APIs. All AWS service calls use VPC endpoints (see Section 9) and do not go through NAT.

---

## 3. Compute

### ECS Cluster

| Resource | Name | Terraform resource |
|---|---|---|
| ECS Cluster | `sentorix-cluster` | `aws_ecs_cluster.main` |

The cluster uses **Fargate** capacity only — no EC2 instances to manage. Fargate bills per second for vCPU and memory used while tasks are running.

### Gateway Task Definition

| Parameter | Value | Reason |
|---|---|---|
| Name | `sentorix-gateway` | |
| CPU | 1024 (1 vCPU) | spaCy and FastAPI under concurrent load |
| Memory | 3072 MB (3 GB) | en_core_web_lg requires ~1.5 GB; headroom for concurrency |
| Launch type | FARGATE | |
| Network mode | `awsvpc` | Required for Fargate; each task gets its own ENI |
| Image | `${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-gateway:latest` | Updated by CI/CD to SHA tag on each deploy |
| Port | 8000 | FastAPI/uvicorn default |
| Log driver | `awslogs` | Ships logs to CloudWatch |
| Log group | `/ecs/sentorix-gateway` | |
| Secrets | `ALLOWED_API_KEYS`, `OPENAI_API_KEY`, `ANTHROPIC_API_KEY` | Fetched from Secrets Manager at startup |
| Environment | All non-secret vars | Set in task definition directly |

### Dashboard Task Definition

| Parameter | Value |
|---|---|
| Name | `sentorix-dashboard` |
| CPU | 512 (0.5 vCPU) |
| Memory | 1024 MB (1 GB) |
| Port | 3000 |
| Log group | `/ecs/sentorix-dashboard` |

### ECS Services

| Service | Task Definition | Desired Count | Min Healthy % | Max % |
|---|---|---|---|---|
| `sentorix-gateway-service` | `sentorix-gateway` | 1 | 100% | 200% |
| `sentorix-dashboard-service` | `sentorix-dashboard` | 1 | 100% | 200% |

**Min healthy 100%, max 200%:** During a deployment, ECS starts the new task before stopping the old one. At peak, both tasks run simultaneously (200%), then the old task is drained and stopped. This ensures zero downtime — traffic always has at least one healthy task (100%) during the rollout.

**Desired count 1:** The MVP runs one task per service. Scaling to 2+ tasks is straightforward (change the desired count) but requires moving the in-memory rate limiter to Redis first (see [known-issues.md](../runbooks/known-issues.md)).

---

## 4. Load Balancing

### Application Load Balancer

| Resource | Name | Scheme | Subnets |
|---|---|---|---|
| ALB | `sentorix-alb` | internet-facing | `sentorix-public-1a`, `sentorix-public-1b` |

**HTTPS listener (port 443):**
- Certificate: ACM wildcard cert for `*.sentorix.io`
- Default action: forward to gateway target group
- Rule 1: `Host is app.sentorix.io` → forward to dashboard target group
- Rule 2: `Host is api.sentorix.io` → forward to gateway target group

**HTTP listener (port 80):**
- Redirect all traffic to HTTPS (301)

### Target Groups

| Name | Port | Protocol | Health Check Path | Healthy Threshold |
|---|---|---|---|---|
| `sentorix-gateway-tg` | 8000 | HTTP | `/v1/health` | 2 consecutive 200s |
| `sentorix-dashboard-tg` | 3000 | HTTP | `/` | 2 consecutive 200/307s |

Health checks run every 30 seconds. An unhealthy task is removed from rotation after 2 consecutive failures. ECS replaces the failed task automatically.

### WAF

| Resource | Name | Rule Groups |
|---|---|---|
| WAF Web ACL | `sentorix-waf` | AWS-AWSManagedRulesCommonRuleSet, AWS-AWSManagedRulesKnownBadInputsRuleSet |

The WAF ACL is associated with the ALB. Managed rule groups block common attack patterns (SQL injection, XSS, known bad user agents) before requests reach the application.

---

## 5. Container Registry

### ECR Repositories

| Repository | URI | Purpose |
|---|---|---|
| `sentorix-gateway` | `${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-gateway` | Gateway Docker images |
| `sentorix-dashboard` | `${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-dashboard` | Dashboard Docker images |

### Lifecycle Policies

Both repositories have lifecycle policies that:
- Retain the 10 most recent images tagged with a git SHA
- Delete untagged images (dangling layers) after 1 day
- Delete images older than the 10 most recent

This prevents unbounded storage growth while keeping enough history for rollbacks.

---

## 6. Audit Pipeline

### SQS Queue

| Resource | Name | Type | Retention | DLQ |
|---|---|---|---|---|
| Audit Queue | `sentorix-audit-queue` | Standard | 14 days | `sentorix-audit-dlq` |
| Dead Letter Queue | `sentorix-audit-dlq` | Standard | 14 days | — |

Messages that fail Lambda processing 3 times are moved to the DLQ. A CloudWatch alarm triggers when DLQ depth exceeds 0 — this indicates the Lambda function is failing to process events.

### Lambda Function

| Resource | Name | Runtime | Memory | Timeout |
|---|---|---|---|---|
| Audit Worker | `sentorix-audit-worker` | Python 3.12 | 256 MB | 30 seconds |

The Lambda is triggered by SQS events in batches of up to 10 messages. It processes each message by writing to S3 and DynamoDB. If a batch partially fails, failed messages are returned to SQS for retry.

**Why Lambda and not a container:** The audit workload is bursty (high volume during peak hours, near-zero at night) and the processing logic is simple (deserialize → write to S3 → write to DynamoDB). Lambda's per-invocation billing and automatic scaling make it significantly cheaper and simpler to operate than a continuously running ECS service.

### S3 Audit Bucket

| Parameter | Value |
|---|---|
| Bucket name | `sentorix-audit-${AWS_ACCOUNT_ID}-us-east-1` |
| Versioning | Enabled |
| Object Lock | COMPLIANCE mode |
| Retention period | 2555 days (7 years) |
| Server-side encryption | AES-256 (SSE-S3) |
| Public access | Fully blocked |
| Access logging | Enabled (to a separate logs bucket) |

**Object key structure:**
```
{year}/{month}/{day}/{request-id}.json
```
Example: `2026/03/26/req_abc123.json`

**Why 7 years:** This matches common enterprise compliance requirements (SOC 2, HIPAA, financial regulations). Objects cannot be deleted before this retention expires — not even by root.

### DynamoDB Audit Table

| Parameter | Value |
|---|---|
| Table name | `sentorix-audit-events` |
| Billing mode | Pay-per-request (on-demand) |
| Partition key | `tenant_id` (String) |
| Sort key | `timestamp#request_id` (String) |
| TTL | None (no automatic expiry) |
| Point-in-time recovery | Enabled |

**GSI (Global Secondary Index):**
- `user-id-index`: Partition key `user_id`, Sort key `timestamp#request_id`
- Enables the dashboard to query by user ID

**Why on-demand billing:** Audit writes are directly proportional to AI request volume, which varies significantly. On-demand billing avoids over-provisioning during low-traffic periods and scales automatically during bursts.

---

## 7. Security

### Secrets Manager

| Secret Name | Contents | Rotation |
|---|---|---|
| `sentorix/production/api-keys` | JSON array of valid API keys (`ALLOWED_API_KEYS`) | Manual |
| `sentorix/production/openai-api-key` | OpenAI API key | Manual |
| `sentorix/production/anthropic-api-key` | Anthropic API key | Manual |

Secrets are referenced in the ECS task definition by ARN. ECS fetches the secret value at task startup and injects it as an environment variable. The secret value is never stored in the task definition itself.

### SSM Parameter Store

| Parameter | Value | Type |
|---|---|---|
| `/sentorix/production/sqs-audit-queue-url` | SQS queue URL | String |
| `/sentorix/production/s3-audit-bucket` | S3 bucket name | String |
| `/sentorix/production/dynamodb-audit-table` | DynamoDB table name | String |

Non-sensitive configuration values that need to be accessible to application code. Unlike Secrets Manager (which has a per-secret cost), SSM Standard parameters are free.

### IAM Roles

| Role | Attached to | Key permissions |
|---|---|---|
| `sentorix-gateway-task-role` | ECS Gateway task | `sqs:SendMessage` on audit queue, `secretsmanager:GetSecretValue` for gateway secrets, `ssm:GetParameter` for gateway params |
| `sentorix-dashboard-task-role` | ECS Dashboard task | `dynamodb:Query`, `dynamodb:GetItem` on audit table, `secretsmanager:GetSecretValue` for dashboard secrets |
| `sentorix-lambda-role` | Lambda audit worker | `sqs:ReceiveMessage`, `sqs:DeleteMessage`, `s3:PutObject` on audit bucket, `dynamodb:PutItem` on audit table |
| `sentorix-github-actions-role` | GitHub Actions (OIDC) | `ecr:*` on sentorix repos, `ecs:UpdateService`, `ecs:RegisterTaskDefinition`, `ecs:DescribeServices`, `ecs:DescribeTaskDefinition`, `iam:PassRole` for task roles, `s3:*` on Terraform state bucket |

All roles follow least-privilege — no `*` on service-level permissions where specific actions can be named.

---

## 8. Observability

### CloudWatch Log Groups

| Log Group | Retention | Contents |
|---|---|---|
| `/ecs/sentorix-gateway` | 30 days | FastAPI application logs, uvicorn access logs |
| `/ecs/sentorix-dashboard` | 30 days | Next.js server logs |
| `/aws/lambda/sentorix-audit-worker` | 14 days | Lambda function logs |

### CloudWatch Alarms

| Alarm | Metric | Threshold | Action |
|---|---|---|---|
| `sentorix-gateway-high-error-rate` | ECS 5xx count | > 10 per minute | SNS → email |
| `sentorix-gateway-high-latency` | ALB TargetResponseTime p99 | > 5000ms | SNS → email |
| `sentorix-audit-dlq-depth` | SQS DLQ ApproximateNumberOfMessagesVisible | > 0 | SNS → email |
| `sentorix-ecs-gateway-task-stopped` | ECS service desired vs running | Running < Desired | SNS → email |

### SNS Topic

`sentorix-alerts` — email subscriptions for on-call engineers. Add subscriptions in the AWS Console under SNS → Topics → sentorix-alerts → Create subscription.

---

## 9. VPC Endpoints

| Endpoint | Type | Monthly cost | Why it exists |
|---|---|---|---|
| S3 | Gateway | Free | Lambda writes audit logs to S3 without going through NAT |
| DynamoDB | Gateway | Free | Lambda writes to DynamoDB; Dashboard reads from DynamoDB without NAT |
| SQS | Interface | ~$7.30 | Gateway publishes audit events to SQS without internet |
| Secrets Manager | Interface | ~$7.30 | ECS tasks fetch secrets at startup without internet |
| SSM Parameter Store | Interface | ~$7.30 | ECS tasks fetch config at startup without internet |
| CloudWatch Logs | Interface | ~$7.30 | ECS containers ship logs without internet |
| CloudWatch Monitoring | Interface | ~$7.30 | ECS/Lambda publish metrics without internet |
| ECR API | Interface | ~$7.30 | ECS pulls task definition metadata without internet |
| ECR DKR | Interface | ~$7.30 | ECS pulls Docker image layers without internet |
| Bedrock | Interface | ~$7.30 | Gateway calls Bedrock models without leaving AWS network |

**Gateway endpoints** (S3, DynamoDB) are free — they are implemented as route table entries and have no hourly cost.

**Interface endpoints** each cost $0.01/hour per AZ × 2 AZs = ~$14.60/month each. With 8 interface endpoints, the total VPC endpoint cost is approximately $116.80/month.

**Why the cost is worth it:** Each NAT Gateway charges $0.045 per GB of data processed. A high-volume deployment pushing 1TB/month through NAT would cost $45 in data processing fees — plus $32.85/month for the two NAT Gateways themselves. VPC endpoints are economical once traffic volume is meaningful. More importantly, they improve security by keeping traffic within the AWS network.

---

## 10. Monthly Cost Breakdown

Current estimated cost (MVP, 1 ECS task per service, modest traffic):

| Resource | Monthly Cost |
|---|---|
| ECS Fargate — Gateway (1 vCPU, 3GB, ~730 hrs) | ~$50.00 |
| ECS Fargate — Dashboard (0.5 vCPU, 1GB, ~730 hrs) | ~$14.50 |
| ALB (fixed + LCU) | ~$18.00 |
| NAT Gateways (2x, light traffic) | ~$35.00 |
| VPC Interface Endpoints (8x, 2 AZs) | ~$116.80 |
| ECR storage (two repos, ~5GB each) | ~$1.00 |
| S3 (audit bucket, small volume) | ~$2.00 |
| DynamoDB (on-demand, light traffic) | ~$1.00 |
| SQS (small message volume) | ~$0.50 |
| Lambda (small invocation count) | ~$0.10 |
| Secrets Manager (3 secrets) | ~$1.50 |
| CloudWatch Logs + Alarms | ~$5.00 |
| WAF | ~$6.00 |
| **Total** | **~$251/month** |

> Actual costs vary with traffic. The above reflects an early MVP with a single customer.

### Cost savings already achieved

The VPC endpoint configuration saves significant NAT Gateway data processing fees as audit event volume grows. The ECS task sizing (3GB for gateway, not more) was the minimum that accommodates spaCy reliably — using 4GB unnecessarily would add ~$7/month per instance.

The Lambda audit worker saves approximately $35/month versus running an equivalent ECS Fargate task continuously for the same workload.
