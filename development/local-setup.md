> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# Local Development Setup

This guide takes you from a fresh MacBook to a fully running Sentorix development environment. Follow every step in order. Both Apple Silicon (M1/M2/M3) and Intel Macs are covered.

Estimated time: 30–45 minutes (mostly waiting for downloads).

---

## 1. Prerequisites

Install each tool in order. All commands assume you have Homebrew installed. If you do not have Homebrew, install it first:

```bash
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"
```

### Python 3.12

```bash
brew install python@3.12
```

Verify:
```bash
python3.12 --version
# Python 3.12.x
```

### Node.js 20+

```bash
brew install node
```

Verify:
```bash
node --version
# v20.x.x or higher
npm --version
```

### Docker Desktop

Download from [https://www.docker.com/products/docker-desktop/](https://www.docker.com/products/docker-desktop/) and install. Choose the Apple Silicon or Intel version to match your Mac.

After installation, open Docker Desktop and wait for it to show "Docker Desktop is running" in the menu bar.

Verify:
```bash
docker --version
docker compose version
```

### AWS CLI v2

```bash
brew install awscli
```

Verify:
```bash
aws --version
# aws-cli/2.x.x
```

### Git

```bash
brew install git
```

Verify:
```bash
git --version
```

### Claude Code (optional, recommended)

Claude Code provides AI-assisted development directly in your terminal and is the preferred editor for working on Sentorix.

```bash
npm install -g @anthropic-ai/claude-code
```

Verify:
```bash
claude --version
```

---

## 2. AWS Profile Setup

Sentorix uses a named AWS profile (`sentorix`) to avoid accidentally running infrastructure commands against the wrong account.

### Configure the profile

```bash
aws configure --profile sentorix
```

You will be prompted for:

```
AWS Access Key ID [None]: AKIA...
AWS Secret Access Key [None]: <your-secret-key>
Default region name [None]: us-east-1
Default output format [None]: json
```

Get your access key from AWS IAM. If you do not have one, ask the team lead to create one for you in the `sentorix` AWS account (${AWS_ACCOUNT_ID}).

### Verify it works

```bash
aws sts get-caller-identity --profile sentorix
```

Expected output:
```json
{
    "UserId": "AIDA...",
    "Account": "${AWS_ACCOUNT_ID}",
    "Arn": "arn:aws:iam::${AWS_ACCOUNT_ID}:user/your-username"
}
```

If you see an error like `InvalidClientTokenId`, your access key is wrong. If you see a different account ID, you have the wrong credentials.

---

## 3. Clone All Repositories

```bash
mkdir -p ~/workspace/sentorix
cd ~/workspace/sentorix

git clone https://github.com/sentorixhq/sentorix-gateway
git clone https://github.com/sentorixhq/sentorix-dashboard
git clone https://github.com/sentorixhq/sentorix-infra
git clone https://github.com/sentorixhq/sentorix-docs
```

After cloning, your directory structure should look like:

```
~/workspace/sentorix/
├── sentorix-gateway/
├── sentorix-dashboard/
├── sentorix-infra/
└── sentorix-docs/
```

---

## 4. Running the Gateway Locally

All commands in this section are run from inside the `sentorix-gateway` directory.

### a) Create a virtual environment

```bash
cd ~/workspace/sentorix/sentorix-gateway
python3.12 -m venv .venv
source .venv/bin/activate
```

Your shell prompt will change to show `(.venv)` indicating the virtual environment is active. You must have the virtual environment active in every terminal session where you run gateway commands.

To deactivate later: `deactivate`

### b) Install dependencies

```bash
pip install -r requirements.txt
pip install -r requirements-dev.txt
```

This installs FastAPI, Presidio, boto3, aioboto3, pytest, and all other dependencies. Expect this to take 2–5 minutes on a fast connection.

### c) Download the spaCy language model

This is a one-time step. The model is approximately 750MB and takes 5–10 minutes to download.

```bash
python -m spacy download en_core_web_lg
```

**This step is critical.** The gateway uses `en_core_web_lg` (large model) for accurate PII detection, including SSN recognition. If you accidentally download `en_core_web_sm` (small model), SSN and many other entity types will not be detected. Do not substitute a different model.

Verify the model is installed:
```bash
python -c "import spacy; nlp = spacy.load('en_core_web_lg'); print('OK')"
```

### d) Create the `.env` file

```bash
cp .env.example .env
```

Open `.env` in your editor and fill in the required values:

```bash
# Application
APP_NAME=sentorix-gateway
APP_VERSION=0.1.0
ENVIRONMENT=development
DEBUG=true
LOG_LEVEL=INFO

# Authentication - must be a valid JSON array of strings
ALLOWED_API_KEYS=["your-test-key-here"]

# PII Detection
PII_DETECTION_ENABLED=true
PII_SCORE_THRESHOLD=0.7
PII_ENTITIES=["PERSON","EMAIL_ADDRESS","PHONE_NUMBER","US_SSN","CREDIT_CARD","IBAN_CODE","IP_ADDRESS"]

# Policy
POLICY_BLOCK_ON_HIGH_RISK=true
RISK_SCORE_THRESHOLD_WARN=0.3
RISK_SCORE_THRESHOLD_BLOCK=0.7

# AWS (only needed if testing audit logging locally)
AWS_REGION=us-east-1
AWS_SQS_AUDIT_QUEUE_URL=
AWS_S3_AUDIT_BUCKET=
AWS_DYNAMODB_AUDIT_TABLE=

# LLM Providers - set at least one
OPENAI_API_KEY=sk-your-openai-key-here
ANTHROPIC_API_KEY=
DEFAULT_LLM_PROVIDER=openai

# Payload limits
MAX_INLINE_PAYLOAD_BYTES=524288
MAX_PAYLOAD_BYTES=10485760
```

**Required for basic local testing:**
- `ALLOWED_API_KEYS`: Must be a valid JSON array. Common mistake: using `your-test-key` instead of `["your-test-key"]`. The brackets and quotes are required.
- `PII_ENTITIES`: Must be a valid JSON array. Same requirement.
- `OPENAI_API_KEY`: Required if `DEFAULT_LLM_PROVIDER=openai`. Get this from your OpenAI account.

**Optional for local development:**
- `AWS_SQS_AUDIT_QUEUE_URL`, `AWS_S3_AUDIT_BUCKET`, `AWS_DYNAMODB_AUDIT_TABLE`: Leave empty locally. The gateway will skip audit publishing if the queue URL is not set.

See [environment-variables.md](environment-variables.md) for a full description of every variable.

### e) Run unit tests first

Before starting the server, verify your setup is correct by running the unit tests:

```bash
pytest tests/unit/ -v
```

All 58 tests should pass. If any test fails, check:
- Is your virtual environment active? (`which python` should show `.venv/bin/python`)
- Did you install `requirements-dev.txt`?
- Did you install the `en_core_web_lg` spaCy model?

### f) Start the gateway

```bash
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

The `--reload` flag enables hot reloading — the server restarts automatically when you save a Python file. Do not use `--reload` in production.

Expected output:
```
INFO:     Loaded environment variables from .env
INFO:     PII detection enabled with entities: ['PERSON', 'EMAIL_ADDRESS', ...]
INFO:     Started server process [12345]
INFO:     Waiting for application startup.
INFO:     Application startup complete.
INFO:     Uvicorn running on http://0.0.0.0:8000 (Press CTRL+C to quit)
```

### g) Verify the gateway is running

```bash
curl http://localhost:8000/v1/health
```

Expected response:
```json
{"status": "ok", "version": "0.1.0"}
```

### h) Test PII detection (should be blocked)

This test sends a prompt containing a US Social Security Number. The gateway should block it.

```bash
curl -X POST http://localhost:8000/v1/gateway/chat \
  -H "Content-Type: application/json" \
  -H "X-Sentorix-API-Key: your-test-key-here" \
  -H "X-Tenant-ID: test-tenant" \
  -H "X-User-ID: test-user" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "My SSN is 123-45-6789, can you help me with my account?"
      }
    ]
  }'
