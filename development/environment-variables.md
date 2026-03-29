> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# Environment Variables

This document covers every environment variable for both the gateway and the dashboard. For each variable: what it does, where it lives in production, and example values.

---

## Gateway Environment Variables

### Quick Reference Table

| Variable | Required | Default | Production Storage |
|---|---|---|---|
| `APP_NAME` | No | `sentorix-gateway` | ECS task definition |
| `APP_VERSION` | No | `0.1.0` | ECS task definition |
| `ENVIRONMENT` | No | `production` | ECS task definition |
| `DEBUG` | No | `false` | ECS task definition |
| `LOG_LEVEL` | No | `INFO` | ECS task definition |
| `API_KEY_HEADER` | No | `X-Sentorix-API-Key` | ECS task definition |
| `ALLOWED_API_KEYS` | **Yes** | — | Secrets Manager |
| `MAX_INLINE_PAYLOAD_BYTES` | No | `524288` | ECS task definition |
| `MAX_PAYLOAD_BYTES` | No | `10485760` | ECS task definition |
| `CHUNK_SIZE_BYTES` | No | `131072` | ECS task definition |
| `CHUNK_OVERLAP_BYTES` | No | `512` | ECS task definition |
| `REQUEST_TIMEOUT_SECONDS` | No | `30` | ECS task definition |
| `PII_DETECTION_ENABLED` | No | `true` | ECS task definition |
| `PII_SCORE_THRESHOLD` | No | `0.7` | ECS task definition |
| `PII_ENTITIES` | No | (see below) | ECS task definition |
| `POLICY_BLOCK_ON_HIGH_RISK` | No | `true` | ECS task definition |
| `RISK_SCORE_THRESHOLD_WARN` | No | `0.3` | ECS task definition |
| `RISK_SCORE_THRESHOLD_BLOCK` | No | `0.7` | ECS task definition |
| `AWS_REGION` | No | `us-east-1` | ECS task definition |
| `AWS_SQS_AUDIT_QUEUE_URL` | No | — | SSM Parameter Store |
| `AWS_S3_AUDIT_BUCKET` | No | — | SSM Parameter Store |
| `AWS_DYNAMODB_AUDIT_TABLE` | No | — | SSM Parameter Store |
| `OPENAI_API_KEY` | Conditional | — | Secrets Manager |
| `ANTHROPIC_API_KEY` | Conditional | — | Secrets Manager |
| `DEFAULT_LLM_PROVIDER` | No | `openai` | ECS task definition |

---

### Detailed Descriptions

#### `APP_NAME`
- **Required:** No
- **Default:** `sentorix-gateway`
- **Example:** `sentorix-gateway`
- **Description:** Application name included in log output and the health check response. Useful for identifying which service produced a log entry in aggregated logging.
- **Production storage:** ECS task definition environment variable.

---

#### `APP_VERSION`
- **Required:** No
- **Default:** `0.1.0`
- **Example:** `0.2.1`
- **Description:** Application version string returned by `GET /v1/health`. Set by the CI/CD pipeline to the git tag or commit SHA for production deployments.
- **Production storage:** ECS task definition environment variable.

---

#### `ENVIRONMENT`
- **Required:** No
- **Default:** `production`
- **Example:** `development`, `staging`, `production`
- **Description:** Environment name. Used in log output and audit event metadata so you can distinguish events from different environments in the same DynamoDB table.
- **Production storage:** ECS task definition environment variable.

---

#### `DEBUG`
- **Required:** No
- **Default:** `false`
- **Example:** `true`
- **Description:** Enables debug mode. When `true`, FastAPI returns full stack traces in error responses and enables additional debug logging. **Never set to `true` in production** — stack traces expose internal implementation details.
- **Production storage:** ECS task definition environment variable (value: `false`).

---

