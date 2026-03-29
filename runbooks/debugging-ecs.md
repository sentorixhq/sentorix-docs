> **Note:** Commands in this document use placeholder variables (e.g. `${AWS_ACCOUNT_ID}`). Replace these with your actual values before running.

# Debugging ECS

Use this runbook when an ECS task is failing, crashing, or behaving unexpectedly. All commands use `--profile sentorix` and `--region us-east-1`.

---

## 1. Checking Service Status

### Quick overview of both services

```bash
aws ecs describe-services \
  --cluster sentorix-cluster \
  --services sentorix-gateway-service sentorix-dashboard-service \
  --query 'services[*].{Service:serviceName,Status:status,Desired:desiredCount,Running:runningCount,Pending:pendingCount}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

**Healthy state:** `Running` equals `Desired` (usually 1).
**Problem indicators:**
- `Running=0, Pending=1` — task is starting (may be stuck if this persists > 5 min)
- `Running=0, Pending=0` — task failed to start and ECS stopped retrying (check events)
- `Running=1, Pending=1` — deployment in progress

### View recent service events

Service events show deployment progress, task start/stop events, and health check failures:

```bash
aws ecs describe-services \
  --cluster sentorix-cluster \
  --services sentorix-gateway-service \
  --query 'services[0].events[:15]' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

Common error messages in events:
- `(service ... is unable to consistently start tasks)` — task is crashing on startup; check logs
- `(instance ... unhealthy)` — ALB health check is failing; app may be crashing or not listening on the right port
- `(service ... has reached a steady state)` — deployment completed successfully

---

## 2. Checking Running Tasks

### List running tasks for the gateway

```bash
aws ecs list-tasks \
  --cluster sentorix-cluster \
  --service-name sentorix-gateway-service \
  --desired-status RUNNING \
  --query 'taskArns' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

### Describe a specific running task

```bash
TASK_ARN=arn:aws:ecs:us-east-1:${AWS_ACCOUNT_ID}:task/sentorix-cluster/TASK_ID

aws ecs describe-tasks \
  --cluster sentorix-cluster \
  --tasks ${TASK_ARN} \
  --query 'tasks[0].{Status:lastStatus,Health:healthStatus,StartedAt:startedAt,TaskDef:taskDefinitionArn,StopCode:stopCode,StopReason:stoppedReason}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

---

## 3. Getting Task Logs from CloudWatch

### Stream live logs (follow mode)

```bash
# Gateway logs
aws logs tail /ecs/sentorix-gateway \
  --follow \
  --format short \
  --profile sentorix \
  --region us-east-1

# Dashboard logs
aws logs tail /ecs/sentorix-dashboard \
  --follow \
  --format short \
  --profile sentorix \
  --region us-east-1
```

Press `Ctrl+C` to stop following.

### Get the last 100 lines of gateway logs

```bash
aws logs tail /ecs/sentorix-gateway \
  --since 1h \
  --format short \
  --profile sentorix \
  --region us-east-1 \
  | tail -100
```

### Search logs for errors in the last 2 hours

```bash
aws logs filter-log-events \
  --log-group-name /ecs/sentorix-gateway \
  --start-time $(date -u -v-2H +%s000 2>/dev/null || date -u -d '2 hours ago' +%s000) \
  --filter-pattern "ERROR" \
  --query 'events[*].{Time:timestamp,Message:message}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

### Get Lambda audit worker logs

```bash
aws logs tail /aws/lambda/sentorix-audit-worker \
  --since 1h \
  --format short \
  --profile sentorix \
  --region us-east-1
```

---

## 4. Describing a Stopped Task and Why It Stopped

When a task keeps stopping, find the stopped task and read its stop reason.

### List recently stopped tasks

```bash
aws ecs list-tasks \
  --cluster sentorix-cluster \
  --service-name sentorix-gateway-service \
  --desired-status STOPPED \
  --query 'taskArns[:5]' \
  --output json \
  --profile sentorix \
  --region us-east-1
```

### Get the stop reason for a specific task

```bash
STOPPED_TASK_ARN=arn:aws:ecs:us-east-1:${AWS_ACCOUNT_ID}:task/sentorix-cluster/TASK_ID

