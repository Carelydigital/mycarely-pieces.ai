# Production Environment — ECS Bootstrap

Production deployment of Activepieces on **AWS ECS (EC2 launch type)** in the existing `ConnectorsAppCluster`. CI builds an image on every merged PR to `main`, pushes to ECR, and rolls the ECS services forward. Postgres runs on RDS, Redis on ElastiCache, file storage on S3, ingress through an ALB fronted by Cloudflare.

This guide walks through the **one-time setup** of every missing piece and the **ongoing CI/CD** loop. It is opinionated — where there's a choice, we explain the trade-off inline and pick one.

> **Two ways to apply this.** This document uses `aws` CLI commands as a narrative walkthrough — good for learning the stack and for one-off surgery. For the actual bring-up, prefer the numbered CloudFormation templates in `cloudformation/` (see `cloudformation/README.md`); they encode exactly the same resources with stack-level rollback and drift detection. The manual steps this doc calls out (ALB listener, ACM cert, Cloudflare CNAME, worker token generation) are the same regardless of which path you take.

## 1. Architecture overview

```
                  ┌──────────────┐
   Internet  ───▶ │  Cloudflare  │ ──CNAME──▶  <PROD_DOMAIN>
                  │   (DNS/WAF)  │
                  └──────┬───────┘
                         ▼
                  ┌──────────────┐
                  │     ALB      │  (HTTPS:443, ACM cert)
                  │  + TG :80    │
                  └──────┬───────┘
                         ▼
   ┌──────────── ConnectorsAppCluster (ECS on EC2) ────────────┐
   │                                                           │
   │   ┌─────────────────────────┐   ┌────────────────────┐    │
   │   │ Service: APP            │   │ Service: WORKER    │    │
   │   │  ConnectorsECSServices3 │   │  ConnectorsECSSer… │    │
   │   │  -ActivePieces-<RAND>   │   │  -ActivePiecesWor… │    │
   │   │  (desired count: 1+)    │   │  (desired count: 2+)│   │
   │   └──────────┬──────────────┘   └─────────┬──────────┘    │
   │              │                            │               │
   └──────────────┼────────────────────────────┼───────────────┘
                  │ (reads/writes)             │ (HTTPS to ALB)
                  ▼                            │
   ┌──────────────────────┐   ┌─────────────────┐
   │ RDS Postgres 14      │   │ ElastiCache     │
   │ activepieces-prod-db │   │ Redis 7         │
   └──────────────────────┘   └─────────────────┘

   S3: connectors-activepieces-prod-files    (flow run logs, uploads)
   Secrets Manager: /activepieces/prod/*     (JWT, encryption, DB, worker token)
```

**Key invariants:**

- The **APP service** is the only service behind the ALB. Workers never receive external traffic; they outbound-call the APP over HTTPS.
- Workers **do not** touch RDS or Redis directly — they authenticate to the APP with a JWT worker token signed by `AP_JWT_SECRET`. This means the RDS security group must **not** allow the worker SG.
- APP runs migrations on boot. Deployments must order **APP → WORKER** so workers never start against a DB schema older than their code.
- Storage of truth: RDS for state, S3 for files, Redis for queue + cache (ephemeral). If Redis is wiped, in-flight runs are lost but no data.

## 2. Prerequisites & reference values

| Name | Value |
| --- | --- |
| AWS account | `557690597652` |
| Region | `eu-central-1` |
| ECS cluster | `ConnectorsAppCluster` (already exists) |
| ECR repo ARN | `arn:aws:ecr:eu-central-1:557690597652:repository/connectors/activepieces` (already exists) |
| ECR image URI | `557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces` |
| Domain | `<PROD_DOMAIN>` — replace throughout, and in Cloudflare for the CNAME |
| Trigger | merged PRs to `main` (unmerged PR closes do **not** deploy) |

Install locally: `aws` CLI v2, `jq`, `docker` with buildx, and authenticate (`aws configure` or SSO profile) against the target account.

**Missing, created in this guide**: VPC resource discovery, 4 security groups, ACM cert, RDS instance + subnet group + param group, ElastiCache cluster + subnet group, S3 bucket, 4 Secrets Manager entries, 3 IAM roles, 2 CloudWatch log groups, ASG + launch template + capacity provider, ALB + target group + listener, Cloudflare CNAME, worker token, 2 task definitions, 2 ECS services, 1 CI workflow.

## 3. Networking

### 3.1 Discover VPC and subnets from the existing cluster

The cluster already has EC2 capacity hosting `ConnectorsECSServices2-FlerpService`. Reuse its VPC so RDS, ElastiCache, and the new services share the same network.

```bash
# Find a container instance in the cluster and derive its VPC + subnets
INSTANCE_ARN=$(aws ecs list-container-instances \
  --cluster ConnectorsAppCluster --region eu-central-1 \
  --query 'containerInstanceArns[0]' --output text)

EC2_ID=$(aws ecs describe-container-instances \
  --cluster ConnectorsAppCluster \
  --container-instances "$INSTANCE_ARN" \
  --query 'containerInstances[0].ec2InstanceId' --output text \
  --region eu-central-1)

aws ec2 describe-instances --instance-ids "$EC2_ID" --region eu-central-1 \
  --query 'Reservations[0].Instances[0].{VpcId:VpcId,Subnets:[NetworkInterfaces[].SubnetId]}'
```

Record these as `VPC_ID`, `PRIVATE_SUBNETS=<subnet-a>,<subnet-b>`, `PUBLIC_SUBNETS=<subnet-c>,<subnet-d>` for use in the rest of the guide. ALB needs public subnets; tasks, RDS, and Redis sit in private subnets.

If the existing cluster lives in public subnets only (check route tables for a default route via IGW), document it and either add private subnets now or accept that RDS/Redis will be in public subnets with tight SG rules. For prod, **add private subnets** if they don't exist — NAT Gateway + private subnet is cheap compared to a data leak.

### 3.2 Security groups

Four SGs, each tightly scoped. Create them in order:

