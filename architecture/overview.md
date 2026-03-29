# Architecture Overview

## What Sentorix Does

Sentorix is an AI Governance Gateway that enables enterprises to use large language models — from providers like OpenAI, Anthropic, and AWS Bedrock — without exposing sensitive data or violating internal compliance policies. Every AI request made by your applications passes through Sentorix first: the gateway scans the prompt for personally identifiable information, checks it against your configured policies, redacts or blocks as required, and logs every decision immutably for audit purposes. The result is a complete, auditable record of every AI interaction in your organization, with privacy and security enforced automatically — without requiring developers to change how they write AI-powered features.

---

## System Architecture Diagram

```
                          ┌─────────────────┐
                          │   Client App    │
                          │ (your software) │
                          └────────┬────────┘
                                   │ HTTPS
                                   ▼
                          ┌─────────────────┐
                          │  Cloudflare DNS │
                          │  (DNS only,     │
                          │   no proxy)     │
                          └────────┬────────┘
                                   │
                                   ▼
                   ┌───────────────────────────────┐
                   │        AWS ALB (HTTPS :443)    │
                   │       ACM wildcard cert        │
                   │            + WAF               │
                   └───────┬───────────────┬────────┘
                           │               │
              app.sentorix.io         api.sentorix.io
                           │               │
                           ▼               ▼
                  ┌──────────────┐  ┌──────────────────────┐
                  │ ECS Dashboard│  │    ECS Gateway       │
                  │  Next.js 14  │  │  FastAPI / uvicorn   │
                  │  port 3000   │  │  port 8000           │
                  │  0.5vCPU/1GB │  │  1vCPU / 3GB RAM     │
                  └──────────────┘  └──────────┬───────────┘
                                               │
                              ┌────────────────┴─────────────────┐
                              │           HOT PATH               │
                              │        (<20ms overhead)          │
                              │                                  │
                              │  asyncio.gather(                 │
                              │    PII Detection (Presidio),     │
                              │    Policy Check (Rule Engine)    │
                              │  )                               │
                              └────────────────┬─────────────────┘
                                               │
                         ┌─────────────────────┴──────────────────┐
                         │                                        │
                         ▼                                        ▼
              ┌────────────────────┐               ┌─────────────────────┐
              │    LLM Provider    │               │     SQS Queue       │
              │  OpenAI /          │               │  (fire-and-forget   │
              │  Anthropic /       │               │   audit events)     │
              │  AWS Bedrock       │               └──────────┬──────────┘
              └────────────────────┘                          │
                                                              ▼
                                                   ┌─────────────────────┐
                                                   │   Lambda Worker     │
                                                   │  (audit processor)  │
                                                   └──────────┬──────────┘
                                                              │
                                             ┌────────────────┴──────────────┐
                                             │                               │
                                             ▼                               ▼
                                    ┌────────────────┐             ┌─────────────────┐
                                    │   S3 Bucket    │             │    DynamoDB     │
                                    │  (Object Lock  │             │  (queryable     │
                                    │  COMPLIANCE,   │             │   audit index)  │
                                    │  7yr retention)│             └─────────────────┘
                                    └────────────────┘
```

---

## Hot Path vs Cold Path

### Hot Path (synchronous, <20ms overhead)

The hot path is everything that must complete before the gateway can respond to the client. It runs for every request:

1. Receive incoming request from the client
2. Authenticate the API key
3. Concurrently (via `asyncio.gather()`): run PII detection with Presidio AND evaluate the request against configured policies
4. Based on the policy decision: block the request (return 403), warn and redact (forward modified prompt), or allow (forward original prompt)
5. Forward the (possibly redacted) prompt to the LLM provider
6. Return the LLM response to the client

The `asyncio.gather()` pattern is critical here. PII detection and policy evaluation are independent operations — neither depends on the result of the other. Running them concurrently rather than sequentially cuts the synchronous overhead by roughly half (from ~16ms to ~8ms in typical workloads).

The hot path adds less than 20ms of overhead to the round-trip time. The LLM provider itself typically takes 500ms–5000ms, so the gateway overhead is not perceptible to end users.

### Cold Path (asynchronous, fire-and-forget)

