# Activepieces Prod — CloudFormation Templates

Seven numbered templates bring the prod stack online from scratch. They use CloudFormation cross-stack exports to thread IDs/ARNs between themselves — the number prefix **is** the deploy order.

The narrative guide (`../bootstrap.md`) documents what these templates do, why, and how they fit together with CI/CD. This README is the runbook for applying them.

## Assumptions (already in place)

| Resource | Identifier |
| --- | --- |
| Account / region | `557690597652` / `eu-central-1` |
| ECS cluster | `ConnectorsAppCluster` |
| ECR repo | `connectors/activepieces` (URI `557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces`) |
| VPC | The one hosting `ConnectorsAppCluster`. Discover with the snippet in `../bootstrap.md` §3.1. |

## What is intentionally NOT in these templates

- **ALB HTTPS listener + HTTP→HTTPS redirect** — operator creates manually on the ALB from stack 5, with the ACM certificate ARN.
- **ACM certificate** — operator issues separately; DNS validation CNAME goes into Cloudflare (not Route 53).
- **Cloudflare DNS record** — operator pastes the ALB DNS name into Cloudflare as a proxied CNAME.
- **Cluster-level capacity-provider association** — stack 4 creates the capacity provider but does not attach it to the cluster's default strategy, so the existing `ConnectorsECSServices2-FlerpService` default strategy isn't overwritten.
- **First ECR image push** — operator pushes `main-latest` manually once before stack 6 registers the task def against it; CI takes over afterwards.
- **Worker token value** — generated locally via `npm run workers token` after the APP service is healthy, then written to the pre-created secret with `aws secretsmanager put-secret-value`.

## Deploy order

Each `aws cloudformation deploy` call below assumes `AWS_REGION=eu-central-1` in your shell environment (or add `--region eu-central-1`). Replace placeholder values before running.

### 0 — one-time per account: discover VPC + subnets

```bash
export AWS_REGION=eu-central-1

INSTANCE_ARN=$(aws ecs list-container-instances --cluster ConnectorsAppCluster \
  --query 'containerInstanceArns[0]' --output text)
EC2_ID=$(aws ecs describe-container-instances --cluster ConnectorsAppCluster \
  --container-instances "$INSTANCE_ARN" \
  --query 'containerInstances[0].ec2InstanceId' --output text)

aws ec2 describe-instances --instance-ids "$EC2_ID" \
  --query 'Reservations[0].Instances[0].{VpcId:VpcId,Subnets:NetworkInterfaces[*].SubnetId}'
```

Record the VPC and classify subnets as public/private (route table with default `igw-*` = public; default via NAT gateway = private).

### 1 — foundation (security groups, IAM, secrets, S3, log groups)

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-foundation \
  --template-file 1_foundation.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides VpcId=<VPC_ID>
```

**Post-1 manual steps (do before stack 6):**

```bash
# Encryption key MUST be exactly 32 hex chars. CFN's GenerateSecretString
# cannot constrain to hex, so overwrite the CFN-generated seed value:
aws secretsmanager put-secret-value \
  --secret-id /activepieces/prod/encryption-key \
  --secret-string "$(openssl rand -hex 16)"

# JWT secret: any sufficiently-long random string works; the CFN-generated
# value is fine. Only overwrite if you want hex specifically:
# aws secretsmanager put-secret-value \
#   --secret-id /activepieces/prod/jwt-secret \
#   --secret-string "$(openssl rand -hex 32)"
```

> **Back up the encryption key offline** (password manager, hardware token) before continuing. Losing it bricks every stored OAuth token, API key, and webhook secret in the DB — irreversibly.

### 2 — database (RDS Postgres)

Slow: RDS create takes ~10 min. Kick this off in parallel with stack 3.

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-database \
  --template-file 2_database.yaml \
  --parameter-overrides \
    PrivateSubnetIds=<subnet-a>\\,<subnet-b>
```

> Note the escaped comma in `PrivateSubnetIds`. CloudFormation `deploy` splits parameters by `,` at the shell level; escape or single-quote to preserve the list.

### 3 — cache (ElastiCache Redis)

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-cache \
  --template-file 3_cache.yaml \
  --parameter-overrides \
    PrivateSubnetIds=<subnet-a>\\,<subnet-b>