```bash
# ALB — inbound 443 from the internet (or Cloudflare IPs; see note)
ALB_SG=$(aws ec2 create-security-group \
  --group-name activepieces-prod-alb \
  --description "Activepieces prod ALB" \
  --vpc-id $VPC_ID --region eu-central-1 \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $ALB_SG \
  --protocol tcp --port 443 --cidr 0.0.0.0/0 --region eu-central-1

# Tasks — inbound 80 from ALB only
TASKS_SG=$(aws ec2 create-security-group \
  --group-name activepieces-prod-tasks \
  --description "Activepieces prod ECS tasks (APP+WORKER)" \
  --vpc-id $VPC_ID --region eu-central-1 \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $TASKS_SG \
  --protocol tcp --port 80 --source-group $ALB_SG --region eu-central-1

# RDS — inbound 5432 from tasks SG only (APP talks to it, WORKER does not)
RDS_SG=$(aws ec2 create-security-group \
  --group-name activepieces-prod-rds \
  --description "Activepieces prod RDS" \
  --vpc-id $VPC_ID --region eu-central-1 \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $RDS_SG \
  --protocol tcp --port 5432 --source-group $TASKS_SG --region eu-central-1

# Redis — inbound 6379 from tasks SG only
REDIS_SG=$(aws ec2 create-security-group \
  --group-name activepieces-prod-redis \
  --description "Activepieces prod ElastiCache" \
  --vpc-id $VPC_ID --region eu-central-1 \
  --query 'GroupId' --output text)
aws ec2 authorize-security-group-ingress --group-id $REDIS_SG \
  --protocol tcp --port 6379 --source-group $TASKS_SG --region eu-central-1
```

> **Cloudflare-only ingress (hardening):** Cloudflare proxies traffic from ~20 IP ranges published at `https://www.cloudflare.com/ips-v4/`. For a tighter setup, replace the `0.0.0.0/0` ALB rule with CIDRs from that list. Keep a runbook for rotation; Cloudflare updates it occasionally.

## 4. ACM certificate (DNS validation via Cloudflare)

ACM issues the TLS cert for the ALB. DNS validation requires creating a CNAME in Cloudflare — Route 53 is not used in this project.

```bash
CERT_ARN=$(aws acm request-certificate \
  --domain-name <PROD_DOMAIN> \
  --validation-method DNS \
  --region eu-central-1 \
  --query CertificateArn --output text)

# Wait a few seconds, then fetch the validation record
aws acm describe-certificate --certificate-arn $CERT_ARN --region eu-central-1 \
  --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
```

The `ResourceRecord` output contains `Name`, `Type` (`CNAME`), and `Value`. In Cloudflare:

1. Open the zone, **DNS → Records → Add record**.
2. Type: `CNAME`, Name: copy from `Name` (strip the domain suffix Cloudflare adds automatically), Target: copy from `Value`.
3. **Proxy status: DNS only** (grey cloud) — ACM validation will not resolve through Cloudflare's proxy.
4. TTL: Auto.

Validation usually completes within 5 minutes. Track it:

```bash
aws acm describe-certificate --certificate-arn $CERT_ARN --region eu-central-1 \
  --query 'Certificate.Status'   # expect "ISSUED"
```

The validation CNAME can be deleted after issuance, but ACM's auto-renewal also uses DNS, so leaving it in Cloudflare is free and future-proof.

## 5. RDS Postgres

### 5.1 Subnet group + parameter group

```bash
aws rds create-db-subnet-group \
  --db-subnet-group-name activepieces-prod \
  --db-subnet-group-description "Activepieces prod" \
  --subnet-ids $(echo $PRIVATE_SUBNETS | tr ',' ' ') \
  --region eu-central-1

aws rds create-db-parameter-group \
  --db-parameter-group-name activepieces-prod-pg14 \
  --db-parameter-group-family postgres14 \
  --description "Activepieces prod Postgres 14" \
  --region eu-central-1
```

### 5.2 DB instance

`db.t4g.small` (2 vCPU, 2 GB) is a defensible starting point. Encryption at rest on, backups on, Multi-AZ **off** initially to keep cost low (~$40–60/mo) — flip to Multi-AZ when SLAs demand.

```bash
DB_PASSWORD=$(openssl rand -hex 32)
aws secretsmanager create-secret \
  --name /activepieces/prod/postgres-password \
  --secret-string "$DB_PASSWORD" --region eu-central-1

aws rds create-db-instance \
  --db-instance-identifier activepieces-prod-db \
  --db-instance-class db.t4g.small \
  --engine postgres --engine-version 14.12 \
  --master-username activepieces \
  --master-user-password "$DB_PASSWORD" \
  --allocated-storage 20 --storage-type gp3 --storage-encrypted \
  --db-name activepieces \
  --vpc-security-group-ids $RDS_SG \
  --db-subnet-group-name activepieces-prod \
  --db-parameter-group-name activepieces-prod-pg14 \
  --backup-retention-period 7 \
  --no-publicly-accessible \
  --deletion-protection \
  --region eu-central-1

# Wait for 'available' and capture the endpoint
aws rds wait db-instance-available \
  --db-instance-identifier activepieces-prod-db --region eu-central-1
DB_HOST=$(aws rds describe-db-instances \
  --db-instance-identifier activepieces-prod-db \
  --region eu-central-1 \
  --query 'DBInstances[0].Endpoint.Address' --output text)
echo "DB host: $DB_HOST"
```

> `--deletion-protection` is cheap insurance. To drop the instance later you must disable protection first — a speed bump, not a lock.

## 6. ElastiCache Redis

Single-node `cache.t4g.small` for starters (~$15/mo). Switch to a replication group with a replica when the queue workload justifies it.

```bash
aws elasticache create-cache-subnet-group \
  --cache-subnet-group-name activepieces-prod \
  --cache-subnet-group-description "Activepieces prod" \
  --subnet-ids $(echo $PRIVATE_SUBNETS | tr ',' ' ') \
  --region eu-central-1

aws elasticache create-cache-cluster \
  --cache-cluster-id activepieces-prod-redis \
  --engine redis --engine-version 7.0 \
  --cache-node-type cache.t4g.small \
  --num-cache-nodes 1 \
  --cache-subnet-group-name activepieces-prod \
  --security-group-ids $REDIS_SG \
  --region eu-central-1

aws elasticache wait cache-cluster-available \
  --cache-cluster-id activepieces-prod-redis --region eu-central-1

REDIS_HOST=$(aws elasticache describe-cache-clusters \
  --cache-cluster-id activepieces-prod-redis \
  --show-cache-node-info --region eu-central-1 \
  --query 'CacheClusters[0].CacheNodes[0].Endpoint.Address' --output text)
echo "Redis host: $REDIS_HOST"
```