```

Expected response (HTTP 403):
```json
{
  "error": "POLICY_VIOLATION",
  "message": "Request blocked due to policy violation",
  "action": "block",
  "risk_score": 0.9,
  "detected_entities": ["US_SSN"]
}
```

### i) Test a normal request (should be allowed)

```bash
curl -X POST http://localhost:8000/v1/gateway/chat \
  -H "Content-Type: application/json" \
  -H "X-Sentorix-API-Key: your-test-key-here" \
  -H "X-Tenant-ID: test-tenant" \
  -H "X-User-ID: test-user" \
  -d '{
    "model": "gpt-4o-mini",
    "messages": [
      {
        "role": "user",
        "content": "What is the capital of France?"
      }
    ]
  }'
```

Expected: HTTP 200 with a response from OpenAI containing the answer "Paris".

---

## 5. Running the Dashboard Locally

All commands in this section are run from inside the `sentorix-dashboard` directory.

### a) Install dependencies

```bash
cd ~/workspace/sentorix/sentorix-dashboard
npm install
```

### b) Create `.env.local`

```bash
cp .env.example .env.local
```

For local development with mock data (no gateway required):

```bash
NEXT_PUBLIC_GATEWAY_URL=http://localhost:8000
NEXT_PUBLIC_USE_MOCK_DATA=true
NEXT_PUBLIC_APP_URL=http://localhost:3000
NEXT_PUBLIC_API_KEY=your-test-key-here
```

### c) Run the development server

```bash
npm run dev
```

Expected output:
```
  ▲ Next.js 14.x
  - Local:        http://localhost:3000
  - Environments: .env.local

 ✓ Ready in 2.1s
