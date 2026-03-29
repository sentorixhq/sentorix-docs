> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# Manual Deployment

Use this runbook when CI/CD is broken, GitHub Actions is unavailable, or you need to deploy urgently without waiting for the full pipeline.

All commands use `--profile sentorix` and `--region us-east-1`.

**Prerequisites:**
- AWS CLI v2 installed
- Docker installed and running
- `sentorix` AWS profile configured (`aws sts get-caller-identity --profile sentorix` must succeed)
- ECR login active (Step 1 below)

---

## 1. Authenticate to ECR

This step is required before any push or pull from ECR. The token is valid for 12 hours.

```bash
aws ecr get-login-password \
  --region us-east-1 \
  --profile sentorix \
  | docker login \
    --username AWS \
    --password-stdin \
    ${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
```

Expected output: `Login Succeeded`

---

## 2. Manual Gateway Deployment

### Build the image

```bash
cd ~/workspace/sentorix/sentorix-gateway

# Tag with current git SHA (recommended) or a custom label
IMAGE_TAG=$(git rev-parse --short HEAD)
ECR_REPO=${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-gateway

docker build \
  --platform linux/amd64 \
  -t ${ECR_REPO}:${IMAGE_TAG} \
  -t ${ECR_REPO}:latest \
  .
```

**First build warning:** The Dockerfile downloads the spaCy `en_core_web_lg` model (~750MB). First build takes 8–15 minutes. Subsequent builds with Docker layer cache take 45–90 seconds.

### Push the image to ECR

```bash
docker push ${ECR_REPO}:${IMAGE_TAG}
docker push ${ECR_REPO}:latest
```

### Deploy the new image to ECS

```bash
# Step 1: Download the current task definition
aws ecs describe-task-definition \
  --task-definition sentorix-gateway \
  --query taskDefinition \
  --output json \
  --profile sentorix \
  --region us-east-1 \
  > /tmp/gateway-task-def.json

# Step 2: Strip read-only fields that cannot be in a register call
cat /tmp/gateway-task-def.json | python3 -c "
import json, sys
td = json.load(sys.stdin)
for field in ['taskDefinitionArn','revision','status','requiresAttributes',
              'compatibilities','registeredAt','registeredBy']:
    td.pop(field, None)
print(json.dumps(td, indent=2))
" > /tmp/gateway-task-def-clean.json

# Step 3: Update the image in the task definition JSON
# Replace the image URI with the new tag
python3 -c "
import json, sys
with open('/tmp/gateway-task-def-clean.json') as f:
    td = json.load(f)
ECR_REPO = '${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-gateway'
IMAGE_TAG = sys.argv[1]
for container in td['containerDefinitions']:
    if 'sentorix-gateway' in container.get('image', ''):
        container['image'] = f'{ECR_REPO}:{IMAGE_TAG}'
with open('/tmp/gateway-task-def-updated.json', 'w') as f:
    json.dump(td, f, indent=2)
print('Updated image to:', f'{ECR_REPO}:{IMAGE_TAG}')
" ${IMAGE_TAG}

# Step 4: Register new task definition revision
NEW_REVISION=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/gateway-task-def-updated.json \
  --query 'taskDefinition.revision' \
  --output text \
  --profile sentorix \
  --region us-east-1)

echo "Registered task definition revision: ${NEW_REVISION}"

# Step 5: Update the ECS service
aws ecs update-service \
  --cluster sentorix-cluster \
  --service sentorix-gateway-service \
  --task-definition sentorix-gateway:${NEW_REVISION} \
  --profile sentorix \
  --region us-east-1

echo "Deployment started. Waiting for service to stabilize..."

# Step 6: Wait for deployment to complete
aws ecs wait services-stable \
  --cluster sentorix-cluster \
  --services sentorix-gateway-service \
  --profile sentorix \
  --region us-east-1

echo "Deployment complete."
```

### Verify the deployment

```bash
curl https://api.sentorix.io/v1/health
```

Expected: `{"status": "ok", "version": "..."}` with HTTP 200.

---

## 3. Manual Dashboard Deployment