Note: no auth/TLS on this cluster for simplicity. Because the Redis SG is scoped to the tasks SG only, traffic never leaves the VPC. If compliance requires in-transit encryption, recreate with `--transit-encryption-enabled` and set `AP_REDIS_USE_SSL=true`.

## 7. S3 bucket for file storage

Flow run logs and uploaded files ≥ 25 MB go to S3 when `AP_FILE_STORAGE_LOCATION=S3`.

```bash
BUCKET=connectors-activepieces-prod-files
aws s3api create-bucket --bucket $BUCKET \
  --region eu-central-1 \
  --create-bucket-configuration LocationConstraint=eu-central-1

aws s3api put-bucket-encryption --bucket $BUCKET \
  --server-side-encryption-configuration \
    '{"Rules":[{"ApplyServerSideEncryptionByDefault":{"SSEAlgorithm":"AES256"}}]}'

aws s3api put-public-access-block --bucket $BUCKET \
  --public-access-block-configuration \
    BlockPublicAcls=true,IgnorePublicAcls=true,BlockPublicPolicy=true,RestrictPublicBuckets=true

aws s3api put-bucket-versioning --bucket $BUCKET \
  --versioning-configuration Status=Enabled

# Lifecycle: expire old flow-run log objects after 90 days to cap storage cost
aws s3api put-bucket-lifecycle-configuration --bucket $BUCKET \
  --lifecycle-configuration '{"Rules":[{"ID":"ExpireLogs","Status":"Enabled","Filter":{"Prefix":"flow-runs/"},"Expiration":{"Days":90}}]}'
```

The bucket stays private; the APP uses signed URLs (`AP_S3_USE_SIGNED_URLS=true`) to hand temporary download links to the browser without making objects public.

## 8. Secrets Manager entries

Four secrets. The task definitions reference them by ARN — never inline.

```bash
aws secretsmanager create-secret --name /activepieces/prod/jwt-secret \
  --secret-string "$(openssl rand -hex 32)" --region eu-central-1

aws secretsmanager create-secret --name /activepieces/prod/encryption-key \
  --secret-string "$(openssl rand -hex 16)" --region eu-central-1

# Postgres password was stored in step 5.2 — skip if you already did it
# aws secretsmanager create-secret --name /activepieces/prod/postgres-password ...

# Worker token — generated in §13 after we know AP_JWT_SECRET.
# Create a placeholder now so the task def ARN resolves; we'll overwrite the value:
aws secretsmanager create-secret --name /activepieces/prod/worker-token \
  --secret-string "PLACEHOLDER_SET_IN_STEP_13" --region eu-central-1
```

> **Back up `AP_ENCRYPTION_KEY` offline.** If you lose it, every stored connection in the DB (API keys, OAuth tokens, webhook secrets) becomes unreadable — there is no recovery path. Store a copy in a password manager separate from AWS.

Record the four ARNs — you'll paste them into the task definition in §14:

```bash
for name in jwt-secret encryption-key postgres-password worker-token; do
  aws secretsmanager describe-secret --secret-id /activepieces/prod/$name \
    --region eu-central-1 --query 'ARN' --output text
done
```

## 9. IAM roles

Three roles:

| Role | Purpose |
| --- | --- |
| `ecsTaskExecutionRole` | ECS agent uses this to pull the ECR image, read secrets, and write logs. One shared role across APP + WORKER. |
| `activepieces-prod-app-task` | Runtime role for the **APP** task: read Secrets Manager, read/write the S3 bucket. |
| `activepieces-prod-worker-task` | Runtime role for the **WORKER** task: read Secrets Manager (worker token only). No DB, no S3 direct — workers proxy through the APP. |

### 9.1 Execution role

Likely already exists in the account (name: `ecsTaskExecutionRole`). If not:

```bash
aws iam create-role --role-name ecsTaskExecutionRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
aws iam attach-role-policy --role-name ecsTaskExecutionRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonECSTaskExecutionRolePolicy

# Allow it to resolve the Secrets Manager ARNs this project uses:
aws iam put-role-policy --role-name ecsTaskExecutionRole \
  --policy-name activepieces-prod-secrets-read \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":["secretsmanager:GetSecretValue"],
      "Resource":"arn:aws:secretsmanager:eu-central-1:557690597652:secret:/activepieces/prod/*"
    }]
  }'
```

### 9.2 APP task role

```bash
aws iam create-role --role-name activepieces-prod-app-task \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'

aws iam put-role-policy --role-name activepieces-prod-app-task \
  --policy-name s3-files \
  --policy-document '{
    "Version":"2012-10-17",
    "Statement":[{
      "Effect":"Allow",
      "Action":["s3:GetObject","s3:PutObject","s3:DeleteObject","s3:ListBucket"],
      "Resource":[
        "arn:aws:s3:::connectors-activepieces-prod-files",
        "arn:aws:s3:::connectors-activepieces-prod-files/*"
      ]
    }]
  }'
```

### 9.3 WORKER task role

Empty runtime permissions — workers only need what the execution role provides (image pull, secret resolution, log write).

```bash
aws iam create-role --role-name activepieces-prod-worker-task \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ecs-tasks.amazonaws.com"},"Action":"sts:AssumeRole"}]}'
```

## 10. CloudWatch log groups

One per service so logs are greppable by task type.

```bash
aws logs create-log-group --log-group-name /ecs/activepieces-prod-app --region eu-central-1
aws logs create-log-group --log-group-name /ecs/activepieces-prod-worker --region eu-central-1

aws logs put-retention-policy --log-group-name /ecs/activepieces-prod-app \
  --retention-in-days 30 --region eu-central-1
aws logs put-retention-policy --log-group-name /ecs/activepieces-prod-worker \
  --retention-in-days 30 --region eu-central-1
```

## 11. EC2 capacity provider

The cluster may already have EC2 capacity, but this section documents the full setup in case you need to expand or rebuild it.

### 11.1 Launch template

