# Gateway API Reference

**Base URL:** `https://api.sentorix.io`
**Version:** v1

---

## Overview

The Sentorix Gateway is a drop-in replacement for direct OpenAI API calls that adds AI governance to every request. It sits between your application and the LLM provider, detecting and redacting PII, enforcing compliance policies, and creating an immutable audit trail — all with less than 20ms of added latency.

**The core concept is simple:** change one URL in your existing OpenAI integration, add two headers, and every AI call in your application is automatically governed.

```python
# Before Sentorix
client = openai.OpenAI(api_key="sk-...")

# After Sentorix — one line change
client = openai.OpenAI(
    api_key="sk-...",          # Your existing OpenAI key
    base_url="https://api.sentorix.io/v1/gateway"
)
```

Your application code does not change. The request format, response format, and streaming behavior are all identical to the OpenAI API.

---

## Authentication

Every request to the gateway must include a Sentorix API key and tenant identification headers.

### Required Headers

| Header | Description | Example |
|---|---|---|
| `X-Sentorix-API-Key` | Your Sentorix API key. Contact support to get one. | `snx_live_abc123xyz` |
| `X-Tenant-ID` | Your organization's tenant identifier in Sentorix. | `acme-corp` |
| `X-User-ID` | The identifier of the end user making the request. Used for per-user audit trails and policy enforcement. | `user_12345` |

### Getting an API Key

API keys are issued by the Sentorix team. Contact support at support@sentorix.io to request one for your organization. Include your company name and the email address of the technical contact.

### Invalid or missing key

Requests with a missing or invalid `X-Sentorix-API-Key` receive:

```json
HTTP/1.1 401 Unauthorized

{
  "error": "INVALID_API_KEY",
  "message": "Invalid or missing API key"
}
```

---

## Base URL

```
https://api.sentorix.io
```

All endpoints are versioned under `/v1/`.

---

## Endpoints

### POST /v1/gateway/chat

The primary endpoint. Accepts an OpenAI-compatible chat completion request, runs it through Sentorix governance, and forwards it to your configured LLM provider.

#### Request

```
POST https://api.sentorix.io/v1/gateway/chat
Content-Type: application/json
X-Sentorix-API-Key: snx_live_abc123xyz
X-Tenant-ID: acme-corp
X-User-ID: user_12345
```

**Request body:**

```json
{
  "model": "gpt-4o",
  "messages": [
    {
      "role": "system",
      "content": "You are a helpful customer service assistant."
    },
    {
      "role": "user",
      "content": "How do I reset my password?"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 500,
  "stream": false
}
```

The request body is the standard OpenAI `ChatCompletion` request format. All standard OpenAI fields are supported and passed through to the provider unchanged (after PII redaction if applicable).

---

#### Response: Successful allowed request

The request contained no policy violations. The response is the OpenAI completion response with additional Sentorix headers.

```
HTTP/1.1 200 OK
X-Sentorix-Request-ID: req_01HZ4K9BVM3VBMAQEP5X8YVJT4
X-Sentorix-Action: allow
X-Sentorix-Risk-Score: 0.05
X-Sentorix-Latency-Ms: 12
Content-Type: application/json
```

```json
{
  "id": "chatcmpl-abc123",
  "object": "chat.completion",
  "created": 1743000000,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "To reset your password, go to the login page and click 'Forgot Password'..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 35,
    "completion_tokens": 80,
    "total_tokens": 115
  }
}
```

---

#### Response: Warned request (PII detected, redacted)

The request contained PII that was detected and redacted before forwarding to the LLM. The request was allowed, but the prompt sent to the LLM had sensitive data replaced.

```
HTTP/1.1 200 OK
X-Sentorix-Request-ID: req_01HZ4K9BVM3VBMAQEP5X8YVJT5
X-Sentorix-Action: warn
X-Sentorix-Risk-Score: 0.45
X-Sentorix-Latency-Ms: 15
X-Sentorix-Redacted-Entities: EMAIL_ADDRESS,PERSON
Content-Type: application/json
```

```json
{
  "id": "chatcmpl-def456",
  "object": "chat.completion",
  "created": 1743000001,
  "model": "gpt-4o",
  "choices": [
    {
      "index": 0,
      "message": {
        "role": "assistant",
        "content": "I can help with that. The account associated with the provided email has been located..."
      },
      "finish_reason": "stop"
    }
  ],
  "usage": {
    "prompt_tokens": 28,
    "completion_tokens": 45,
    "total_tokens": 73
  },
  "_sentorix": {
    "action": "warn",
    "redacted_entities": ["EMAIL_ADDRESS", "PERSON"],
    "original_prompt_redacted": true
  }
}
```