```bash
cd ~/workspace/sentorix/sentorix-dashboard

IMAGE_TAG=$(git rev-parse --short HEAD)
ECR_REPO=${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-dashboard

# Build
docker build \
  --platform linux/amd64 \
  -t ${ECR_REPO}:${IMAGE_TAG} \
  -t ${ECR_REPO}:latest \
  .

# Push
docker push ${ECR_REPO}:${IMAGE_TAG}
docker push ${ECR_REPO}:latest

# Download and clean task definition
aws ecs describe-task-definition \
  --task-definition sentorix-dashboard \
  --query taskDefinition \
  --output json \
  --profile sentorix \
  --region us-east-1 \
  > /tmp/dashboard-task-def.json

cat /tmp/dashboard-task-def.json | python3 -c "
import json, sys
td = json.load(sys.stdin)
for field in ['taskDefinitionArn','revision','status','requiresAttributes',
              'compatibilities','registeredAt','registeredBy']:
    td.pop(field, None)
print(json.dumps(td, indent=2))
" > /tmp/dashboard-task-def-clean.json

python3 -c "
import json, sys
with open('/tmp/dashboard-task-def-clean.json') as f:
    td = json.load(f)
ECR_REPO = '${AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/sentorix-dashboard'
IMAGE_TAG = sys.argv[1]
for container in td['containerDefinitions']:
    if 'sentorix-dashboard' in container.get('image', ''):
        container['image'] = f'{ECR_REPO}:{IMAGE_TAG}'
with open('/tmp/dashboard-task-def-updated.json', 'w') as f:
    json.dump(td, f, indent=2)
" ${IMAGE_TAG}

NEW_REVISION=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/dashboard-task-def-updated.json \
  --query 'taskDefinition.revision' \
  --output text \
  --profile sentorix \
  --region us-east-1)

aws ecs update-service \
  --cluster sentorix-cluster \
  --service sentorix-dashboard-service \
  --task-definition sentorix-dashboard:${NEW_REVISION} \
  --profile sentorix \
  --region us-east-1

aws ecs wait services-stable \
  --cluster sentorix-cluster \
  --services sentorix-dashboard-service \
  --profile sentorix \
  --region us-east-1

echo "Dashboard deployment complete."
curl -L https://app.sentorix.io
```

---

## 4. Update ECS to a Specific Image Tag (Rollback)

Use this to point ECS at a previously built image without rebuilding anything.

```bash
# List recent images in ECR to find available tags
aws ecr describe-images \
  --repository-name sentorix-gateway \
  --query 'sort_by(imageDetails, &imagePushedAt)[-10:].{Tag:imageTags[0],Pushed:imagePushedAt}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

Once you have the target tag (e.g., `a1b2c3d`):

```bash
TARGET_TAG=a1b2c3d  # the git SHA of the version you want

# Download, clean, and update task def to use the target tag
aws ecs describe-task-definition \
  --task-definition sentorix-gateway \
  --query taskDefinition \
  --output json \
  --profile sentorix \
  --region us-east-1 \
  | python3 -c "
import json, sys
td = json.load(sys.stdin)
for field in ['taskDefinitionArn','revision','status','requiresAttributes',
              'compatibilities','registeredAt','registeredBy']:
    td.pop(field, None)
print(json.dumps(td))
" > /tmp/rollback-task-def.json

python3 -c "
import json, sys
with open('/tmp/rollback-task-def.json') as f:
    td = json.load(f)
tag = sys.argv[1]
for c in td['containerDefinitions']:
    if 'sentorix-gateway' in c.get('image',''):
        base = c['image'].rsplit(':',1)[0]
        c['image'] = f'{base}:{tag}'
        print('Rolling back to:', c['image'])
with open('/tmp/rollback-task-def.json','w') as f:
    json.dump(td, f)
" ${TARGET_TAG}

NEW_REVISION=$(aws ecs register-task-definition \
  --cli-input-json file:///tmp/rollback-task-def.json \
  --query 'taskDefinition.revision' \
  --output text \
  --profile sentorix \
  --region us-east-1)

aws ecs update-service \
  --cluster sentorix-cluster \
  --service sentorix-gateway-service \
  --task-definition sentorix-gateway:${NEW_REVISION} \
  --profile sentorix \
  --region us-east-1

aws ecs wait services-stable \
  --cluster sentorix-cluster \
  --services sentorix-gateway-service \
  --profile sentorix \
  --region us-east-1

echo "Rollback to ${TARGET_TAG} complete."
```

---

## 5. Checking Deployment Status

### Check which image is currently running

```bash
aws ecs describe-services \
  --cluster sentorix-cluster \
  --services sentorix-gateway-service sentorix-dashboard-service \
  --query 'services[*].{Service:serviceName,Desired:desiredCount,Running:runningCount,Pending:pendingCount,TaskDef:taskDefinition}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

### Check recent deployment events

```bash
aws ecs describe-services \
  --cluster sentorix-cluster \
  --services sentorix-gateway-service \
  --query 'services[0].events[:10]' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

### Check what image the current task definition uses

```bash
aws ecs describe-task-definition \
  --task-definition sentorix-gateway \
  --query 'taskDefinition.containerDefinitions[0].image' \
  --output text \
  --profile sentorix \
  --region us-east-1
```

---

## 6. Verifying Health After Manual Deploy

```bash
# Gateway
curl -s https://api.sentorix.io/v1/health | python3 -m json.tool

# Dashboard (expect 200 or 307 redirect)
curl -sI https://app.sentorix.io | head -5

# ALB target group health
aws elbv2 describe-target-health \
  --target-group-arn $(aws elbv2 describe-target-groups \
    --names sentorix-gateway-tg \
    --query 'TargetGroups[0].TargetGroupArn' \
    --output text \
    --profile sentorix \
    --region us-east-1) \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,Health:TargetHealth.State}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

The target group health should show `healthy` for the gateway target. If it shows `unhealthy` or `initial`, check the task logs (see [debugging-ecs.md](debugging-ecs.md)).