```bash
LATEST_AMI=$(aws ssm get-parameter --region eu-central-1 \
  --name /aws/service/ecs/optimized-ami/amazon-linux-2/recommended/image_id \
  --query 'Parameter.Value' --output text)

# Instance profile — allows the ECS agent on the box to register with the cluster
aws iam create-role --role-name ecsInstanceRole \
  --assume-role-policy-document '{"Version":"2012-10-17","Statement":[{"Effect":"Allow","Principal":{"Service":"ec2.amazonaws.com"},"Action":"sts:AssumeRole"}]}' \
  2>/dev/null || true
aws iam attach-role-policy --role-name ecsInstanceRole \
  --policy-arn arn:aws:iam::aws:policy/service-role/AmazonEC2ContainerServiceforEC2Role
aws iam create-instance-profile --instance-profile-name ecsInstanceRole 2>/dev/null || true
aws iam add-role-to-instance-profile --instance-profile-name ecsInstanceRole \
  --role-name ecsInstanceRole 2>/dev/null || true

# Launch template
aws ec2 create-launch-template \
  --launch-template-name activepieces-prod-lt \
  --region eu-central-1 \
  --launch-template-data '{
    "ImageId": "'$LATEST_AMI'",
    "InstanceType": "t3.medium",
    "IamInstanceProfile": {"Name": "ecsInstanceRole"},
    "SecurityGroupIds": ["'$TASKS_SG'"],
    "UserData": "'$(echo -n "#!/bin/bash
echo 'ECS_CLUSTER=ConnectorsAppCluster' >> /etc/ecs/ecs.config
echo 'ECS_ENABLE_CONTAINER_METADATA=true' >> /etc/ecs/ecs.config" | base64)'",
    "BlockDeviceMappings": [{
      "DeviceName": "/dev/xvda",
      "Ebs": {"VolumeSize": 30, "VolumeType": "gp3", "Encrypted": true}
    }]
  }'
```

### 11.2 Auto Scaling Group

```bash
aws autoscaling create-auto-scaling-group \
  --auto-scaling-group-name activepieces-prod-asg \
  --launch-template LaunchTemplateName=activepieces-prod-lt,Version='$Latest' \
  --min-size 2 --max-size 6 --desired-capacity 2 \
  --vpc-zone-identifier $PRIVATE_SUBNETS \
  --region eu-central-1 \
  --tags Key=Name,Value=activepieces-prod-ecs,PropagateAtLaunch=true \
         Key=AmazonECSManaged,Value=true,PropagateAtLaunch=true
```

### 11.3 Capacity provider

```bash
aws ecs create-capacity-provider \
  --name activepieces-prod-cp \
  --auto-scaling-group-provider \
    "autoScalingGroupArn=$(aws autoscaling describe-auto-scaling-groups \
      --auto-scaling-group-names activepieces-prod-asg \
      --region eu-central-1 \
      --query 'AutoScalingGroups[0].AutoScalingGroupARN' --output text),\
managedScaling={status=ENABLED,targetCapacity=75,minimumScalingStepSize=1,maximumScalingStepSize=2},\
managedTerminationProtection=ENABLED" \
  --region eu-central-1

aws ecs put-cluster-capacity-providers \
  --cluster ConnectorsAppCluster \
  --capacity-providers activepieces-prod-cp \
  --default-capacity-provider-strategy capacityProvider=activepieces-prod-cp,weight=1,base=1 \
  --region eu-central-1
```

Sanity check: `aws ecs describe-clusters --clusters ConnectorsAppCluster --region eu-central-1` should show `registeredContainerInstancesCount >= 2` within ~3 minutes.

## 12. ALB + target group + listener

```bash
ALB_ARN=$(aws elbv2 create-load-balancer \
  --name activepieces-prod-alb \
  --subnets $(echo $PUBLIC_SUBNETS | tr ',' ' ') \
  --security-groups $ALB_SG \
  --scheme internet-facing --type application \
  --region eu-central-1 \
  --query 'LoadBalancers[0].LoadBalancerArn' --output text)

TG_ARN=$(aws elbv2 create-target-group \
  --name activepieces-prod-app-tg \
  --protocol HTTP --port 80 \
  --vpc-id $VPC_ID \
  --target-type instance \
  --health-check-path /api/v1/flags \
  --health-check-interval-seconds 30 \
  --healthy-threshold-count 2 --unhealthy-threshold-count 3 \
  --region eu-central-1 \
  --query 'TargetGroups[0].TargetGroupArn' --output text)

# HTTPS listener with the ACM cert
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTPS --port 443 \
  --certificates CertificateArn=$CERT_ARN \
  --default-actions Type=forward,TargetGroupArn=$TG_ARN \
  --region eu-central-1

# HTTP→HTTPS redirect on :80 (handy for manual curls; Cloudflare will already force HTTPS)
aws elbv2 create-listener \
  --load-balancer-arn $ALB_ARN \
  --protocol HTTP --port 80 \
  --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}' \
  --region eu-central-1

# Capture the ALB DNS name — you paste this into Cloudflare
aws elbv2 describe-load-balancers --load-balancer-arns $ALB_ARN \
  --region eu-central-1 \
  --query 'LoadBalancers[0].DNSName' --output text
```

`/api/v1/flags` is a public, unauthenticated endpoint that's cheap to call and returns quickly — a good health check target.

### 12.1 Cloudflare CNAME

In the Cloudflare dashboard:

1. DNS → Records → **Add record**.
2. Type: `CNAME`, Name: `<subdomain matching PROD_DOMAIN>`, Target: the ALB DNS name from above.
3. **Proxy status: Proxied (orange cloud)** — Cloudflare terminates the client connection, inspects, and forwards to the ALB over HTTPS. The ACM cert on the ALB handles the Cloudflare ↔ ALB leg.
4. TTL: Auto.

In **SSL/TLS → Overview**, set encryption mode to **Full (strict)** — Cloudflare will verify the ALB's ACM cert. "Flexible" would terminate TLS at Cloudflare and talk to the ALB over HTTP, which bypasses the whole point.

Test once DNS propagates:

```bash
curl -sS -o /dev/null -w "%{http_code}\n" https://<PROD_DOMAIN>/api/v1/flags
# expect 200 (even with no tasks yet, the ALB returns 503; 200 confirms a task is registered)
```

## 13. Worker token generation

Workers authenticate to the APP using a JWT signed with `AP_JWT_SECRET`. Generate it locally against the repo — no ECS round-trip needed.

```bash
# On your laptop, in the repo root
npm install   # once
npm run workers token
# Prompt: paste the AP_JWT_SECRET from Secrets Manager:
aws secretsmanager get-secret-value \
  --secret-id /activepieces/prod/jwt-secret \
  --region eu-central-1 \
  --query SecretString --output text
# Copy the printed token and overwrite the placeholder secret:
WORKER_TOKEN=<paste>
aws secretsmanager put-secret-value \
  --secret-id /activepieces/prod/worker-token \
  --secret-string "$WORKER_TOKEN" \
  --region eu-central-1
```