#### `LOG_LEVEL`
- **Required:** No
- **Default:** `INFO`
- **Example:** `DEBUG`, `INFO`, `WARNING`, `ERROR`
- **Description:** Controls the minimum log level for output. In production, `INFO` is appropriate. Set to `DEBUG` temporarily when diagnosing issues. Setting to `DEBUG` in production generates very high log volume.
- **Production storage:** ECS task definition environment variable.

---

#### `API_KEY_HEADER`
- **Required:** No
- **Default:** `X-Sentorix-API-Key`
- **Example:** `X-Sentorix-API-Key`
- **Description:** The HTTP header name the gateway looks for when authenticating requests. Changing this would break all existing client integrations. Do not change this value.
- **Production storage:** ECS task definition environment variable.

---

#### `ALLOWED_API_KEYS`
- **Required:** Yes
- **Default:** None
- **Example:** `["key-abc123","key-xyz789"]`
- **Description:** JSON array of valid API key strings. Any request presenting an `X-Sentorix-API-Key` value that is not in this list receives a 401 response. **Must be valid JSON** — the value must be a properly formatted JSON array with double-quoted strings. Common mistake: `my-key` instead of `["my-key"]`.
- **Production storage:** AWS Secrets Manager. The ECS task definition references the secret ARN, and ECS injects it as an environment variable at task startup.
- **Security note:** This is a sensitive secret. Never commit it to source code. Never log it. Rotate keys by adding the new key to the array, updating the secret, redeploying, then removing the old key after clients have migrated.

---

#### `MAX_INLINE_PAYLOAD_BYTES`
- **Required:** No
- **Default:** `524288` (512KB)
- **Example:** `524288`
- **Description:** The maximum size of a request payload that the gateway will process inline (in memory). Requests larger than this value receive a 413 response with error code `PAYLOAD_OFFLOAD_REQUIRED`, instructing the client to use the presigned upload endpoint instead. This limit protects the gateway from memory exhaustion.
- **Production storage:** ECS task definition environment variable.

---

#### `MAX_PAYLOAD_BYTES`
- **Required:** No
- **Default:** `10485760` (10MB)
- **Example:** `10485760`
- **Description:** The absolute maximum payload size. Payloads larger than this are rejected with 413 `PAYLOAD_TOO_LARGE` regardless of whether the client uses the presigned upload path. This is a hard limit at the ALB level as well.
- **Production storage:** ECS task definition environment variable.

---

#### `CHUNK_SIZE_BYTES`
- **Required:** No
- **Default:** `131072` (128KB)
- **Example:** `131072`
- **Description:** When a payload exceeds `MAX_INLINE_PAYLOAD_BYTES` and is processed via the presigned upload path, the gateway reads it in chunks of this size to avoid loading the entire payload into memory at once.
- **Production storage:** ECS task definition environment variable.

---

#### `CHUNK_OVERLAP_BYTES`
- **Required:** No
- **Default:** `512`
- **Example:** `512`
- **Description:** The number of bytes of overlap between adjacent chunks during large payload processing. Overlap ensures that PII entities that span a chunk boundary are not missed by Presidio.
- **Production storage:** ECS task definition environment variable.

---

#### `REQUEST_TIMEOUT_SECONDS`
- **Required:** No
- **Default:** `30`
- **Example:** `60`
- **Description:** Maximum time in seconds the gateway will wait for a response from the LLM provider. If the provider does not respond within this time, the gateway returns 504 to the client. For streaming endpoints, this is the timeout for the first token — not the full stream duration.
- **Production storage:** ECS task definition environment variable.

---

#### `PII_DETECTION_ENABLED`
- **Required:** No
- **Default:** `true`
- **Example:** `true`, `false`
- **Description:** Master switch for PII detection. When `false`, Presidio is not initialized and no PII scanning occurs. Disabling PII detection is not recommended for production and should only be used for performance testing or when debugging.
- **Production storage:** ECS task definition environment variable.

---

