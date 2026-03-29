> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# CI/CD Pipeline

---

## 1. Overview

At Sentorix, every push to the `main` branch of the gateway or dashboard repositories triggers an automated pipeline that tests, builds, and deploys to production. There is no manual promotion step between environments — `main` is production.

Three pipelines exist:

| Pipeline | Repository | File | Trigger |
|---|---|---|---|
| Gateway | sentorix-gateway | `.github/workflows/deploy-gateway.yml` | Push to main (path-filtered) |
| Dashboard | sentorix-dashboard | `.github/workflows/deploy-dashboard.yml` | Push to main (path-filtered) |
| Infrastructure | sentorix-infra | `.github/workflows/terraform.yml` | PR to main (plan) + merge to main (apply) |

**The principle:** Merge to main = deployed to production. Deployments are zero-downtime rolling updates via ECS. The old task continues serving traffic until the new task passes health checks.

---

## 2. Authentication

### Why OIDC instead of access keys

Traditional CI/CD setups store long-lived AWS access keys as GitHub Secrets. These keys are permanent credentials — if they leak (via a compromised secret, a log line, or a supply chain attack), an attacker has persistent AWS access until the key is manually revoked and rotated.

Sentorix uses OpenID Connect (OIDC) instead. There are no long-lived AWS access keys anywhere in the GitHub configuration.

### How OIDC works

1. When a GitHub Actions workflow runs, GitHub generates a short-lived OIDC token signed by GitHub's identity provider. This token contains claims about the workflow: the repository name, the branch, the workflow file path, and the triggering event.

2. The GitHub Actions step calls `aws-actions/configure-aws-credentials` with the IAM role ARN. This action presents the OIDC token to AWS STS.

3. AWS STS verifies the token signature against GitHub's OIDC provider (configured once in the AWS account). It checks that the token's claims match the conditions on the IAM role trust policy (for example, that the repository is `sentorixhq/sentorix-gateway` and the branch is `main`).

4. AWS STS issues temporary credentials (access key + secret key + session token) valid for 1 hour. These are injected as environment variables into the workflow step.

5. The workflow steps use these temporary credentials to interact with AWS. After 1 hour, the credentials expire automatically — there is nothing to rotate or revoke.

### IAM Role

**Role name:** `sentorix-github-actions-role`
**AWS Account:** ${AWS_ACCOUNT_ID}
**Trust policy:** Only workflow runs from the `sentorixhq` GitHub organization on the `main` branch can assume this role.

### Verifying OIDC is working

In any pipeline run, look for the step "Configure AWS credentials". If it succeeds and shows an assumed role ARN, OIDC is working correctly. If it fails, check:
- The AWS OIDC provider is configured in the account for `token.actions.githubusercontent.com`
- The IAM role trust policy allows the specific repository and branch
- The workflow is running on a branch/event that matches the trust policy conditions

---

## 3. Gateway Pipeline (`deploy-gateway.yml`)

### When it triggers

The pipeline runs on push to `main` when any of these paths change:

```yaml
paths:
  - 'app/**'          # Any application code
  - 'requirements.txt'
  - 'Dockerfile'
  - '.github/workflows/deploy-gateway.yml'
```

Path filtering prevents unnecessary deployments. If you only change a README or a test file, the deployment pipeline does not run.

The pipeline can also be triggered manually.

### How to trigger manually