The token is long-lived (signed, no expiry) — if `AP_JWT_SECRET` rotates, regenerate the worker token and deploy the worker service.

## 14. Task definitions

One task per service. Both use the same image; `AP_CONTAINER_TYPE` determines behavior.

### 14.1 APP task definition

Save as `task-def-app.json` locally:

```json
{
  "family": "activepieces-prod-app",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::557690597652:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::557690597652:role/activepieces-prod-app-task",
  "containerDefinitions": [
    {
      "name": "app",
      "image": "557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces:main-latest",
      "essential": true,
      "portMappings": [{"containerPort": 80, "hostPort": 0, "protocol": "tcp"}],
      "environment": [
        {"name": "AP_CONTAINER_TYPE", "value": "APP"},
        {"name": "AP_ENVIRONMENT", "value": "prod"},
        {"name": "AP_FRONTEND_URL", "value": "https://<PROD_DOMAIN>"},
        {"name": "AP_EXECUTION_MODE", "value": "SANDBOX_CODE_AND_PROCESS"},
        {"name": "AP_ENGINE_EXECUTABLE_PATH", "value": "dist/packages/engine/main.js"},
        {"name": "AP_TELEMETRY_ENABLED", "value": "false"},
        {"name": "AP_QUEUE_UI_ENABLED", "value": "false"},
        {"name": "AP_DB_TYPE", "value": "POSTGRES"},
        {"name": "AP_POSTGRES_HOST", "value": "<DB_HOST>"},
        {"name": "AP_POSTGRES_PORT", "value": "5432"},
        {"name": "AP_POSTGRES_DATABASE", "value": "activepieces"},
        {"name": "AP_POSTGRES_USERNAME", "value": "activepieces"},
        {"name": "AP_POSTGRES_USE_SSL", "value": "true"},
        {"name": "AP_REDIS_TYPE", "value": "STANDALONE"},
        {"name": "AP_REDIS_HOST", "value": "<REDIS_HOST>"},
        {"name": "AP_REDIS_PORT", "value": "6379"},
        {"name": "AP_FILE_STORAGE_LOCATION", "value": "S3"},
        {"name": "AP_S3_BUCKET", "value": "connectors-activepieces-prod-files"},
        {"name": "AP_S3_REGION", "value": "eu-central-1"},
        {"name": "AP_S3_USE_IRSA", "value": "false"},
        {"name": "AP_S3_USE_SIGNED_URLS", "value": "true"}
      ],
      "secrets": [
        {"name": "AP_JWT_SECRET",         "valueFrom": "arn:aws:secretsmanager:eu-central-1:557690597652:secret:/activepieces/prod/jwt-secret"},
        {"name": "AP_ENCRYPTION_KEY",     "valueFrom": "arn:aws:secretsmanager:eu-central-1:557690597652:secret:/activepieces/prod/encryption-key"},
        {"name": "AP_POSTGRES_PASSWORD",  "valueFrom": "arn:aws:secretsmanager:eu-central-1:557690597652:secret:/activepieces/prod/postgres-password"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/activepieces-prod-app",
          "awslogs-region": "eu-central-1",
          "awslogs-stream-prefix": "app"
        }
      },
      "stopTimeout": 60,
      "ulimits": [{"name": "nofile", "softLimit": 65535, "hardLimit": 65535}]
    }
  ]
}
```

Replace `<DB_HOST>` and `<REDIS_HOST>` with the values captured in §5.2 / §6. Note: `AP_S3_ACCESS_KEY_ID` / `AP_S3_SECRET_ACCESS_KEY` are **absent** — we're using the task IAM role (`activepieces-prod-app-task`) for S3 auth.

Register it:

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-def-app.json --region eu-central-1
```

### 14.2 WORKER task definition

Save as `task-def-worker.json`. The worker talks outward to the APP URL and needs neither DB nor Redis env vars.

```json
{
  "family": "activepieces-prod-worker",
  "networkMode": "bridge",
  "requiresCompatibilities": ["EC2"],
  "cpu": "1024",
  "memory": "2048",
  "executionRoleArn": "arn:aws:iam::557690597652:role/ecsTaskExecutionRole",
  "taskRoleArn": "arn:aws:iam::557690597652:role/activepieces-prod-worker-task",
  "containerDefinitions": [
    {
      "name": "worker",
      "image": "557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces:main-latest",
      "essential": true,
      "environment": [
        {"name": "AP_CONTAINER_TYPE", "value": "WORKER"},
        {"name": "AP_FRONTEND_URL", "value": "https://<PROD_DOMAIN>"},
        {"name": "AP_EXECUTION_MODE", "value": "SANDBOX_CODE_AND_PROCESS"},
        {"name": "AP_ENGINE_EXECUTABLE_PATH", "value": "dist/packages/engine/main.js"},
        {"name": "AP_WORKER_CONCURRENCY", "value": "10"}
      ],
      "secrets": [
        {"name": "AP_WORKER_TOKEN", "valueFrom": "arn:aws:secretsmanager:eu-central-1:557690597652:secret:/activepieces/prod/worker-token"}
      ],
      "logConfiguration": {
        "logDriver": "awslogs",
        "options": {
          "awslogs-group": "/ecs/activepieces-prod-worker",
          "awslogs-region": "eu-central-1",
          "awslogs-stream-prefix": "worker"
        }
      },
      "stopTimeout": 60,
      "mountPoints": [{"sourceVolume": "cache", "containerPath": "/usr/src/app/cache"}]
    }
  ],
  "volumes": [{"name": "cache", "host": {}}]
}
```

The `cache` host volume gives each worker a persistent piece cache per EC2 instance — cold cache is very slow, so this matters. It's host-scoped, so a task moving between EC2 instances starts cold.

```bash
aws ecs register-task-definition \
  --cli-input-json file://task-def-worker.json --region eu-central-1
```

## 15. ECS services

### 15.1 APP service

```bash
APP_SERVICE_NAME="ConnectorsECSServices3-ActivePieces-$(openssl rand -base64 9 | tr -dc 'A-Za-z0-9' | head -c 12)"
echo "APP service: $APP_SERVICE_NAME"