#### `PII_SCORE_THRESHOLD`
- **Required:** No
- **Default:** `0.7`
- **Example:** `0.7`
- **Description:** The minimum confidence score (0.0–1.0) from Presidio for an entity detection to be considered valid. Detections below this threshold are ignored. Lower values = more sensitive (more false positives). Higher values = less sensitive (may miss genuine PII).
- **Production storage:** ECS task definition environment variable.

---

#### `PII_ENTITIES`
- **Required:** No
- **Default:** `["PERSON","EMAIL_ADDRESS","PHONE_NUMBER","US_SSN","CREDIT_CARD","IBAN_CODE","IP_ADDRESS"]`
- **Example:** `["PERSON","EMAIL_ADDRESS","US_SSN"]`
- **Description:** JSON array of Presidio entity type names to detect. Only entity types listed here will be scanned. The full list of supported entity types is defined by Presidio — see the [Presidio documentation](https://microsoft.github.io/presidio/supported_entities/) for all available types. **Must be valid JSON array with double-quoted strings.**
- **Production storage:** ECS task definition environment variable.

---

#### `POLICY_BLOCK_ON_HIGH_RISK`
- **Required:** No
- **Default:** `true`
- **Example:** `true`, `false`
- **Description:** When `true`, requests with a risk score at or above `RISK_SCORE_THRESHOLD_BLOCK` are blocked with HTTP 403. When `false`, high-risk requests are warned but allowed through. Setting this to `false` is not recommended for production.
- **Production storage:** ECS task definition environment variable.

---

#### `RISK_SCORE_THRESHOLD_WARN`
- **Required:** No
- **Default:** `0.3`
- **Example:** `0.3`
- **Description:** Requests with a risk score at or above this threshold (but below `RISK_SCORE_THRESHOLD_BLOCK`) receive a WARN action. The prompt may be redacted, and response headers include `X-Sentorix-Action: warn`.
- **Production storage:** ECS task definition environment variable.

---

#### `RISK_SCORE_THRESHOLD_BLOCK`
- **Required:** No
- **Default:** `0.7`
- **Example:** `0.7`
- **Description:** Requests with a risk score at or above this threshold receive a BLOCK action (HTTP 403). The threshold must be greater than `RISK_SCORE_THRESHOLD_WARN`.
- **Production storage:** ECS task definition environment variable.

---

#### `AWS_REGION`
- **Required:** No
- **Default:** `us-east-1`
- **Example:** `us-east-1`
- **Description:** The AWS region where SQS, S3, and DynamoDB resources are located. Must match the region where Terraform deployed the infrastructure.
- **Production storage:** ECS task definition environment variable.

---

#### `AWS_SQS_AUDIT_QUEUE_URL`
- **Required:** No (audit logging is skipped if not set)
- **Default:** None
- **Example:** `https://sqs.us-east-1.amazonaws.com/${AWS_ACCOUNT_ID}/sentorix-audit-queue`
- **Description:** The URL of the SQS queue where audit events are published. If this is empty, the gateway skips audit publishing entirely (no error, just no audit log). In production, this must be set for compliance.
- **Production storage:** AWS SSM Parameter Store (`/sentorix/production/sqs-audit-queue-url`).

---

#### `AWS_S3_AUDIT_BUCKET`
- **Required:** No
- **Default:** None
- **Example:** `sentorix-audit-${AWS_ACCOUNT_ID}-us-east-1`
- **Description:** The name of the S3 bucket where the Lambda audit worker writes audit event JSON files. This variable is used by the Lambda function, not directly by the gateway.
- **Production storage:** AWS SSM Parameter Store.

---

#### `AWS_DYNAMODB_AUDIT_TABLE`
- **Required:** No
- **Default:** None
- **Example:** `sentorix-audit-events`
- **Description:** The name of the DynamoDB table where the Lambda audit worker writes queryable audit records.
- **Production storage:** AWS SSM Parameter Store.

---

#### `OPENAI_API_KEY`
- **Required:** Yes if `DEFAULT_LLM_PROVIDER=openai`
- **Default:** None
- **Example:** `sk-proj-...`
- **Description:** OpenAI API key. Required when the gateway is configured to forward requests to OpenAI. Get this from your OpenAI account at platform.openai.com.
- **Production storage:** AWS Secrets Manager.
- **Security note:** This is a sensitive secret. Compromised keys incur charges and must be rotated immediately.

---

#### `ANTHROPIC_API_KEY`
- **Required:** Yes if `DEFAULT_LLM_PROVIDER=anthropic`
- **Default:** None
- **Example:** `sk-ant-...`
- **Description:** Anthropic API key. Required when the gateway is configured to forward requests to Anthropic Claude models.
- **Production storage:** AWS Secrets Manager.
- **Security note:** Sensitive secret. Same rotation procedure as `OPENAI_API_KEY`.

---

#### `DEFAULT_LLM_PROVIDER`
- **Required:** No
- **Default:** `openai`
- **Example:** `openai`, `anthropic`, `bedrock`
- **Description:** The LLM provider to use when the client request does not specify one. Valid values: `openai`, `anthropic`, `bedrock`. When set to `bedrock`, the gateway uses the AWS Bedrock API via the VPC endpoint.
- **Production storage:** ECS task definition environment variable.

---

## Dashboard Environment Variables

### Quick Reference Table

| Variable | Required | Default | Production Storage |
|---|---|---|---|
| `NEXT_PUBLIC_GATEWAY_URL` | **Yes** | — | ECS task definition |
| `NEXT_PUBLIC_USE_MOCK_DATA` | No | `false` | ECS task definition |
| `NEXT_PUBLIC_APP_URL` | No | — | ECS task definition |
| `NEXT_PUBLIC_API_KEY` | **Yes** | — | Secrets Manager |

---

### Detailed Descriptions

#### `NEXT_PUBLIC_GATEWAY_URL`
- **Required:** Yes
- **Default:** None
- **Example (local):** `http://localhost:8000`
- **Example (production):** `https://api.sentorix.io`
- **Description:** The base URL of the Sentorix gateway API. The dashboard makes API calls to this URL. In production this is `https://api.sentorix.io`. Locally, point this at your running gateway instance.
- **Production storage:** ECS task definition environment variable.
- **Note:** The `NEXT_PUBLIC_` prefix makes this variable available in the browser (client-side). It is not a secret.

---

#### `NEXT_PUBLIC_USE_MOCK_DATA`
- **Required:** No
- **Default:** `false`
- **Example:** `true`, `false`
- **Description:** When `true`, the dashboard uses hardcoded mock data instead of making API calls to the gateway. Use `true` for local UI development when you do not need real data. **Must be `false` in production.**
- **Production storage:** ECS task definition environment variable (value: `false`).

---

#### `NEXT_PUBLIC_APP_URL`
- **Required:** No
- **Default:** None
- **Example (local):** `http://localhost:3000`
- **Example (production):** `https://app.sentorix.io`
- **Description:** The public URL of the dashboard itself. Used for generating absolute URLs in the application (for example, in redirect flows or email links).
- **Production storage:** ECS task definition environment variable.

---

#### `NEXT_PUBLIC_API_KEY`
- **Required:** Yes
- **Default:** None
- **Example:** `snx-dashboard-...`
- **Description:** The API key the dashboard uses to authenticate with the gateway when fetching data to display. This is a dedicated dashboard-specific key, separate from customer API keys.
- **Production storage:** AWS Secrets Manager. Injected into the ECS task at startup.
- **Security note:** Although it carries the `NEXT_PUBLIC_` prefix (which makes Next.js expose it to browser code), this key should be treated as sensitive. In a future architecture update, dashboard-to-gateway authentication should use a backend proxy to avoid exposing any credentials to the browser.