The cold path handles audit logging. After the gateway has made its policy decision and begun proxying the request, it publishes an audit event to SQS without waiting for the publish to complete (fire-and-forget). A Lambda function reads from SQS and writes the event to both S3 (for long-term immutable storage) and DynamoDB (for queryable audit logs accessible to the dashboard).

**Why is audit logging fire-and-forget?**

Because audit logging must never affect the latency or reliability of the gateway. If the audit pipeline has an outage (Lambda is throttled, DynamoDB is degraded), client requests should continue working normally. Audit events accumulate in SQS until the pipeline recovers — SQS provides durability and the message retention period is 14 days, giving the pipeline ample time to catch up.

Making audit logging synchronous would mean that a DynamoDB hiccup causes every AI request to fail. That is not an acceptable trade-off.

---

## Component Descriptions

### ALB (Application Load Balancer)

**What it is:** The single entry point for all HTTPS traffic to Sentorix. It terminates TLS, applies WAF rules, and routes traffic to ECS services based on the `Host` header.

**Why it exists:** ECS tasks run in private subnets and have no public IP addresses. The ALB is the only internet-facing resource for the application tier. It also provides health checking, enabling zero-downtime rolling deployments.

**If it goes down:** All traffic to `api.sentorix.io` and `app.sentorix.io` is unavailable. AWS ALBs are managed, highly available services — they do not go down unless the entire `us-east-1` region has an incident.

---

### ECS Gateway (FastAPI)

**What it is:** The core proxy service. A Python 3.12 FastAPI application running on uvicorn in an ECS Fargate task with 1 vCPU and 3GB RAM.

**Why it has 3GB RAM:** Microsoft Presidio requires the `en_core_web_lg` spaCy model for accurate NER (Named Entity Recognition) used in PII detection. This model alone consumes approximately 1.5GB of RAM at runtime. The remaining memory is used by the FastAPI application, uvicorn workers, and headroom for concurrent request handling.

**If it goes down:** Client AI requests return 503. ECS automatically restarts failed tasks. The ALB health check detects the failure and stops routing traffic to the task within 30 seconds. Once a replacement task passes health checks, traffic resumes.

---

### ECS Dashboard (Next.js)

**What it is:** A Next.js 14 TypeScript application that displays compliance metrics, audit logs, and real-time policy decision data. Runs in an ECS Fargate task with 0.5 vCPU and 1GB RAM.

**Why it exists:** Compliance teams need a way to query audit logs, review policy decisions, and monitor PII detection rates without accessing AWS directly.

**If it goes down:** `app.sentorix.io` is unavailable. The gateway continues operating — the dashboard is purely a read-only view of data already in DynamoDB and S3.

---

### Presidio PII Detector

**What it is:** Microsoft Presidio is an open-source PII detection and anonymization library. It runs in-process within the ECS Gateway task (no separate service call required).

**Why it exists:** To detect and redact personally identifiable information in prompts before they are sent to LLM providers. Entities detected include: PERSON names, EMAIL addresses, PHONE numbers, US SSNs, credit card numbers, and more (configurable via `PII_ENTITIES` environment variable).

**Why in-process:** Making a network call to a separate PII detection service would add 10–50ms of latency per request. Running Presidio in-process adds only 5–10ms and eliminates a network dependency in the hot path.

**If it fails:** Controlled by `PII_DETECTION_ENABLED`. If the Presidio analyzer encounters an error, the gateway logs the failure and falls back to allowing the request through with a warning (configurable behavior). A complete crash of the Presidio component crashes the ECS task, which ECS restarts automatically.

---

### Policy Engine

**What it is:** An in-process rule evaluation engine that assigns a risk score to each request and makes a block/warn/allow decision.

**How it works:** Rules are evaluated against the request (prompt content, detected PII entities, client metadata). Each matching rule contributes to a risk score. The final score is compared against two thresholds:
- Score ≥ `RISK_SCORE_THRESHOLD_BLOCK`: request is blocked, 403 returned
- Score ≥ `RISK_SCORE_THRESHOLD_WARN`: request is allowed but the response includes warning headers and the prompt may be redacted
- Score < `RISK_SCORE_THRESHOLD_WARN`: request is allowed

**If it fails:** Same as Presidio — an engine crash crashes the ECS task.

---

### SQS Audit Queue

**What it is:** An AWS SQS standard queue that receives audit events from the gateway asynchronously.