aws ecs create-service \
  --cluster ConnectorsAppCluster \
  --service-name "$APP_SERVICE_NAME" \
  --task-definition activepieces-prod-app \
  --desired-count 1 \
  --launch-type EC2 \
  --load-balancers "targetGroupArn=$TG_ARN,containerName=app,containerPort=80" \
  --health-check-grace-period-seconds 180 \
  --deployment-configuration 'maximumPercent=200,minimumHealthyPercent=100,deploymentCircuitBreaker={enable=true,rollback=true}' \
  --region eu-central-1
```

> **Why `minimumHealthyPercent=100` with one task?** With a single task, `100%` means ECS starts the new task first, waits until it's healthy on the ALB, then stops the old one — no downtime window. Requires `maximumPercent=200` so a second task can boot before the first stops. Scale `desired-count` to 2+ to tolerate instance failure.

### 15.2 WORKER service

```bash
WORKER_SERVICE_NAME="ConnectorsECSServices3-ActivePiecesWorker-$(openssl rand -base64 9 | tr -dc 'A-Za-z0-9' | head -c 12)"
echo "WORKER service: $WORKER_SERVICE_NAME"

aws ecs create-service \
  --cluster ConnectorsAppCluster \
  --service-name "$WORKER_SERVICE_NAME" \
  --task-definition activepieces-prod-worker \
  --desired-count 2 \
  --launch-type EC2 \
  --deployment-configuration 'maximumPercent=200,minimumHealthyPercent=50,deploymentCircuitBreaker={enable=true,rollback=true}' \
  --region eu-central-1
```

Workers don't register with the ALB. `minimumHealthyPercent=50` lets ECS restart them in-place without doubling capacity.

**Save both service names** — they go into GitHub Secrets for the CI workflow (§17).

## 16. Initial manual deploy & verification

The task definition references `:main-latest` but nothing has pushed to that tag yet. Build and push once from your laptop before wiring CI:

```bash
# From the repo root on your Mac (remember: Apple Silicon → x86 EC2)
aws ecr get-login-password --region eu-central-1 \
  | docker login --username AWS \
    --password-stdin 557690597652.dkr.ecr.eu-central-1.amazonaws.com

docker buildx create --name mycarely --use --bootstrap 2>/dev/null || true
docker buildx build \
  --platform linux/amd64 \
  --push \
  -t 557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces:main-latest \
  -t 557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces:bootstrap-$(git rev-parse --short HEAD) \
  .
```

Force the ECS services to pick up the newly-available image:

```bash
aws ecs update-service --cluster ConnectorsAppCluster \
  --service "$APP_SERVICE_NAME" --force-new-deployment --region eu-central-1
aws ecs wait services-stable --cluster ConnectorsAppCluster \
  --services "$APP_SERVICE_NAME" --region eu-central-1

# APP must land first so migrations complete before workers run
aws ecs update-service --cluster ConnectorsAppCluster \
  --service "$WORKER_SERVICE_NAME" --force-new-deployment --region eu-central-1
aws ecs wait services-stable --cluster ConnectorsAppCluster \
  --services "$WORKER_SERVICE_NAME" --region eu-central-1
```

Verify:

```bash
# 200 expected once a task is healthy on the ALB
curl -sS -o /dev/null -w "HTTP %{http_code}\n" https://<PROD_DOMAIN>/api/v1/flags

# Logs
aws logs tail /ecs/activepieces-prod-app --follow --region eu-central-1
aws logs tail /ecs/activepieces-prod-worker --follow --region eu-central-1
```

Open `https://<PROD_DOMAIN>` in a browser and sign up as the first admin.

Inside the app: **Platform Admin → Infra → Workers** — expect 2 workers online (matches `desired-count=2`). If this is empty, workers failed to authenticate — check `AP_WORKER_TOKEN` against the current `AP_JWT_SECRET`.

## 17. GitHub Actions CI/CD

### 17.1 AWS IAM user for CI

Create an IAM user `gh-actions-ecs-prod` with this inline policy — scoped to **only the prod repo and the two prod services**, so a compromised key can't push to dev or touch other services on the cluster.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Sid": "EcrAuth",
      "Effect": "Allow",
      "Action": "ecr:GetAuthorizationToken",
      "Resource": "*"
    },
    {
      "Sid": "EcrPushProdOnly",
      "Effect": "Allow",
      "Action": [
        "ecr:BatchCheckLayerAvailability",
        "ecr:PutImage",
        "ecr:InitiateLayerUpload",
        "ecr:UploadLayerPart",
        "ecr:CompleteLayerUpload"
      ],
      "Resource": "arn:aws:ecr:eu-central-1:557690597652:repository/connectors/activepieces"
    },
    {
      "Sid": "EcsDescribeAll",
      "Effect": "Allow",
      "Action": [
        "ecs:DescribeTaskDefinition",
        "ecs:RegisterTaskDefinition"
      ],
      "Resource": "*"
    },
    {
      "Sid": "EcsUpdateProdServicesOnly",
      "Effect": "Allow",
      "Action": [
        "ecs:UpdateService",
        "ecs:DescribeServices"
      ],
      "Resource": [
        "arn:aws:ecs:eu-central-1:557690597652:service/ConnectorsAppCluster/ConnectorsECSServices3-ActivePieces-*",
        "arn:aws:ecs:eu-central-1:557690597652:service/ConnectorsAppCluster/ConnectorsECSServices3-ActivePiecesWorker-*"
      ]
    },
    {
      "Sid": "PassRolesNeededByTaskDef",
      "Effect": "Allow",
      "Action": "iam:PassRole",
      "Resource": [
        "arn:aws:iam::557690597652:role/ecsTaskExecutionRole",
        "arn:aws:iam::557690597652:role/activepieces-prod-app-task",
        "arn:aws:iam::557690597652:role/activepieces-prod-worker-task"
      ]
    }
  ]
}
```

Add the access key + secret to GitHub repo **Settings → Secrets and variables → Actions**:

| Secret name                  | Value                                       |
| ---------------------------- | ------------------------------------------- |
| `PROD_AWS_ACCESS_KEY_ID`     | from IAM user                               |
| `PROD_AWS_SECRET_ACCESS_KEY` | from IAM user                               |
| `PROD_ECS_SERVICE_APP`       | `ConnectorsECSServices3-ActivePieces-<RAND>` |
| `PROD_ECS_SERVICE_WORKER`    | `ConnectorsECSServices3-ActivePiecesWorker-<RAND>` |

Non-secret values are hardcoded in the workflow.

> **Hardening follow-up:** migrate to GitHub OIDC federation — same policy, no long-lived access key. Set it up once the workflow is stable.

### 17.2 The workflow

Create `.github/workflows/deploy-prod.yml`:

```yaml
name: Deploy to prod ECS