The LLM responded to the redacted prompt (e.g., `"contact [REDACTED] at [REDACTED] email"` instead of the original). Your application receives the LLM's response to the safe version of the prompt.

---

#### Response: Blocked request (policy violation)

The request exceeded the risk threshold. It was not forwarded to the LLM provider.

```
HTTP/1.1 403 Forbidden
X-Sentorix-Request-ID: req_01HZ4K9BVM3VBMAQEP5X8YVJT6
X-Sentorix-Action: block
X-Sentorix-Risk-Score: 0.95
X-Sentorix-Latency-Ms: 8
Content-Type: application/json
```

```json
{
  "error": "POLICY_VIOLATION",
  "message": "Request blocked: high-risk content detected",
  "action": "block",
  "risk_score": 0.95,
  "detected_entities": ["US_SSN"],
  "request_id": "req_01HZ4K9BVM3VBMAQEP5X8YVJT6"
}
```

Your application should handle 403 responses from this endpoint and inform the end user that their request could not be processed due to sensitive content.

---

#### Response headers explained

| Header | Description | Example |
|---|---|---|
| `X-Sentorix-Request-ID` | Unique identifier for this request. Use this when contacting support or correlating audit logs. | `req_01HZ4K9BVM3VBMAQEP5X8YVJT4` |
| `X-Sentorix-Action` | The policy decision made for this request: `allow`, `warn`, or `block`. | `warn` |
| `X-Sentorix-Risk-Score` | The calculated risk score from 0.0 (no risk) to 1.0 (maximum risk). | `0.45` |
| `X-Sentorix-Latency-Ms` | Gateway processing time in milliseconds, excluding LLM provider latency. | `12` |
| `X-Sentorix-Redacted-Entities` | Comma-separated list of entity types that were redacted. Only present when `X-Sentorix-Action: warn`. | `EMAIL_ADDRESS,PERSON` |

---

#### HTTP status codes

| Status Code | Meaning |
|---|---|
| `200 OK` | Request processed. Check `X-Sentorix-Action` header for `allow` or `warn`. |
| `400 Bad Request` | Malformed request body. Response includes details. |
| `401 Unauthorized` | Missing or invalid `X-Sentorix-API-Key`. |
| `403 Forbidden` | Request blocked by policy. Check response body for `detected_entities`. |
| `413 Content Too Large` | Payload exceeds inline limit (512KB). Use presigned upload for large payloads. |
| `429 Too Many Requests` | Rate limit exceeded. See Rate Limits section. |
| `502 Bad Gateway` | LLM provider returned an error. Check `error` field in response body. |
| `504 Gateway Timeout` | LLM provider did not respond within 30 seconds. |

---

### POST /v1/gateway/presigned-upload

For requests where the payload exceeds 512KB (for example, prompts that include large document excerpts), upload the payload to S3 first using a presigned URL, then reference it in your chat request.

#### Step 1: Request a presigned URL

```
POST https://api.sentorix.io/v1/gateway/presigned-upload
Content-Type: application/json
X-Sentorix-API-Key: snx_live_abc123xyz
X-Tenant-ID: acme-corp
X-User-ID: user_12345
```

```json
{
  "content_type": "application/json",
  "content_length": 750000
}
```

**Response:**
```json
{
  "upload_url": "https://sentorix-uploads-....s3.amazonaws.com/uploads/tenant/...",
  "upload_id": "upload_01HZ4K9BVM3VBMAQEP5X8YVJT7",
  "expires_at": "2026-03-26T19:30:00Z"
}
```

#### Step 2: Upload the payload

Use the `upload_url` to PUT your payload directly to S3:

```bash
curl -X PUT \
  -H "Content-Type: application/json" \
  --data-binary @large-payload.json \
  "https://sentorix-uploads-....s3.amazonaws.com/uploads/tenant/..."
```

#### Step 3: Submit the chat request with the upload ID

```json
{
  "model": "gpt-4o",
  "messages": [...],
  "upload_id": "upload_01HZ4K9BVM3VBMAQEP5X8YVJT7"
}
```

The gateway retrieves the payload from S3, processes it through PII detection and policy evaluation, and forwards it to the LLM.

---

### GET /v1/health

Health check endpoint. Returns the service status. No authentication required.

**Request:**
```
GET https://api.sentorix.io/v1/health
```

**Response:**
```json
{
  "status": "ok",
  "version": "0.2.1"
}
```

---

## Quick Start

### Step 1: Get your API key

Contact support@sentorix.io with your company name and the email address of the engineer who will integrate the gateway. You will receive:
- Your `X-Sentorix-API-Key`
- Your `X-Tenant-ID`

### Step 2: Make your first request

Replace `YOUR_API_KEY`, `YOUR_TENANT_ID`, and `YOUR_OPENAI_KEY` with real values:

```bash
curl -X POST https://api.sentorix.io/v1/gateway/chat \
  -H "Content-Type: application/json" \
  -H "X-Sentorix-API-Key: YOUR_API_KEY" \
  -H "X-Tenant-ID: YOUR_TENANT_ID" \
  -H "X-User-ID: test-user-1" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "What is 2 + 2?"}
    ]
  }'
```

You should receive a 200 response with `X-Sentorix-Action: allow` and the answer from the LLM.

### Step 3: Test PII detection

```bash
curl -X POST https://api.sentorix.io/v1/gateway/chat \
  -H "Content-Type: application/json" \
  -H "X-Sentorix-API-Key: YOUR_API_KEY" \
  -H "X-Tenant-ID: YOUR_TENANT_ID" \
  -H "X-User-ID: test-user-1" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {"role": "user", "content": "My SSN is 123-45-6789, can you help me?"}
    ]
  }'
```

You should receive a 403 response with `"detected_entities": ["US_SSN"]`.

### Step 4: Integrate using the OpenAI SDK

#### Python

```python
import openai

client = openai.OpenAI(
    api_key="your-openai-key",            # Your existing OpenAI API key
    base_url="https://api.sentorix.io/v1/gateway",
    default_headers={
        "X-Sentorix-API-Key": "snx_live_abc123xyz",
        "X-Tenant-ID": "acme-corp",
        "X-User-ID": "user_12345",        # Set this dynamically per request
    }
)

response = client.chat.completions.create(
    model="gpt-4o",
    messages=[
        {"role": "user", "content": "Help me write a product description."}
    ]
)

print(response.choices[0].message.content)

# Check the Sentorix action (optional)
# Note: the OpenAI SDK does not expose custom headers directly.
# For header access, use httpx directly (see below).
```

#### Python with header access

If you need to read the `X-Sentorix-Action` or `X-Sentorix-Risk-Score` headers:

```python
import httpx
import json

headers = {
    "Content-Type": "application/json",
    "X-Sentorix-API-Key": "snx_live_abc123xyz",
    "X-Tenant-ID": "acme-corp",
    "X-User-ID": "user_12345",
    "Authorization": f"Bearer your-openai-key",
}

payload = {
    "model": "gpt-4o",
    "messages": [
        {"role": "user", "content": "Help me write a product description."}
    ]
}

with httpx.Client() as client:
    response = client.post(
        "https://api.sentorix.io/v1/gateway/chat",
        headers=headers,
        json=payload,
        timeout=60.0
    )

action = response.headers.get("X-Sentorix-Action")
risk_score = response.headers.get("X-Sentorix-Risk-Score")
request_id = response.headers.get("X-Sentorix-Request-ID")

print(f"Action: {action}, Risk: {risk_score}, Request ID: {request_id}")

if response.status_code == 403:
    error = response.json()
    print(f"Blocked: {error['detected_entities']}")
else:
    result = response.json()
    print(result["choices"][0]["message"]["content"])
```

#### Node.js

```javascript
import OpenAI from 'openai';

const client = new OpenAI({
  apiKey: 'your-openai-key',
  baseURL: 'https://api.sentorix.io/v1/gateway',
  defaultHeaders: {
    'X-Sentorix-API-Key': 'snx_live_abc123xyz',
    'X-Tenant-ID': 'acme-corp',
    'X-User-ID': 'user_12345',  // Set dynamically per request in production
  },
});

async function main() {
  const response = await client.chat.completions.create({
    model: 'gpt-4o',
    messages: [
      { role: 'user', content: 'Help me write a product description.' }
    ],
  });

  console.log(response.choices[0].message.content);
}

main();
```

#### Node.js with header access (fetch)

```javascript
const response = await fetch('https://api.sentorix.io/v1/gateway/chat', {
  method: 'POST',
  headers: {
    'Content-Type': 'application/json',
    'X-Sentorix-API-Key': 'snx_live_abc123xyz',
    'X-Tenant-ID': 'acme-corp',
    'X-User-ID': userId,
    'Authorization': `Bearer ${openAiKey}`,
  },
  body: JSON.stringify({
    model: 'gpt-4o',
    messages: [{ role: 'user', content: userMessage }],
  }),
});

const action = response.headers.get('X-Sentorix-Action');
const riskScore = response.headers.get('X-Sentorix-Risk-Score');

if (response.status === 403) {
  const error = await response.json();
  console.error('Blocked:', error.detected_entities);
} else {
  const data = await response.json();
  console.log(data.choices[0].message.content);
}
```

---

## What Gets Detected

### PII Entity Types