1. Go to [github.com/sentorixhq/sentorix-gateway/actions](https://github.com/sentorixhq/sentorix-gateway/actions)
2. Click "Deploy Gateway" in the left sidebar
3. Click "Run workflow" (top right of the workflow list)
4. Select the branch (usually `main`)
5. Click "Run workflow"

This is useful when you need to redeploy without making a code change — for example, after updating a secret in Secrets Manager.

---

### Job 1: `test`

**Runs on:** `ubuntu-latest`

**Steps:**
1. Check out code
2. Install Python 3.12
3. Install `requirements.txt` and `requirements-dev.txt`
4. Download spaCy `en_core_web_sm` (the smaller model, for CI speed — the large model is baked into the Docker image)
5. Run `pytest tests/unit/ -v --tb=short`
6. Upload coverage report as a GitHub Actions artifact (available for 30 days)

**If this job fails:** The build-and-push job does not start. No Docker image is built. No deployment happens. Fix the failing test and push again.

**Why `en_core_web_sm` in CI:** The `en_core_web_lg` model is ~750MB and takes 5–10 minutes to download. In CI, we run unit tests with the small model to keep the pipeline fast. The unit tests mock Presidio's output, so the actual model size does not affect test correctness. The large model is still baked into the Docker image for production use.

---

### Job 2: `build-and-push`

**Runs on:** `ubuntu-latest`
**Depends on:** `test` job passing

**Steps:**
1. Check out code
2. Configure AWS credentials via OIDC
3. Log in to Amazon ECR
4. Set up Docker Buildx with GitHub Actions cache
5. Generate image tag from git SHA: `$(git rev-parse --short HEAD)` (first 7 characters, e.g., `a1b2c3d`)
6. Build Docker image for `linux/amd64` platform
   - Build cache: uses `type=gha` (GitHub Actions cache) to cache Docker layers
   - First build after a cache miss: 8–15 minutes (downloads spaCy model)
   - Cached builds: 45–90 seconds
7. Push two tags to ECR:
   - `latest` — always points to the most recent build
   - `{git-sha}` — permanent, immutable tag for this exact build (e.g., `a1b2c3d`)

**Output:** The image tag (git SHA) is passed as an output to the deploy job.

**Why two tags:** `latest` is convenient for humans checking what the current version is. The SHA tag is what gets deployed — it is immutable and makes rollbacks deterministic. "Roll back to `a1b2c3d`" is unambiguous.

---

### Job 3: `deploy`

**Runs on:** `ubuntu-latest`
**Depends on:** `build-and-push` job passing
**Environment:** `production` (GitHub environment — can be gated with required reviewers)

**Steps:**
1. Configure AWS credentials via OIDC
2. Download current ECS task definition JSON from AWS:
   ```bash
   aws ecs describe-task-definition \
     --task-definition sentorix-gateway \
     --query taskDefinition \
     --output json > task-def.json
   ```
3. Strip read-only fields from the JSON that cannot be included in a `register-task-definition` call (fields like `taskDefinitionArn`, `revision`, `status`, `requiresAttributes`, `compatibilities`, `registeredAt`, `registeredBy`)
4. Update the container image URL in the JSON to the new SHA tag:
   ```
   ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-gateway:a1b2c3d
   ```
5. Register new task definition revision with AWS ECS
6. Update the ECS service to use the new task definition revision:
   ```bash
   aws ecs update-service \
     --cluster sentorix-cluster \
     --service sentorix-gateway-service \
     --task-definition sentorix-gateway:{new-revision}
   ```
7. Wait for the rolling deployment to complete:
   ```bash
   aws ecs wait services-stable \
     --cluster sentorix-cluster \
     --services sentorix-gateway-service
   ```
   ECS performs a rolling update: starts a new task with the new image, waits for it to pass the ALB health check, then drains and stops the old task. Zero downtime.
8. Verify deployment health:
   ```bash
   curl --fail https://api.sentorix.io/v1/health
   ```
   Returns non-zero exit code if the health check fails, marking the deployment as failed.

### Total pipeline time

| Scenario | Estimated Time |
|---|---|
| First run (no Docker cache) | 20–25 minutes |
| Subsequent runs (with cache) | 8–12 minutes |
| Tests only (no code change) | 3–4 minutes |

---

## 4. Dashboard Pipeline (`deploy-dashboard.yml`)

### When it triggers

```yaml
paths:
  - 'src/**'
  - 'public/**'
  - 'package.json'
  - 'package-lock.json'
  - 'next.config.ts'
  - 'Dockerfile'
```

### The two jobs

**Job 1: `build-and-push`**

Same pattern as the gateway. Builds a Next.js Docker image and pushes to ECR with `latest` and `{git-sha}` tags.

**Job 2: `deploy`**

Same pattern as the gateway deploy job — downloads task definition, updates image, registers new revision, updates service, waits for stability.

Health check verification: `curl --fail https://app.sentorix.io` (accepts HTTP 200 or 307 redirect to login).

### Key differences from the gateway pipeline

- **No test job.** The dashboard does not yet have a test suite. When tests are written, add a test job before build-and-push.
- **Faster build.** The Next.js image has no large model downloads. First build: 3–5 minutes. Subsequent builds: 1–2 minutes.
- **Different health check URL.** Uses `app.sentorix.io` not `api.sentorix.io`.

---

## 5. Terraform Pipeline (`terraform.yml`)

### When it triggers

- **Pull request to main:** runs `plan` only, comments the plan output on the PR
- **Push to main (merge):** runs `plan` then `apply`

### On a pull request

When you open a PR that changes Terraform files, the pipeline:

1. `terraform init` — initializes providers and backend
2. `terraform validate` — checks HCL syntax
3. `terraform fmt -check` — fails if any `.tf` file is not properly formatted (fix with `terraform fmt` locally)
4. `terraform plan` — generates an execution plan showing what will be created, modified, or destroyed
5. Posts the plan output as a comment on your PR

**Review the plan comment on your PR before merging.** This is your opportunity to catch unintended infrastructure changes — for example, if a seemingly small code change would accidentally destroy a database.

### On merge to main

The same steps run, followed by:

5. `terraform apply -auto-approve` — applies the plan. No manual confirmation step.

Changes are applied within minutes of merging.

### How to preview infrastructure changes safely

```bash
# Make changes to Terraform files
git checkout -b infra/add-new-vpc-endpoint
# ... edit terraform files ...
git push origin infra/add-new-vpc-endpoint
# Open a PR — GitHub Actions posts the plan as a PR comment
# Review the plan: check for any unexpected destroys or replacements
# Merge when the plan looks correct
# terraform apply runs automatically
```

---

## 6. Rolling Back a Deployment

### Option A — Revert the code (preferred)

```bash
git revert HEAD
git push origin main
```

This creates a new commit that undoes the last commit, then triggers a new pipeline that builds and deploys the reverted code. The revert commit is recorded in git history, which is the preferred approach for traceability.

### Option B — Force deploy a previous image tag

Use this when you need to restore service immediately without waiting for a full pipeline run.

1. Find the previous deployment's git SHA in GitHub Actions history (look at the last successful "build-and-push" job — the image tag is shown in the output).

2. Identify the previous ECS task definition revision number:
   ```bash
   aws ecs describe-services \
     --cluster sentorix-cluster \
     --services sentorix-gateway-service \
     --profile sentorix \
     --region us-east-1 \
     --query 'services[0].taskDefinition'
   ```

3. Update the service to use the previous revision:
   ```bash
   aws ecs update-service \
     --cluster sentorix-cluster \
     --service sentorix-gateway-service \
     --task-definition sentorix-gateway:{previous-revision-number} \
     --profile sentorix \
     --region us-east-1
   ```

4. Wait for deployment to complete:
   ```bash
   aws ecs wait services-stable \
     --cluster sentorix-cluster \
     --services sentorix-gateway-service \
     --profile sentorix \
     --region us-east-1
   ```

5. Verify:
   ```bash
   curl https://api.sentorix.io/v1/health
   ```

---

## 7. Monitoring Deployments

### GitHub Actions

| Pipeline | URL |
|---|---|
| Gateway | [github.com/sentorixhq/sentorix-gateway/actions](https://github.com/sentorixhq/sentorix-gateway/actions) |
| Dashboard | [github.com/sentorixhq/sentorix-dashboard/actions](https://github.com/sentorixhq/sentorix-dashboard/actions) |
| Infrastructure | [github.com/sentorixhq/sentorix-infra/actions](https://github.com/sentorixhq/sentorix-infra/actions) |

### CloudWatch

For ECS deployment events and task health:

```
AWS Console → CloudWatch → Dashboards → Sentorix
```

ECS service events (task starts, stops, deployment progress) are visible in:
```
AWS Console → ECS → Clusters → sentorix-cluster → Services → sentorix-gateway-service → Events
```

---

## 8. Required GitHub Secrets

These secrets must be configured in each repository's Settings → Secrets and variables → Actions.

### sentorix-gateway

| Secret Name | Value | Where to get it |
|---|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::${AWS_ACCOUNT_ID}:role/sentorix-github-actions-role` | AWS IAM console |
| `ECR_REGISTRY` | `${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com` | AWS ECR console |

### sentorix-dashboard

| Secret Name | Value | Where to get it |
|---|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::${AWS_ACCOUNT_ID}:role/sentorix-github-actions-role` | AWS IAM console |
| `ECR_REGISTRY` | `${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com` | AWS ECR console |

### sentorix-infra

| Secret Name | Value | Where to get it |
|---|---|---|
| `AWS_ROLE_ARN` | `arn:aws:iam::${AWS_ACCOUNT_ID}:role/sentorix-github-actions-role` | AWS IAM console |
| `TF_STATE_BUCKET` | `sentorix-terraform-state-${AWS_ACCOUNT_ID}` | Terraform backend config |

### How to add a secret

1. Go to the repository on GitHub
2. Settings → Secrets and variables → Actions
3. Click "New repository secret"
4. Enter the name and value
5. Click "Add secret"

Secrets are encrypted and only visible to GitHub Actions — they are never shown in plaintext after creation.

---

## 9. GitHub Environments

The `deploy` job in both the gateway and dashboard pipelines runs against the `production` GitHub environment.

### What environments provide

- **Deployment tracking:** GitHub records each deployment to the `production` environment, showing which commit is currently deployed.
- **Required reviewers (optional):** You can require one or more people to approve a deployment before it proceeds. Currently not configured — all merges to main deploy automatically.
- **Wait timer (optional):** A configurable delay before deployment starts, giving time to abort if a mistake is caught.

### Configuring required reviewers

To require approval before production deployments:

1. Go to the repository → Settings → Environments → `production`
2. Under "Required reviewers", add team members who must approve
3. Save

After this change, the deploy job will pause and wait for an approval in the GitHub Actions UI before continuing.

### Emergency bypass

If you need to deploy immediately without waiting for reviewers (e.g., a critical security fix):

1. Go to the paused workflow run
2. Click "Review deployments"
3. Select `production`
4. Click "Approve and deploy"

Any team member with write access to the repository can approve, even without being in the required reviewers list, if the "Prevent self-review" option is not enabled.
