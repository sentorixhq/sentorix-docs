# Known Issues

This document tracks all known bugs, limitations, and workarounds in the Sentorix system.

---

## Issue 1: SSN PII Detection Not Triggering in Production

**Severity:** High
**Status:** Open
**GitHub Issue:** [sentorixhq/sentorix-gateway#XX](https://github.com/sentorixhq/sentorix-gateway/issues)

### Symptom

Prompts containing US Social Security Numbers (formatted as `XXX-XX-XXXX` or `XXXXXXXXX`) are not being detected by the PII engine in the production environment. Requests containing SSNs pass through with `action: allow` instead of being blocked or warned.

The same prompts sent to a local development instance where `en_core_web_lg` is installed correctly are detected and blocked.

### Root Cause

The production Docker image was built with the wrong spaCy language model. The `en_core_web_sm` (small) model was downloaded instead of `en_core_web_lg` (large).

The small model (`en_core_web_sm`) has limited Named Entity Recognition capability and does not reliably recognize `US_SSN` entities. Microsoft Presidio's SSN recognizer relies on spaCy's NER pipeline to identify context around potential SSN patterns. Without the large model's NER, the contextual recognition fails, and only exact regex pattern matches may fire — which is insufficient for real-world SSN formats in natural language.

The `en_core_web_lg` model is ~750MB and contains a more complete NER model trained on a larger corpus. It is required for accurate detection of SSN, person names in context, and several other entity types.

**How to confirm the model in the current image:**

Using ECS Exec (see [debugging-ecs.md](debugging-ecs.md)):

```bash
python3 -c "
import spacy
import en_core_web_lg
print('Model:', en_core_web_lg.meta['name'])
print('Vectors:', en_core_web_lg.meta.get('vectors', {}).get('width', 'none'))
"
```

If this raises `ModuleNotFoundError: No module named 'en_core_web_lg'`, the wrong model is installed.

### Workaround

Currently no workaround is available without redeploying with the correct model. The issue affects all SSN detections in production.

Email addresses, phone numbers, credit card numbers, and person names are largely unaffected as they rely more on regex patterns than NER context.

### Fix

Update the gateway Dockerfile to explicitly download `en_core_web_lg`:

```dockerfile
# In Dockerfile — ensure this line uses the large model
RUN python -m spacy download en_core_web_lg
```

Verify the CI pipeline's test step does not accidentally override this. The CI uses `en_core_web_sm` for speed in unit tests — this is acceptable because unit tests mock Presidio output and do not depend on the real model. The production Docker image must use `en_core_web_lg`.

After the fix, redeploy and validate:

```bash
curl -X POST https://api.sentorix.io/v1/gateway/chat \
  -H "X-Sentorix-API-Key: YOUR_KEY" \
  -H "X-Tenant-ID: test" \
  -H "X-User-ID: test" \
  -d '{"model":"gpt-4o-mini","messages":[{"role":"user","content":"My SSN is 123-45-6789"}]}'
# Expected: HTTP 403 with detected_entities: ["US_SSN"]
```

---

## Issue 2: moto Incompatibility with aiobotocore in Tests

**Severity:** Low
**Status:** Resolved
**GitHub Issue:** N/A (resolved without a formal issue)

### Symptom

When writing tests for the audit publisher using the `moto` library (which mocks AWS services), tests either fail with `botocore.exceptions.NoRegionError` or the mock does not activate — causing the test to attempt a real AWS API call.

### Root Cause

`moto` works by monkey-patching `botocore`, the synchronous AWS SDK core library. The `aioboto3` library used by the audit publisher is built on `aiobotocore`, which has its own implementation of the HTTP request layer that bypasses the synchronous `botocore` path that `moto` patches.

When `@mock_aws` is applied and `aioboto3` is used inside the test, the async client makes a real network call (or fails attempting to) rather than hitting the moto in-memory mock.

### Resolution

The `test_audit_publisher.py` file uses `unittest.mock.AsyncMock` to mock the SQS client directly, rather than relying on moto:

```python
from unittest.mock import AsyncMock, patch

@pytest.mark.asyncio
async def test_publishes_audit_event():
    mock_sqs = AsyncMock()
    mock_sqs.send_message.return_value = {"MessageId": "test-id-123"}

    with patch("app.audit_publisher.get_sqs_client", return_value=mock_sqs):
        publisher = AuditPublisher(queue_url="https://sqs.us-east-1.amazonaws.com/123/test")
        await publisher.publish(sample_audit_event)

    mock_sqs.send_message.assert_called_once()
```

This approach correctly tests that:
- `send_message` is called
- The correct `QueueUrl` is passed
- The `MessageBody` is valid JSON containing the expected fields

It does not test SQS itself — that is AWS's responsibility.

If future tests need to verify S3 or DynamoDB write behavior in async code, the same `AsyncMock` pattern applies. Use `moto` only for synchronous boto3 code.

---

## Issue 3: Rate Limiter Is In-Memory Only

**Severity:** Medium
**Status:** Open
**GitHub Issue:** [sentorixhq/sentorix-gateway#YY](https://github.com/sentorixhq/sentorix-gateway/issues)

### Symptom

The rate limiter enforces per-API-key request limits correctly in single-instance deployments (current MVP). However, if the ECS service is scaled to 2 or more tasks, each task maintains its own independent rate limit counter. A client can make double the allowed requests per minute by having their requests distributed across two tasks.

For example, with a limit of 60 requests/minute and 2 ECS tasks running:
- Task A receives 60 requests → rate limited
- Task B receives 60 requests → rate limited
- But the client effectively sent 120 requests/minute

### Root Cause

The rate limiter is implemented using a Python in-memory dictionary (or `slowapi` / similar in-process limiter) that is not shared between processes. Each ECS task instance has its own counter state, so tasks do not coordinate.

### Impact

During the current MVP with `desiredCount=1` (one ECS task), this issue does not manifest. The limiter works correctly. The issue becomes relevant when scaling to multiple instances for high availability or increased throughput.

### Workaround

Keep `desiredCount=1` for the gateway ECS service until the rate limiter is moved to a shared backend. This limits availability during ECS task replacement events (rolling deploy, task crash) to the ~30-second window before the new task passes health checks.

### Fix

Replace the in-memory rate limiter with a Redis-backed limiter (using `aioredis` and a sliding window counter). Options:

1. **AWS ElastiCache for Redis** — managed Redis cluster accessible from the ECS private subnets
2. **Redis via a small ECS task** — a single-task Redis container in the private subnet

The rate limiter implementation should be updated to use `redis.asyncio` with an atomic Lua script or `INCR` + `EXPIRE` for the sliding window:

```python
async def check_rate_limit(api_key: str, window_seconds: int, max_requests: int) -> bool:
    key = f"ratelimit:{api_key}:{int(time.time() // window_seconds)}"
    count = await redis.incr(key)
    if count == 1:
        await redis.expire(key, window_seconds * 2)
    return count <= max_requests
```

This requires adding ElastiCache or a Redis task to Terraform, adding a `REDIS_URL` environment variable, and updating the rate limiter code in `app/middleware/rate_limiter.py`.

---

## Issue 4: Dashboard Lacks Automated Tests

**Severity:** Low
**Status:** Open

### Symptom

The `sentorix-dashboard` repository has no automated test suite. The CI/CD pipeline deploys directly without a test step.

### Impact

Regressions in the dashboard UI or API data-fetching layer are caught only by manual observation or user reports. A broken build can deploy to production if the Docker build succeeds.

### Fix

Add a test step to `deploy-dashboard.yml` before `build-and-push`:

1. **Unit tests** with Vitest or Jest for utility functions and data transformations
2. **Component tests** with React Testing Library for key UI components
3. **Integration tests** (optional) with Playwright for end-to-end flows

Minimum viable: add `npm run test` (even if it just runs `echo "no tests yet"`) so the job structure is in place and tests can be added incrementally.

---

## Issue 5: Audit Events Missing During Gateway Startup

**Severity:** Low
**Status:** Open

### Symptom

During the ~30–60 second window after a new ECS task starts (before it passes ALB health checks), the old task is still running. During a rolling deployment, there is a brief window where two tasks may be running simultaneously. In rare cases, audit events from the old task that are in-flight during shutdown may not complete their SQS publish.

### Root Cause

ECS sends a `SIGTERM` to the container when draining begins. If `uvicorn` shuts down before all in-flight SQS publish coroutines complete, those events are dropped.

### Impact

Extremely rare — only affects requests that are actively being processed at the exact moment the container receives `SIGTERM`. For compliance-critical deployments, this is worth addressing.

### Fix

Implement graceful shutdown handling in FastAPI/uvicorn:

1. Catch `SIGTERM` in the FastAPI lifespan handler
2. Stop accepting new requests
3. Wait for in-flight SQS publishes to complete before exiting (with a timeout)
4. Configure the ECS `stopTimeout` in the task definition to give the container enough time to drain (default is 30 seconds)

The `stopTimeout` should be set to 60 seconds in the task definition:

```json
"stopTimeout": 60
```
