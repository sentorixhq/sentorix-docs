# Restore full stack for demo or first customer

Estimated time: 15 minutes from zero to fully live.
Do this the day before any demo — not the morning of.

## When to use this

Run this runbook when you have a demo booked or a customer ready to
integrate. The gateway and dashboard are currently stopped to save
$84/month. The landing page at sentorix.io is on Cloudflare Pages
and is unaffected.

## Pre-check

```bash
aws sts get-caller-identity --profile sentorix --region us-east-1
# Must return account 652063277055
```

## Step 1 — Restore NAT Gateway via Terraform

In `sentorix-infra`, uncomment the NAT Gateway, EIP, and private route
entries in `modules/vpc/main.tf` and `modules/vpc/outputs.tf`.
Also uncomment the WAF module in `environments/dev/main.tf` if needed.

Look for the comment blocks marked:
- `# NAT Gateway and EIP commented out to save $32.85/month pre-customer.`
- `# NAT route commented out — NAT Gateway deleted pre-customer.`
- `# WAF commented out to save $10/month pre-customer.`

```bash
cd /Users/hemanthkp/workspace/sentorix-infra/environments/dev
terraform apply
# Wait 2-3 minutes for NAT Gateway to become available
```

Verify NAT Gateway is available:

```bash
aws ec2 describe-nat-gateways \
  --filter "Name=state,Values=available" \
  --profile sentorix --region us-east-1 \
  --query 'NatGateways[*].{ID:NatGatewayId,State:State}' \
  --output table
```

## Step 2 — Start ECS Gateway

```bash
aws ecs update-service \
  --cluster sentorix-dev-cluster \
  --service sentorix-gateway-service \
  --desired-count 1 \
  --profile sentorix --region us-east-1
```

Wait 3-5 minutes for the spaCy model to load. Watch status:

```bash
aws ecs describe-services \
  --cluster sentorix-dev-cluster \
  --services sentorix-gateway-service \
  --profile sentorix --region us-east-1 \
  --query 'services[0].{Running:runningCount,State:deployments[0].rolloutState}' \
  --output table
```

Also update `modules/ecs/main.tf` `desired_count` back to `1` and commit.

## Step 3 — Start ECS Dashboard

```bash
aws ecs update-service \
  --cluster sentorix-dev-cluster \
  --service sentorix-dashboard-service \
  --desired-count 1 \
  --profile sentorix --region us-east-1
```

## Step 4 — Verify everything is live

```bash
# Gateway health check
curl -s https://api.sentorix.io/v1/health

# Dashboard
curl -s -o /dev/null -w "%{http_code}" https://app.sentorix.io

# Both must return 200
```

## Step 5 — Test PII detection before demo

```bash
curl -s -X POST \
  https://api.sentorix.io/v1/gateway/chat \
  -H "Content-Type: application/json" \
  -H "X-Sentorix-API-Key: your-test-key" \
  -H "X-Tenant-ID: test-tenant" \
  -d '{
    "model": "gpt-4o",
    "messages": [{"role": "user", "content": "My SSN is 123-45-6789"}]
  }'
# Expected: 403 blocked with PII detected
```

## Monthly cost while running

Approximately $122/month with all services active.

## To stop again after demo/customer

```bash
aws ecs update-service --cluster sentorix-dev-cluster \
  --service sentorix-gateway-service \
  --desired-count 0 --profile sentorix --region us-east-1

aws ecs update-service --cluster sentorix-dev-cluster \
  --service sentorix-dashboard-service \
  --desired-count 0 --profile sentorix --region us-east-1
```

Then comment out the NAT Gateway, EIP, route entries, and WAF in Terraform
and run `terraform apply`. Cost drops back to ~$38/month within the hour.