on:
  pull_request:
    types: [closed]
    branches: [main]

permissions:
  contents: read

env:
  AWS_REGION: eu-central-1
  ECR_REGISTRY: 557690597652.dkr.ecr.eu-central-1.amazonaws.com
  ECR_REPOSITORY: connectors/activepieces
  ECS_CLUSTER: ConnectorsAppCluster
  TASK_DEF_APP_FAMILY: activepieces-prod-app
  TASK_DEF_WORKER_FAMILY: activepieces-prod-worker
  APP_CONTAINER_NAME: app
  WORKER_CONTAINER_NAME: worker

jobs:
  build:
    # Skip unmerged PR closes — safer and saves ~8 min of CI per wasted run.
    if: github.event.pull_request.merged == true
    runs-on: ubuntu-latest
    outputs:
      image_app: ${{ steps.tags.outputs.image_app }}
      image_worker: ${{ steps.tags.outputs.image_worker }}
    steps:
      - uses: actions/checkout@v4

      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - uses: aws-actions/amazon-ecr-login@v2
      - uses: docker/setup-buildx-action@v3

      - name: Compute tags
        id: tags
        run: |
          IMG=${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:main-${{ github.sha }}
          echo "image_app=$IMG" >> $GITHUB_OUTPUT
          echo "image_worker=$IMG" >> $GITHUB_OUTPUT

      - name: Build & push
        uses: docker/build-push-action@v5
        with:
          context: .
          push: true
          platforms: linux/amd64
          tags: |
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:main-latest
            ${{ env.ECR_REGISTRY }}/${{ env.ECR_REPOSITORY }}:main-${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy-app:
    needs: build
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Fetch current APP task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.TASK_DEF_APP_FAMILY }} \
            --query taskDefinition > task-def-app.json

      - name: Render new image into task def
        id: render-app
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def-app.json
          container-name: ${{ env.APP_CONTAINER_NAME }}
          image: ${{ needs.build.outputs.image_app }}

      - name: Deploy APP
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render-app.outputs.task-definition }}
          service: ${{ secrets.PROD_ECS_SERVICE_APP }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 15

  deploy-worker:
    needs: deploy-app   # migrations must complete on APP before WORKER picks up new code
    runs-on: ubuntu-latest
    steps:
      - uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.PROD_AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.PROD_AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Fetch current WORKER task definition
        run: |
          aws ecs describe-task-definition \
            --task-definition ${{ env.TASK_DEF_WORKER_FAMILY }} \
            --query taskDefinition > task-def-worker.json

      - name: Render new image into task def
        id: render-worker
        uses: aws-actions/amazon-ecs-render-task-definition@v1
        with:
          task-definition: task-def-worker.json
          container-name: ${{ env.WORKER_CONTAINER_NAME }}
          image: ${{ needs.build.outputs.image_worker }}

      - name: Deploy WORKER
        uses: aws-actions/amazon-ecs-deploy-task-definition@v1
        with:
          task-definition: ${{ steps.render-worker.outputs.task-definition }}
          service: ${{ secrets.PROD_ECS_SERVICE_WORKER }}
          cluster: ${{ env.ECS_CLUSTER }}
          wait-for-service-stability: true
          wait-for-minutes: 10
```

Why these choices:

- **`if: github.event.pull_request.merged == true`** — unmerged PR closes get the event but skip the build. Mirrors the "merged only" trigger semantic without needing a second job-level event filter.
- **APP before WORKER** — the `deploy-worker` job depends on `deploy-app` completing (and therefore migrations being applied). If APP fails, workers are never updated against a schema they may not match.
- **Deployment circuit breaker is on (configured on the service in §15)** — if a task fails to reach healthy within `wait-for-minutes`, ECS rolls back automatically.
- **`main-<sha>` immutable tag** — the task def is pinned to the SHA; `main-latest` is for humans and local debugging, not for what ECS actually pulls.

## 18. First automated deploy checklist

1. Commit the workflow file on `dev` branch (assuming standard git flow), open a PR to `main`, merge.
2. Watch **Actions → Deploy to prod ECS**: `build` → `deploy-app` → `deploy-worker`. Expected total ~10–15 min first run (cache-miss), ~5–8 min after.
3. In AWS ECS console: both services show a new deployment revision `(PRIMARY)`, previous `(ACTIVE)` drains out.
4. `curl https://<PROD_DOMAIN>/api/v1/flags` returns 200 throughout.
5. APP logs show `running migration...` entries (new migrations only; no-op if the branch didn't add any).
6. Workers show online count unchanged in Platform Admin Console.

If any step fails, see §20 Troubleshooting before rolling back.

## 19. Rollback

The deployment circuit breaker handles most bad deploys automatically — ECS stops the rollout and reverts to the prior revision if the new tasks don't stabilize within the wait window.

**Manual rollback** (when a bad deploy stabilized but behaves wrong in production):

```bash
# List revisions for the APP task def
aws ecs list-task-definitions \
  --family-prefix activepieces-prod-app \
  --sort DESC --max-items 10 --region eu-central-1

# Roll APP back one revision (repeat for worker)
aws ecs update-service \
  --cluster ConnectorsAppCluster \
  --service "$APP_SERVICE_NAME" \
  --task-definition activepieces-prod-app:<prev-revision-number> \
  --region eu-central-1
```

> **Migrations are not rolled back.** If the rollback crosses a migration, you keep the new schema but run the old code. Usually fine because migrations are backward-compatible for one release; verify before rolling further than that.

## 20. Troubleshooting

| Symptom | Likely cause | Fix |
| --- | --- | --- |
| New tasks crash-loop with `exec format error` | Wrong arch pushed (arm64 image on x86 host) | CI uses `platforms: linux/amd64`, so this should only happen after a manual laptop push. Rebuild with `docker buildx build --platform linux/amd64 --push`. |
| Tasks stuck in `PENDING`, cluster has no capacity | ASG at `max-size`, or AMI not registering with the cluster | `aws ecs describe-clusters` for registered instance count. Bump `--max-size` on ASG, or check the instance's `/var/log/ecs/ecs-agent.log` for join failures. |
| APP tasks start but ALB target health is `unhealthy` | Security group gap, or the container's port 0 (dynamic) didn't propagate | Confirm TASKS_SG allows ingress from ALB_SG on TCP 80. Confirm `networkMode: bridge` + `hostPort: 0` so ECS assigns a dynamic port and registers it in the target group. |
| APP logs: `error: password authentication failed for user "activepieces"` | `AP_POSTGRES_PASSWORD` secret out of sync with RDS master password | Re-set the master password via `aws rds modify-db-instance --master-user-password`, then update the Secrets Manager secret to match, then force new deployment. |
| Worker logs: `401 Unauthorized` posting to `/api/v1/workers/...` | `AP_WORKER_TOKEN` was signed against a different `AP_JWT_SECRET` | Regenerate per §13. |
| Migration fails mid-rollout, circuit breaker rolls back, `ACTIVE` is old revision | Schema change is not safe under concurrent writes (e.g. `NOT NULL` add on a large table) | Fix the migration to be two-phase (add nullable → backfill → add constraint), deploy each phase as a separate PR. |
| 502 from ALB after Cloudflare update | Cloudflare SSL mode is "Flexible" instead of "Full (strict)" | Switch to Full (strict) — see §12.1. |
| CI `deploy-app` times out on `wait-for-service-stability` | New task definition has a config error; tasks keep restarting | Check `/ecs/activepieces-prod-app` logs. Common: malformed env value, missing secret ARN, or task CPU/memory below what the container needs. |
| `ecs:UpdateService` returns `AccessDenied` in CI | IAM policy resource ARN doesn't match the service name (the `<RAND>` portion) | The policy uses `service/ConnectorsAppCluster/ConnectorsECSServices3-ActivePieces-*` — verify the actual service name matches that prefix. |
| Cloudflare returns 526 (Invalid SSL certificate) | ACM cert expired/rotated but ALB listener still uses the old ARN | `aws elbv2 describe-listener-certificates` to confirm. Re-issue cert + update listener. |

## 21. Operations & monitoring

- **Logs**: `aws logs tail /ecs/activepieces-prod-app --follow` and `/ecs/activepieces-prod-worker --follow`. 30-day retention per §10.
- **Metrics**: CloudWatch container insights on `ConnectorsAppCluster` — enable via `aws ecs update-cluster-settings --cluster ConnectorsAppCluster --settings name=containerInsights,value=enabled`.
- **DB**: Enhanced monitoring + Performance Insights on RDS — `aws rds modify-db-instance --enable-performance-insights ...`.
- **Redis**: queue depth via `AP_QUEUE_UI_ENABLED=true` (gated behind basic auth via `AP_QUEUE_UI_USERNAME`/`_PASSWORD` — only enable if fronted by auth).
- **Image retention**: ECR keeps all pushed images indefinitely. Configure a lifecycle policy to expire untagged and old `main-<sha>` images:

  ```bash
  aws ecr put-lifecycle-policy \
    --repository-name connectors/activepieces \
    --region eu-central-1 \
    --lifecycle-policy-text '{"rules":[
      {"rulePriority":1,"description":"Expire untagged >7 days","selection":{"tagStatus":"untagged","countType":"sinceImagePushed","countUnit":"days","countNumber":7},"action":{"type":"expire"}},
      {"rulePriority":2,"description":"Keep last 30 main-* images","selection":{"tagStatus":"tagged","tagPrefixList":["main-"],"countType":"imageCountMoreThan","countNumber":30},"action":{"type":"expire"}}
    ]}'
  ```

## 22. Scaling further

Knobs to reach for, in order, as load grows:

1. **Worker count** — bump `--desired-count` on the worker service (up to the ASG `max-size`). Each worker runs `AP_WORKER_CONCURRENCY=10` flows in parallel.
2. **Worker concurrency** — raise `AP_WORKER_CONCURRENCY` in `task-def-worker.json`. Each concurrent run is one engine process + sandbox; 10 fits comfortably in 2 GB RAM, 20 needs ~4 GB.
3. **APP count + ALB** — scale APP to 2+ for HA; the ALB already round-robins.
4. **RDS** — vertical scale to `db.t4g.medium` / `db.m6g.*`; enable Multi-AZ when SLAs require synchronous failover.
5. **ElastiCache** — add a replica and enable failover; raise node size if BullMQ memory pressure shows up.
6. **File storage** — already on S3; if egress cost becomes a problem, flip `AP_S3_USE_SIGNED_URLS=false` to proxy through the APP, trading $ for bandwidth consolidation.
7. **Capacity provider** — raise ASG `max-size` to match peak concurrent tasks + headroom.

## 23. What lives where (reference index)

| Resource | Name / ID |
| --- | --- |
| ECS cluster | `ConnectorsAppCluster` |
| ECS service (APP) | `ConnectorsECSServices3-ActivePieces-<RAND>` |
| ECS service (WORKER) | `ConnectorsECSServices3-ActivePiecesWorker-<RAND>` |
| Task def families | `activepieces-prod-app`, `activepieces-prod-worker` |
| ECR repo | `connectors/activepieces` |
| RDS instance | `activepieces-prod-db` |
| ElastiCache cluster | `activepieces-prod-redis` |
| S3 bucket | `connectors-activepieces-prod-files` |
| ALB | `activepieces-prod-alb` |
| Target group | `activepieces-prod-app-tg` |
| Capacity provider | `activepieces-prod-cp` |
| ASG | `activepieces-prod-asg` |
| Launch template | `activepieces-prod-lt` |
| Security groups | `activepieces-prod-{alb,tasks,rds,redis}` |
| IAM roles | `ecsTaskExecutionRole`, `activepieces-prod-app-task`, `activepieces-prod-worker-task`, `ecsInstanceRole` |
| Secrets | `/activepieces/prod/{jwt-secret,encryption-key,postgres-password,worker-token}` |
| Log groups | `/ecs/activepieces-prod-{app,worker}` |
| CI workflow | `.github/workflows/deploy-prod.yml` |
| GitHub secrets | `PROD_AWS_ACCESS_KEY_ID`, `PROD_AWS_SECRET_ACCESS_KEY`, `PROD_ECS_SERVICE_APP`, `PROD_ECS_SERVICE_WORKER` |