```

### d) Open in browser

Navigate to [http://localhost:3000](http://localhost:3000).

### e) Mock data vs live gateway

**`NEXT_PUBLIC_USE_MOCK_DATA=true` (default for local dev)**

The dashboard uses hardcoded mock data. No gateway needs to be running. Use this when you are working purely on UI changes and do not need real data.

**`NEXT_PUBLIC_USE_MOCK_DATA=false`**

The dashboard calls the gateway at `NEXT_PUBLIC_GATEWAY_URL` for real data. The gateway must be running (see Section 4). Set `NEXT_PUBLIC_GATEWAY_URL=http://localhost:8000` to point at your local gateway.

Use this when you are testing the full integration between the dashboard and the gateway, or when you need to see real audit log data.

---

## 6. Running Both Together

This is the typical development setup when working on features that span both services.

**Terminal 1 — Gateway:**
```bash
cd ~/workspace/sentorix/sentorix-gateway
source .venv/bin/activate
uvicorn app.main:app --host 0.0.0.0 --port 8000 --reload
```

**Terminal 2 — Dashboard:**
```bash
cd ~/workspace/sentorix/sentorix-dashboard
npm run dev
```

In `.env.local`, set `NEXT_PUBLIC_USE_MOCK_DATA=false` and `NEXT_PUBLIC_GATEWAY_URL=http://localhost:8000` so the dashboard talks to your local gateway.

**Verify they are connected:**

```bash
curl http://localhost:8000/v1/health
```