aws ecs describe-tasks \
  --cluster sentorix-cluster \
  --tasks ${STOPPED_TASK_ARN} \
  --query 'tasks[0].{StopCode:stopCode,StopReason:stoppedReason,ExitCode:containers[0].exitCode,ContainerReason:containers[0].reason}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

### Common stop codes and meanings

| StopCode | Meaning | Next step |
|---|---|---|
| `EssentialContainerExited` | The container process exited. | Check `exitCode` and container logs. Exit code 1 = application error. |
| `TaskFailedToStart` | Container could not start. | Check if the image exists in ECR. Check if Secrets Manager secrets are accessible. |
| `OutOfMemoryError` | Container ran out of memory. | Increase task memory in the task definition (gateway needs at least 3GB). |
| `CannotPullContainerError` | ECS could not pull the Docker image. | Check ECR VPC endpoint. Check task execution role has ECR permissions. |
| `ResourceInitializationError` | Secrets Manager or SSM fetch failed at startup. | Check the task execution role permissions and VPC endpoint for Secrets Manager. |

---

## 5. Checking Target Group Health

### Check which targets are healthy

```bash
# Get the gateway target group ARN
TG_ARN=$(aws elbv2 describe-target-groups \
  --names sentorix-gateway-tg \
  --query 'TargetGroups[0].TargetGroupArn' \
  --output text \
  --profile sentorix \
  --region us-east-1)

# Describe target health
aws elbv2 describe-target-health \
  --target-group-arn ${TG_ARN} \
  --query 'TargetHealthDescriptions[*].{Target:Target.Id,Port:Target.Port,State:TargetHealth.State,Reason:TargetHealth.Reason,Description:TargetHealth.Description}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

### Health states and their meanings

| State | Meaning |
|---|---|
| `healthy` | ALB health checks passing. Target is receiving traffic. |
| `unhealthy` | Health check is failing. Check if the app is running and responding on the health check path. |
| `initial` | Target just registered; waiting for first health checks to pass. Normal for first 30–60 seconds after a task starts. |
| `draining` | Target is being deregistered (during a deployment or scale-in). Normal. |
| `unused` | Target is registered but not receiving traffic (e.g., wrong port or AZ). Check task definition port mapping. |

### Gateway health check path

The ALB checks `GET /v1/health` on port 8000 every 30 seconds. If this returns a non-2xx response or times out, the target is marked unhealthy.

To manually test the health check from your machine:

```bash
curl https://api.sentorix.io/v1/health
```

---

## 6. Checking ALB Access Logs

ALB access logs are written to S3 if enabled. Check the bucket:

```bash
aws s3 ls s3://sentorix-alb-logs-${AWS_ACCOUNT_ID}/ \
  --profile sentorix \
  --region us-east-1
```

To download and inspect the most recent log file:

```bash
# List log files sorted by date
aws s3 ls s3://sentorix-alb-logs-${AWS_ACCOUNT_ID}/AWSLogs/${AWS_ACCOUNT_ID}/elasticloadbalancing/us-east-1/ \
  --recursive \
  --profile sentorix \
  --region us-east-1 \
  | sort | tail -5

# Download a specific log file
aws s3 cp "s3://sentorix-alb-logs-${AWS_ACCOUNT_ID}/AWSLogs/${AWS_ACCOUNT_ID}/elasticloadbalancing/us-east-1/YYYY/MM/DD/filename.log.gz" \
  /tmp/alb.log.gz \
  --profile sentorix \
  --region us-east-1

gunzip /tmp/alb.log.gz
# Each line is a tab-separated ALB log entry
# Fields: type, timestamp, alb, client:port, target:port, request_processing_time, target_processing_time, response_processing_time, elb_status_code, target_status_code, received_bytes, sent_bytes, request, user_agent, ssl_cipher, ssl_protocol, target_group_arn, trace_id, domain_name, chosen_cert_arn, matched_rule_priority, request_creation_time, actions_executed, redirect_url, error_reason, target:port_list, target_status_code_list, classification, classification_reason
```

---

## 7. ECS Exec — Shell Into a Running Task

ECS Exec gives you an interactive shell inside a running Fargate task. This is the equivalent of SSH for containerized workloads.

### Prerequisites

ECS Exec must be enabled on the service (it is enabled on Sentorix services). You also need the `session-manager-plugin` installed locally:

```bash
brew install --cask session-manager-plugin
```

### Open an interactive shell

```bash
# Get the task ARN of the running gateway task
TASK_ARN=$(aws ecs list-tasks \
  --cluster sentorix-cluster \
  --service-name sentorix-gateway-service \
  --desired-status RUNNING \
  --query 'taskArns[0]' \
  --output text \
  --profile sentorix \
  --region us-east-1)

