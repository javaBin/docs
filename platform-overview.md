# Javabin Platform Overview

The Javabin platform is shared AWS infrastructure that gives javaBin app repos automated CI/CD, infrastructure provisioning, monitoring, and cost management.

## Architecture

```
                    javaBin/registry
                    (app + team YAML)
                          │
                          ▼ repository_dispatch
                    javaBin/platform
                    (Terraform + workflows)
                          │
          ┌───────────────┼───────────────┐
          ▼               ▼               ▼
     IAM / OIDC      ECS Cluster      Monitoring
     (CI roles)      (Fargate)        (SNS, Lambda)
          │               │               │
          ▼               ▼               ▼
     App repos       ALB routing      Slack alerts
     assume roles    *.javazone.no    cost reports
     in CI                            compliance
```

## Key Components

### Shared Infrastructure (`terraform/platform/`)

Seven Terraform sub-modules manage shared resources:

| Module | Resources |
|--------|-----------|
| **networking** | VPC, public/private subnets across 3 AZs, NAT gateway, security groups |
| **ingress** | ALB, ACM wildcard certificate for `*.javazone.no`, Route53 DNS |
| **iam** | GitHub OIDC provider, per-app CI roles, permission boundary |
| **compute** | ECS Fargate cluster (`javabin-platform`), ECR base config |
| **monitoring** | SNS topics, EventBridge rules, AWS Config, GuardDuty, Security Hub |
| **lambdas** | 6 Lambda functions for alerts, cost reporting, compliance, cleanup |
| **identity** | IAM Identity Center + Cognito user pools (stub, pending Google Admin access) |

### Reusable Modules (`terraform/modules/`)

Twelve Terraform modules that app repos source via `git::` URLs. The key one is `app-stack`, the golden path module that reads `app.yaml` and creates all infra for a service (ECR, ECS service, ALB routing, IAM role, optional S3/DynamoDB/SQS/Secrets Manager).

### Reusable Workflows (`.github/workflows/`)

App repos call `javaBin/platform/.github/workflows/javabin.yml` as their CI entrypoint. It orchestrates:

1. **Detect** repo contents (JVM, TypeScript, Docker, Terraform)
2. **Build** (Maven or pnpm)
3. **Docker build** + ECR push
4. **Terraform plan** + S3 artifact upload
5. **LLM risk review** via Bedrock (blocks HIGH risk changes)
6. **Terraform apply** (SHA-verified from plan artifact)
7. **ECS deploy** (task definition update)

### Lambda Functions

| Function | Trigger | Purpose |
|----------|---------|---------|
| `slack-alert` | SNS subscription | Routes security/cost events to Slack with LLM analysis |
| `cost-report` | Weekly schedule (Mon 08:00 UTC) | Cost breakdown with LLM narrative |
| `daily-cost-check` | Daily schedule (08:00 UTC) | Spike detection, silent if no anomalies |
| `compliance-reporter` | EventBridge (resource create/run) | Reports untagged resources to Slack |
| `override-cleanup` | Hourly schedule | Deletes stale SSM override tokens |
| `team-provisioner` | Registry merge (stub) | Syncs teams across Google, GitHub, Cognito, IAM |

## How Apps Get CI/CD

1. Developer creates a repo from `javaBin/app-template`
2. Registers the app in `javaBin/registry` (PR with `apps/{name}.yaml`)
3. Platform owner merges — platform provisions IAM role, ECR repo, budget
4. Developer adds `app.yaml` to repo root with service config
5. Every push to `main` triggers the full pipeline: build, plan, review, deploy

No per-repo workflow files needed. The platform handles everything.

## AWS Account

- **Account**: (private — see platform repo)
- **Region**: eu-central-1 (Frankfurt)
- **Domain**: javazone.no (Route53)
- **ECS cluster**: javabin-platform