**Why it exists:** Decouples the gateway from the audit pipeline. The gateway publishes to SQS and immediately continues processing — it does not wait for the event to be written to S3 or DynamoDB.

**Configuration:** Message retention period is 14 days. If Lambda processing falls behind, messages accumulate in the queue rather than being lost. A dead-letter queue captures messages that fail processing after 3 retries.

**If it goes down:** The gateway logs SQS publish failures and continues processing requests. Audit events for the outage period will be missing from the logs. SQS is a managed, highly available service — true outages are extremely rare.

---

### Lambda Audit Worker

**What it is:** A Python Lambda function that is triggered by SQS events. It processes each audit event and writes it to both S3 and DynamoDB.

**Why Lambda:** Audit processing does not need to run continuously. Lambda scales from zero automatically and costs nothing when idle. It handles burst traffic (when many audit events arrive simultaneously) by scaling up Lambda concurrency.

**If it goes down:** Events accumulate in SQS (up to 14 days). When Lambda recovers, it processes the backlog automatically. No audit events are lost.

---

### S3 Audit Bucket (Object Lock, 7-year retention)

**What it is:** An S3 bucket with Object Lock enabled in COMPLIANCE mode with a 7-year (2555-day) retention period.

**Why Object Lock COMPLIANCE mode:** In COMPLIANCE mode, no one — including the AWS account root user — can delete or modify locked objects before the retention period expires. This is a technical guarantee that audit logs cannot be tampered with, which is required for SOC 2, HIPAA, and financial compliance frameworks.

**If it goes down:** S3 does not go down. It has 99.999999999% (11 nines) durability. If a write fails, the Lambda retries via SQS.

---

### DynamoDB Audit Table

**What it is:** A DynamoDB table that stores audit events in a queryable format. The dashboard reads from this table.

**Why it exists alongside S3:** S3 audit logs are the immutable record of truth, but they cannot be efficiently queried by tenant ID, time range, or user ID. DynamoDB is the queryable index over the same data.

**If it goes down:** The dashboard cannot display audit logs during the outage. The gateway and S3 audit trail are unaffected. DynamoDB is a managed service with multi-AZ replication.

---

### ECR (Elastic Container Registry)

**What it is:** Two private Docker image repositories — one for the gateway, one for the dashboard.

**Lifecycle policies:** Images are tagged with `latest` and with the 7-character git SHA. Lifecycle policies retain the 10 most recent images and delete older ones to control storage costs.

**If it goes down:** Existing ECS tasks continue running (they have the image in memory). New deployments would fail until ECR recovers. ECR is a managed, highly available service.

---

## VPC Architecture

### Subnet Design

The VPC uses a public/private subnet split across two availability zones (`us-east-1a` and `us-east-1b`):

| Subnet Type | Contents | Internet Access |
|---|---|---|
| Public | ALB, NAT Gateways | Direct via Internet Gateway |
| Private | ECS tasks, Lambda, VPC endpoints | Via NAT Gateway (outbound only) |

### Why ECS Runs in Private Subnets

ECS tasks have no public IP addresses and cannot be reached directly from the internet. All inbound traffic must pass through the ALB, which applies WAF rules and TLS termination. This reduces the attack surface: even if an attacker discovered the IP address of an ECS task, they could not connect to it directly.

### VPC Endpoints

VPC endpoints allow ECS tasks in private subnets to communicate with AWS services without the traffic leaving the AWS network and without requiring NAT Gateway.

| Endpoint | Type | Why it exists |
|---|---|---|
| S3 | Gateway (free) | Lambda writes audit logs to S3 |
| DynamoDB | Gateway (free) | Lambda writes audit index to DynamoDB; Dashboard reads from DynamoDB |
| SQS | Interface | Gateway publishes audit events to SQS |
| Secrets Manager | Interface | ECS tasks fetch API keys and LLM credentials at startup |
| SSM Parameter Store | Interface | ECS tasks fetch non-secret configuration |
| CloudWatch Logs | Interface | ECS containers write logs to CloudWatch |
| CloudWatch Monitoring | Interface | ECS and Lambda publish metrics |
| ECR API | Interface | ECS pulls task definition and image metadata |
| ECR DKR | Interface | ECS pulls Docker image layers |
| Bedrock | Interface | Gateway calls AWS Bedrock LLMs without leaving the VPC |

### NAT Gateway Usage