| Entity Type | Description | Example |
|---|---|---|
| `PERSON` | Full names of individuals | "John Smith", "Mary Johnson" |
| `EMAIL_ADDRESS` | Email addresses | "user@example.com" |
| `PHONE_NUMBER` | Phone numbers in various formats | "555-867-5309", "+1 (555) 867-5309" |
| `US_SSN` | US Social Security Numbers | "123-45-6789", "123456789" |
| `CREDIT_CARD` | Credit/debit card numbers | "4532015112830366" |
| `IBAN_CODE` | International bank account numbers | "GB82WEST12345698765432" |
| `IP_ADDRESS` | IPv4 and IPv6 addresses | "192.168.1.1", "::1" |
| `DATE_TIME` | Dates and times (when combined with other PII) | "January 15, 1990" |
| `NRP` | Nationalities, religious groups, political groups | Context-dependent |
| `LOCATION` | Physical addresses and place names | "123 Main St, Anytown" |
| `MEDICAL_LICENSE` | Medical license numbers | Format varies by state |
| `US_PASSPORT` | US passport numbers | "A12345678" |
| `US_DRIVER_LICENSE` | US driver's license numbers | State-dependent format |

The set of enabled entity types is configured per deployment via the `PII_ENTITIES` environment variable. Contact your Sentorix administrator or support to adjust which entity types are scanned for your tenant.

### Prompt Injection Patterns

The gateway scans prompts for common prompt injection patterns, including:

- Instructions to "ignore previous instructions" or "disregard the system prompt"
- Attempts to extract the system prompt or internal instructions
- Jailbreak phrases and role-playing attacks designed to bypass safety guidelines
- Encoded or obfuscated versions of the above

When a prompt injection pattern is detected, the risk score increases. Depending on your policy configuration, the request may be warned or blocked.

---

## Policy Decisions

### Understanding block/warn/allow

| Decision | HTTP Status | What happened | What to do |
|---|---|---|---|
| `allow` | 200 | No issues found. Request forwarded unchanged. | Use the response normally. |
| `warn` | 200 | PII detected and redacted. Request forwarded with redacted prompt. | Use the response. Optionally log `X-Sentorix-Redacted-Entities` for your records. |
| `block` | 403 | Risk score exceeded threshold. Request not forwarded to LLM. | Handle the 403, inform the user that their message could not be processed. Do not retry the same message. |

### Interpreting the response for `warn`

When `X-Sentorix-Action: warn`, the LLM received a version of the prompt with sensitive data replaced by `[REDACTED]` placeholders. The LLM's response is based on the redacted version. For most use cases, this is appropriate — the LLM can still answer the user's question without having access to the sensitive information.

If your use case requires the LLM to process the actual PII (for example, a healthcare application that legitimately needs to work with SSNs), contact Sentorix support to discuss policy configuration options.

---

## Error Reference

| Error Code | HTTP Status | Description | Action |
|---|---|---|---|
| `INVALID_API_KEY` | 401 | The `X-Sentorix-API-Key` header is missing or not recognized. | Check that the header is present and the key value is correct. |
| `POLICY_VIOLATION` | 403 | The request was blocked by the policy engine. | Inform the user. Do not retry. Check `detected_entities` for details. |
| `PROVIDER_ERROR` | 502 | The LLM provider returned an error. | Check the `provider_error` field in the response for details. Retry with exponential backoff. |
| `PAYLOAD_TOO_LARGE` | 413 | Payload exceeds the absolute maximum (10MB). | Reduce payload size. This request cannot be processed even via presigned upload. |
| `PAYLOAD_OFFLOAD_REQUIRED` | 413 | Payload exceeds the inline limit (512KB) but is under the maximum. | Use the presigned upload endpoint. |
| `RATE_LIMIT_EXCEEDED` | 429 | Too many requests per minute for your API key. | Back off and retry after the time indicated in the `Retry-After` header. |
| `INTERNAL_ERROR` | 500 | Unexpected error in the gateway. | Retry once. If persistent, contact support with the `X-Sentorix-Request-ID`. |

---

## Rate Limits

Current limits per API key:

| Window | Limit |
|---|---|
| Per minute | 60 requests |
| Per hour | 1,000 requests |
| Per day | 10,000 requests |

When a rate limit is exceeded, the gateway returns:

```
HTTP/1.1 429 Too Many Requests
Retry-After: 45
Content-Type: application/json

{
  "error": "RATE_LIMIT_EXCEEDED",
  "message": "Rate limit exceeded. Retry after 45 seconds.",
  "retry_after_seconds": 45
}
```

Respect the `Retry-After` header. If you consistently hit rate limits, contact support to discuss increased limits for your tenant.

> **Note:** Rate limits are currently enforced in-memory per gateway instance. In multi-instance deployments, the effective rate limit may be higher than listed. This is a known limitation tracked in [known-issues.md](../runbooks/known-issues.md).