Then open [http://localhost:3000](http://localhost:3000) — the dashboard should show real data if it is connected to the gateway.

---

## 7. Running with Docker Locally

The gateway repository includes a `docker-compose.yml` for running the full stack locally including LocalStack for AWS service emulation.

```bash
cd ~/workspace/sentorix/sentorix-gateway
docker compose up
```

**What docker-compose starts:**

- `gateway`: The FastAPI gateway on port 8000
- `localstack`: Emulates AWS SQS, S3, and DynamoDB locally so audit logging works without a real AWS connection

**First run warning:** The Docker build for the gateway takes 8–15 minutes because it downloads and installs the spaCy `en_core_web_lg` model inside the container. Subsequent builds use Docker layer cache and take 45–90 seconds.

**Access the gateway through docker-compose:**

```bash
curl http://localhost:8000/v1/health
```

Same as running locally directly — the port mapping is the same.

**Stopping:**

```bash
docker compose down
```

---

## 8. Common Local Issues and Fixes

### spaCy model not found

**Symptom:**
```
OSError: [E050] Can't find model 'en_core_web_lg'
```

**Cause:** The spaCy model was not downloaded, was downloaded into a different Python environment, or `en_core_web_sm` was installed instead of `en_core_web_lg`.

**Fix:**
```bash
# Make sure your virtual environment is active
source .venv/bin/activate
python -m spacy download en_core_web_lg
# Verify
python -c "import spacy; spacy.load('en_core_web_lg'); print('OK')"
```

---

### boto3 module not found

**Symptom:**
```
ModuleNotFoundError: No module named 'boto3'
```

**Cause:** Your virtual environment is not active, or `requirements.txt` was not installed.

**Fix:**
```bash
source .venv/bin/activate
pip install -r requirements.txt
```

---

### `ALLOWED_API_KEYS` parse error

**Symptom:**
```
ValidationError: ALLOWED_API_KEYS: value is not a valid list
```
or
```
pydantic_core.ValidationError: 1 validation error for Settings
  ALLOWED_API_KEYS
    Input should be a valid list [type=list_type]
```

**Cause:** The value in `.env` is not a valid JSON array. Common mistakes:

```bash
# WRONG - missing brackets
ALLOWED_API_KEYS=my-test-key

# WRONG - missing quotes around the value
ALLOWED_API_KEYS=[my-test-key]

# CORRECT
ALLOWED_API_KEYS=["my-test-key"]

# CORRECT - multiple keys
ALLOWED_API_KEYS=["key-one","key-two"]
```

**Fix:** Open `.env`, ensure the value is a valid JSON array with double quotes around each key.

---

### `PII_ENTITIES` parse error

**Symptom:** Same type of `ValidationError` as above but for `PII_ENTITIES`.

**Cause:** Same issue — not a valid JSON array.

**Fix:**
```bash
# CORRECT
PII_ENTITIES=["PERSON","EMAIL_ADDRESS","PHONE_NUMBER","US_SSN","CREDIT_CARD"]
```

---

### Port 8000 already in use

**Symptom:**
```
ERROR:    [Errno 48] Address already in use
```

**Cause:** Another process is already listening on port 8000 (often a previous gateway instance that did not shut down cleanly).

**Fix:**
```bash
# Find what is using port 8000
lsof -i :8000
# Kill it by PID
kill -9 <PID>
```

Or use a different port:
```bash
uvicorn app.main:app --host 0.0.0.0 --port 8001 --reload
```

---

### Docker build taking 45+ minutes

**Symptom:** `docker compose up` or `docker build` is very slow on the first run.

**Cause:** The Dockerfile downloads the spaCy `en_core_web_lg` model (~750MB) and installs all Python dependencies from scratch. This is expected on the first build.

**What to know:** After the first successful build, Docker caches the layers. Subsequent builds (when you only change application code) skip the spaCy download entirely and complete in 45–90 seconds. Only changes to `requirements.txt` or the `RUN python -m spacy download` step will cause a slow rebuild.

**Fix:** Wait for the first build to complete. Future builds will be fast.

---

### Dashboard showing mock data when live is expected

**Symptom:** The dashboard shows the same data every time and does not update, even though the gateway is processing requests.

**Cause:** `NEXT_PUBLIC_USE_MOCK_DATA` is set to `true` in `.env.local`.

**Fix:** Open `.env.local` and set:
```bash
NEXT_PUBLIC_USE_MOCK_DATA=false
NEXT_PUBLIC_GATEWAY_URL=http://localhost:8000
```

Then restart the Next.js dev server (Ctrl+C, then `npm run dev`). Next.js does not hot-reload `.env.local` changes.