echo "Connecting to task: ${TASK_ARN}"

# Open a shell
aws ecs execute-command \
  --cluster sentorix-cluster \
  --task ${TASK_ARN} \
  --container sentorix-gateway \
  --interactive \
  --command "/bin/bash" \
  --profile sentorix \
  --region us-east-1
```

Once connected, you can:

```bash
# Check the application process is running
ps aux | grep uvicorn

# Check environment variables (sanitized — do not share output containing secrets)
env | grep -v KEY | grep -v SECRET

# Test the health endpoint from inside the container
curl localhost:8000/v1/health

# Check if the spaCy model is loaded
python3 -c "import spacy; nlp = spacy.load('en_core_web_lg'); print('OK')"

# Check connectivity to SQS via VPC endpoint
curl https://sqs.us-east-1.amazonaws.com

# Check outbound connectivity to OpenAI
curl https://api.openai.com

# Exit the shell
exit
```

---

## 8. Checking Secrets Manager Permissions

If the task fails to start with `ResourceInitializationError`, it may not be able to read its secrets.

### Verify the secret exists

```bash
aws secretsmanager describe-secret \
  --secret-id sentorix/production/api-keys \
  --profile sentorix \
  --region us-east-1 \
  --query '{Name:Name,ARN:ARN,LastChanged:LastChangedDate}'
```

### Test that the task execution role can read the secret

The gateway task execution role is `sentorix-gateway-task-execution-role`. Verify it has the required policy:

```bash
aws iam list-attached-role-policies \
  --role-name sentorix-gateway-task-execution-role \
  --profile sentorix \
  --region us-east-1

aws iam list-role-policies \
  --role-name sentorix-gateway-task-execution-role \
  --profile sentorix \
  --region us-east-1
```

### Check the ECS task definition secret references

```bash
aws ecs describe-task-definition \
  --task-definition sentorix-gateway \
  --query 'taskDefinition.containerDefinitions[0].secrets' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

This shows the secret names and their Secrets Manager ARNs as configured in the task definition.

---

## 9. Checking VPC Endpoint Connectivity

If the gateway cannot reach SQS, Secrets Manager, or other AWS services, the VPC endpoints may be misconfigured or the security group may be blocking traffic.

### List VPC endpoints and their status

```bash
aws ec2 describe-vpc-endpoints \
  --filters "Name=tag:Project,Values=sentorix" \
  --query 'VpcEndpoints[*].{Service:ServiceName,State:State,Type:VpcEndpointType}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

All endpoints should show `available`. An endpoint in `pending` state is still being created. An endpoint in `deleting` state should not be there.

### Test SQS connectivity from inside the container

Using ECS Exec (see Section 7):

```bash
# From inside the container — test SQS endpoint
curl -sv https://sqs.us-east-1.amazonaws.com 2>&1 | grep -E "(Connected|SSL|curl)"

# Test Secrets Manager endpoint
curl -sv https://secretsmanager.us-east-1.amazonaws.com 2>&1 | grep -E "(Connected|SSL|curl)"
```

A successful connection shows `Connected to sqs.us-east-1.amazonaws.com` and a valid SSL handshake. A timeout or `Could not resolve host` indicates the VPC endpoint is not working.

### Check the VPC endpoint security group

VPC interface endpoints have security groups that must allow inbound HTTPS (port 443) from the ECS task security group:

```bash
# Get the security group IDs attached to the SQS VPC endpoint
aws ec2 describe-vpc-endpoints \
  --filters "Name=service-name,Values=com.amazonaws.us-east-1.sqs" \
  --query 'VpcEndpoints[0].Groups[*].{GroupId:GroupId,GroupName:GroupName}' \
  --output table \
  --profile sentorix \
  --region us-east-1
```

The endpoint security group must have an inbound rule allowing port 443 from the ECS task's security group CIDR or security group ID.