```

### 4 — EC2 capacity (launch template + ASG + capacity provider)

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-ec2-capacity \
  --template-file 4_ec2_capacity.yaml \
  --capabilities CAPABILITY_NAMED_IAM \
  --parameter-overrides \
    PrivateSubnetIds=<subnet-a>\\,<subnet-b>
```

Verify instances joined the cluster (~3 min):

```bash
aws ecs describe-clusters --clusters ConnectorsAppCluster \
  --query 'clusters[0].registeredContainerInstancesCount'
# expect a number >= DesiredCapacity (default 2); accumulates as other services' instances
```

**Optional post-4:** attach the new capacity provider to the cluster's default strategy **alongside any existing providers**. `put-cluster-capacity-providers` replaces the whole list, so list every provider already attached:

```bash
aws ecs describe-clusters --clusters ConnectorsAppCluster \
  --query 'clusters[0].{current:capacityProviders,strategy:defaultCapacityProviderStrategy}'

# Example — append our new one to whatever was already there:
aws ecs put-cluster-capacity-providers \
  --cluster ConnectorsAppCluster \
  --capacity-providers <existing-1> <existing-2> activepieces-prod-cp \
  --default-capacity-provider-strategy \
    capacityProvider=<existing-default>,weight=1 \
    capacityProvider=activepieces-prod-cp,weight=0,base=0
```

Stack 7's services use `LaunchType: EC2`, not `CapacityProviderStrategy`, so skipping this step is fine — tasks still place correctly on the ASG's instances because they're registered with the cluster via user-data.

### 5 — ALB + target group (listener done manually)

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-alb \
  --template-file 5_alb.yaml \
  --parameter-overrides \
    VpcId=<VPC_ID> \
    PublicSubnetIds=<subnet-c>\\,<subnet-d>
```

**Post-5 manual steps:**

1. **Issue the ACM cert** for `<PROD_DOMAIN>` with DNS validation:
   ```bash
   aws acm request-certificate --domain-name <PROD_DOMAIN> --validation-method DNS
   aws acm describe-certificate --certificate-arn <CERT_ARN> \
     --query 'Certificate.DomainValidationOptions[0].ResourceRecord'
   ```
   Add the returned CNAME to Cloudflare as **DNS only** (grey cloud).
2. **Create the HTTPS listener** on the ALB, forwarding to the target group created by stack 5:
   ```bash
   ALB_ARN=$(aws cloudformation describe-stacks \
     --stack-name activepieces-prod-alb \
     --query 'Stacks[0].Outputs[?OutputKey==`LoadBalancerArn`].OutputValue' --output text)
   TG_ARN=$(aws cloudformation describe-stacks \
     --stack-name activepieces-prod-alb \
     --query 'Stacks[0].Outputs[?OutputKey==`TargetGroupArn`].OutputValue' --output text)

   aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
     --protocol HTTPS --port 443 \
     --certificates CertificateArn=<CERT_ARN> \
     --default-actions Type=forward,TargetGroupArn=$TG_ARN

   aws elbv2 create-listener --load-balancer-arn $ALB_ARN \
     --protocol HTTP --port 80 \
     --default-actions Type=redirect,RedirectConfig='{Protocol=HTTPS,Port=443,StatusCode=HTTP_301}'
   ```
3. **Point Cloudflare at the ALB** with a proxied CNAME (orange cloud) for `<PROD_DOMAIN>` → the `LoadBalancerDnsName` output. In Cloudflare SSL/TLS → Overview, set mode to **Full (strict)**.

### 6 — task definitions (APP + WORKER)

First, make sure ECR has a `main-latest` image so the task def registers against something that exists:

```bash
# From the repo root on your Mac (Apple Silicon → x86 EC2)
aws ecr get-login-password \
  | docker login --username AWS \
    --password-stdin 557690597652.dkr.ecr.eu-central-1.amazonaws.com
docker buildx build --platform linux/amd64 --push \
  -t 557690597652.dkr.ecr.eu-central-1.amazonaws.com/connectors/activepieces:main-latest \
  .
```

Then:

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-task-definitions \
  --template-file 6_task_definitions.yaml \
  --parameter-overrides \
    ProdDomain=<PROD_DOMAIN>
```

### 7 — ECS services (APP + WORKER)

```bash
aws cloudformation deploy \
  --stack-name activepieces-prod-ecs-services \
  --template-file 7_ecs_services.yaml
```

Service names are auto-generated from the stack id — look them up:

```bash
aws cloudformation describe-stacks \
  --stack-name activepieces-prod-ecs-services \
  --query 'Stacks[0].Outputs'
```

**Post-7 manual steps — complete the WORKER auth loop:**

1. Wait for the APP service to become stable and the ALB target health to report healthy (1–2 min after tasks start):
   ```bash
   curl -sS -o /dev/null -w "HTTP %{http_code}\n" https://<PROD_DOMAIN>/api/v1/flags
   ```
2. Generate a worker token signed with the APP's JWT secret and store it:
   ```bash
   # From the repo root
   npm install
   npm run workers token
   # When prompted, paste the value of:
   aws secretsmanager get-secret-value \
     --secret-id /activepieces/prod/jwt-secret \
     --query SecretString --output text
   # Copy the printed JWT and store it:
   aws secretsmanager put-secret-value \
     --secret-id /activepieces/prod/worker-token \
     --secret-string "<paste-the-token>"
   ```
3. Force the worker service to restart so it picks up the new token:
   ```bash
   WORKER_SVC=$(aws cloudformation describe-stacks \
     --stack-name activepieces-prod-ecs-services \
     --query 'Stacks[0].Outputs[?OutputKey==`WorkerServiceName`].OutputValue' --output text)
   aws ecs update-service --cluster ConnectorsAppCluster \
     --service "$WORKER_SVC" --force-new-deployment
   ```
4. Verify workers connect: open `https://<PROD_DOMAIN>` → Platform Admin → Infra → Workers. The count should match `WorkerDesiredCount` (default 2).

## After bootstrap: CI/CD takes over

Commit `.github/workflows/deploy-prod.yml` (see `../bootstrap.md` §17). It:
1. Builds on merged PRs to `main`,
2. Pushes `main-<sha>` + `main-latest` to ECR,
3. Renders the current task def with the new image and registers a new revision,
4. Updates the service — APP first (runs migrations), WORKER after.

**Drift expectation:** CI registers new task def revisions outside CFN. Stack 6 (task definitions) becomes a structural-changes-only stack: it owns env vars, IAM role ARNs, cpu/memory, etc. Re-running it rebases the task def to the CFN-specified image, causing a short rollback until CI catches up on the next merge. Avoid re-running stack 6 unless you intend to change something declared in it.

## Shared parameters quick-reference

| Template | Required overrides |
| --- | --- |
| `1_foundation.yaml` | `VpcId` |
| `2_database.yaml` | `PrivateSubnetIds` (comma-separated, escaped) |
| `3_cache.yaml` | `PrivateSubnetIds` |
| `4_ec2_capacity.yaml` | `PrivateSubnetIds` |
| `5_alb.yaml` | `VpcId`, `PublicSubnetIds` |
| `6_task_definitions.yaml` | `ProdDomain` |
| `7_ecs_services.yaml` | _(none — all defaults sensible)_ |

## Tearing down

Order is **reverse of create**: 7 → 6 → 5 → 4 → 3 → 2 → 1. Caveats:

- Stack 1's S3 bucket and log groups have `DeletionPolicy: Retain` — delete them by hand if you truly want to reclaim the names.
- Stack 2's RDS has `DeletionPolicy: Snapshot` and `DeletionProtection: true`. Disable deletion protection first: `aws rds modify-db-instance --db-instance-identifier activepieces-prod-db --no-deletion-protection --apply-immediately`.
- Stack 3's ElastiCache has `DeletionPolicy: Retain`. Delete via `aws elasticache delete-cache-cluster --cache-cluster-id activepieces-prod-redis` when you're sure.
- Stack 5's ALB has `deletion_protection.enabled=true`. Disable via the console or `aws elbv2 modify-load-balancer-attributes` before deleting the stack.
- Manual listener created post-5 must be deleted manually before stack 5 teardown (CFN won't manage it).

## Template map

```
1_foundation.yaml          SGs (4), IAM roles (4), Secrets (4), S3, LogGroups (2)
2_database.yaml            RDS Postgres 14 + subnet/parameter groups
3_cache.yaml               ElastiCache Redis 7 (single-node)
4_ec2_capacity.yaml        LaunchTemplate + ASG + CapacityProvider (optional)
5_alb.yaml                 ALB + APP target group (no listener — operator adds)
6_task_definitions.yaml    TaskDef: APP, TaskDef: WORKER
7_ecs_services.yaml        Service: APP (ALB-attached), Service: WORKER
```