The only traffic that goes through the NAT Gateway (and thus out to the internet) is traffic to OpenAI and Anthropic APIs. All AWS service traffic uses VPC endpoints. This minimizes NAT Gateway data processing charges and keeps most traffic within the AWS network.

---

## Security Design

| Control | Implementation |
|---|---|
| TLS everywhere | ACM wildcard certificate on ALB; all HTTP traffic redirected to HTTPS |
| No secrets in code | API keys and LLM credentials stored in AWS Secrets Manager; non-secret config in SSM Parameter Store; ECS tasks fetch at startup |
| Non-root containers | Both Dockerfile configurations create a dedicated non-root user and run the process as that user |
| Private subnets for all compute | ECS tasks, Lambda functions — none have public IP addresses |
| WAF on ALB | AWS WAF with managed rule groups blocks known malicious patterns before requests reach application code |
| OIDC for CI/CD | GitHub Actions uses OIDC to obtain short-lived AWS credentials (1 hour); no long-lived access keys exist for CI/CD |
| Immutable audit logs | S3 Object Lock in COMPLIANCE mode; 7-year retention; no delete API calls permitted |
| Dependency scanning | GitHub Actions runs dependency vulnerability checks on each build |
| Least-privilege IAM | Each ECS task and Lambda function has its own IAM role with only the permissions it requires |

---

## Data Flow for a Single Request

This is what happens from the moment a client sends a request to when they receive a response, with approximate timing:

1. **[0ms]** Client sends `POST https://api.sentorix.io/v1/gateway/chat` with `X-Sentorix-API-Key` header.

2. **[0–2ms]** Cloudflare DNS resolves `api.sentorix.io` to the ALB IP address (cached after first resolution).

3. **[2–5ms]** TCP + TLS handshake with the ALB. ALB terminates TLS using the ACM wildcard certificate. WAF rules are evaluated — if the request matches a WAF rule, it is blocked here with a 403 before reaching any application code.

4. **[5–6ms]** ALB forwards the decrypted HTTP request to the ECS Gateway task on port 8000 via the target group.

5. **[6–7ms]** FastAPI authenticates the `X-Sentorix-API-Key` header. Invalid key → 401 immediately.

6. **[7–15ms]** `asyncio.gather()` runs PII detection and policy evaluation concurrently:
   - **PII Detection:** Presidio analyzer scans the prompt text for configured entity types (PERSON, EMAIL_ADDRESS, PHONE_NUMBER, US_SSN, CREDIT_CARD, etc.) using the `en_core_web_lg` spaCy model. Detected entities are recorded with their position and confidence score.
   - **Policy Evaluation:** The rule engine scores the request based on matched rules. PII detections feed into this score.

7. **[15–16ms]** Policy decision is made:
   - **BLOCK:** Return 403 with `X-Sentorix-Action: block` and the blocking reason. Skip steps 8–9.
   - **WARN:** Redact detected PII in the prompt (replace with `[REDACTED]`). Continue with redacted prompt.
   - **ALLOW:** Continue with original prompt.

8. **[16ms + LLM latency]** Gateway forwards the (possibly redacted) prompt to the configured LLM provider (OpenAI, Anthropic, or Bedrock). The LLM provider processes the request — this typically takes 500ms–5000ms depending on model and prompt length. The gateway streams the response back to the client as it arrives.

9. **[16ms, concurrent with step 8]** Gateway publishes an audit event to SQS asynchronously (fire-and-forget). The event includes: tenant ID, user ID, original prompt hash, redacted prompt, policy decision, risk score, detected PII entities, timestamp, and request ID.

10. **[Shortly after step 9]** Lambda processes the SQS event and writes the audit record to:
    - **S3:** JSON object with the full audit event, stored under `s3://sentorix-audit-{account}/year/month/day/request-id.json`. Object Lock prevents modification.
    - **DynamoDB:** Row keyed by tenant ID + timestamp for dashboard queries.

11. **[Final response to client]** Response includes:
    - `X-Sentorix-Request-ID`: unique ID for tracing
    - `X-Sentorix-Action`: `allow`, `warn`, or `block`
    - `X-Sentorix-Risk-Score`: numeric risk score (e.g., `0.3`)
    - `X-Sentorix-Latency-Ms`: gateway processing time in milliseconds

**Total gateway overhead (excluding LLM latency):** < 20ms
