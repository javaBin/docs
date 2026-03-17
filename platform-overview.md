# Javabin Platform Overview

The Javabin platform is shared AWS infrastructure that gives javaBin app repos automated CI/CD, infrastructure provisioning, monitoring, and cost management.

## Architecture

```
                    javaBin/registry
                    (team YAML)
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
| **iam** | GitHub OIDC provider, per-team CI roles (ABAC), permission boundary |
| **compute** | ECS Fargate cluster (`javabin-platform`), ECR base config |
| **monitoring** | SNS topics, EventBridge rules, AWS Config, GuardDuty, Security Hub |
| **lambdas** | 11 Lambda functions for alerts, cost reporting, compliance, budget enforcement, resource tagging, team provisioning |
| **identity** | Cognito user pools (internal + external). Internal pool connected to Google IdP. Identity Center is in `terraform/org/` (deployed) |

### Reusable Modules (`terraform/modules/`)

Twelve Terraform modules that app repos source via `git::` URLs. CI generates expanded Terraform from `app.yaml` using `expand-modules.py` + `registry.py`. Supported resources: ECR, ECS service, ALB routing, IAM role (configurable for ECS/EC2/Lambda), S3, DynamoDB, RDS PostgreSQL, SQS, Secrets Manager. Cross-service access is auto-wired via `access_policy_json` outputs.

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
| `cost-report` | Weekly schedule (Mon 08:00 UTC) | Cost breakdown with LLM narrative, per-team attribution |
| `daily-cost-check` | Daily schedule (08:00 UTC) | Spike detection with team breakdown, silent if no anomalies |
| `compliance-reporter` | EventBridge (resource create/run) | Reports untagged resources to Slack |
| `resource-tagger` | EventBridge (all AWS create/run) | Auto-tags created-by + commit from CI session names |
| `budget-enforcer` | SNS (AWS Budgets 200%) | Scales team's ECS services to zero, posts Slack alert |
| `override-cleanup` | Hourly schedule | Deletes stale SSM override tokens |
| `team-provisioner` | Registry merge | Syncs Google Groups, GitHub teams, AWS Budgets, Cognito, Identity Center, hero provisioning |
| `apply-gate` | CI invocation | Credential broker for gated Terraform apply with risk verification |
| `securityhub-summary` | Weekly schedule (Mon 08:00 UTC) | HIGH/CRITICAL Security Hub findings summary |
| `password-set` | Function URL | Self-service password set for new hero accounts |

## How Apps Get CI/CD

1. Developer creates a repo from `javaBin/app-template` (or runs `javabin init`)
2. Registers their team in `javaBin/registry` (PR with `teams/{name}.yaml`)
3. Platform owner merges — team-provisioner syncs Google Group, GitHub team, Cognito, IAM
4. Developer adds the repo to their GitHub team (via GitHub UI or `gh api orgs/javaBin/teams/TEAM/repos`)
5. Developer adds `app.yaml` to repo root with service config
6. Every push to `main` triggers the full pipeline: build, plan, review, deploy

The registry is the IAM gate: unregistered teams have no IAM role, so CI can't deploy. For CI/CD to work, a repo must be under a registered team and have an `app.yaml` at root.

No per-repo workflow files needed. The platform handles everything.

## Registry: Source of Truth for Memberships

The [registry](https://github.com/javaBin/registry) serves two purposes:

**Teams** (`teams/`) — app developer teams that need AWS access. Each team gets an IAM role, budget, GitHub team, and Google Group.

**Groups & Heroes** (`groups/`) — organizational membership for all javaBin volunteers:
- `groups.yaml` defines all groups (helter, styret, javazone, pkom, kodesmia, region, drift, admin, developers, etc.) with their properties (Google Workspace email, Cognito/Identity Center flags)
- `heros.yaml` defines all hero members with their group assignments

Changes to `groups/` trigger provisioning: Google Workspace account creation, group membership sync, email aliases, and Cognito/Identity Center sync where configured. Heroes are synced from a yearly Google Sheets application process.

## Tag Schema

Every AWS resource gets 7 tags — 5 static (Terraform-managed) and 2 dynamic (auto-applied by the resource-tagger Lambda):

| Tag | Source | Example | Purpose |
|-----|--------|---------|---------|
| `team` | app.yaml / default_tags | `web-team` | ABAC, cost attribution, budgets |
| `service` | app.yaml / default_tags | `moresleep` | Cost breakdown within team |
| `repo` | app.yaml / default_tags | `javaBin/moresleep` | Link resource to source code |
| `environment` | default_tags | `production` | Multi-env support |
| `managed-by` | default_tags | `terraform` | Distinguish TF vs console |
| `created-by` | resource-tagger Lambda | `alice` | Who created (set once) |
| `commit` | resource-tagger Lambda | `abc12345` | Which commit (set once) |

Cost allocation tags are activated in AWS, so Cost Explorer can group by `team` and `service`.

## Budget Enforcement

Teams get a monthly budget (default 500 NOK). Two thresholds:
- **80%** — SNS alert to #javabin-cost-alerts
- **200%** — `budget-enforcer` Lambda scales the team's ECS services to `desired_count=0` (not destroyed, easy recovery)

## AWS Account

- **Account**: (private — see platform repo)
- **Region**: eu-central-1 (Frankfurt)
- **Domain**: javazone.no (Route53)
- **ECS cluster**: javabin-platform
