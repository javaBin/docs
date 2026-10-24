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
| **iam** | GitHub OIDC provider, per-team CI roles for registry teams, permission boundary |
| **compute** | ECS Fargate cluster (`javabin-platform`), ECR base config |
| **monitoring** | SNS topics, EventBridge rules, AWS Config, GuardDuty, Security Hub |
| **lambdas** | Lambda functions for alerts, cost reporting, compliance, budget enforcement, tagging, team provisioning, CI credential brokering, plan review, Cognito clients, secret rollout |
| **identity** | Cognito user pools (internal + external), `auth-claims` trigger. Internal pool signs in via Google Workspace SAML. Identity Center is in `terraform/org/` |

### Reusable Modules (`terraform/modules/`)

Terraform modules that app repos source via `git::` URLs. CI validates `app.yaml` strictly and generates Terraform module blocks from it with `generate-modules.py`, committing them to the branch. Supported resources: ECR, ECS service, ALB routing, IAM role (configurable for ECS/EC2/Lambda), S3, DynamoDB, RDS PostgreSQL, SQS, SSM secrets, Cognito app clients (requested via SSM, created by the platform). Cross-service access is auto-wired via `access_policy_json` outputs.

### Reusable Workflows (`.github/workflows/`)

App repos call `javaBin/platform/.github/workflows/javabin.yml` from a small `deploy.yml` (no secrets; `permissions: id-token, contents, pull-requests: write`). It orchestrates:

1. **Detect** repo contents (`app.yaml`, Maven/Gradle, pnpm, Docker)
2. **Build** (Maven, Gradle or pnpm)
3. **Expand** `app.yaml` into Terraform and commit it to the branch
4. **Terraform plan** + S3 artifact upload
5. **LLM risk review** of the whole plan by the `plan-review` Lambda (Bedrock); result posted on PRs
6. **Terraform apply** with credentials from the `apply-gate` Lambda (SHA-verified plan; HIGH/FAILED risk needs a board override via `approve-override.yml`)
7. **Docker build** + ECR push, then **ECS deploy**

PRs stop after plan + review. See [reusable workflows](https://github.com/javaBin/platform/blob/main/docs/reusable-workflows.md).

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
| `team-provisioner` | Registry merge | Syncs Google Groups, GitHub teams, AWS Budgets, Identity Center, group memberships, hero accounts, Atlassian groups |
| `ci-broker` | CI invocation | Verifies the job's GitHub OIDC token and repo → `app.yaml` team, vends short-lived team CI credentials |
| `plan-review` | CI invocation (tf-plan) | LLM risk review of app Terraform plans; writes the verdict the apply gate reads |
| `apply-gate` | CI invocation (tf-apply) | Credential broker for apply; blocks HIGH/FAILED risk unless a board override exists |
| `app-auth-sync` | SSM change + daily reconcile | Creates/deletes Cognito app clients requested by `app.yaml` `auth:` |
| `auth-claims` | Cognito sign-in / refresh | Adds registry groups to `cognito:groups`, enforces `auth.groups` |
| `app-secret-rollout` | SSM change | Restarts the app's ECS service after `javabin secret set` |
| `securityhub-summary` | Weekly schedule (Mon 08:00 UTC) | HIGH/CRITICAL Security Hub findings summary |
| `password-set` | Function URL | Self-service password set for new hero accounts |

Details: [lambda-functions.md](https://github.com/javaBin/platform/blob/main/docs/lambda-functions.md).

## How Apps Get CI/CD

1. Register the team in `javaBin/registry` (`teams/<team>.yaml`, or `javabin register-team`). The merge creates the GitHub team, the team's CI roles (`javabin-ci-team-<team>`, `javabin-ci-deploy-<team>`), the SSO permission set `javabin-team-<team>` and a budget.
2. Add the repo to that GitHub team.
3. Add `app.yaml` and `.github/workflows/deploy.yml` (or start with `javabin init`).
4. PRs get plan + review; pushes to `main` (or a manual `workflow_dispatch`) deploy.

The registry is the IAM gate: unregistered teams have no CI roles, so CI can't deploy. The repo must be in the GitHub team named in `app.yaml` `team:`. Step-by-step: [Getting Started](getting-started.md). Cupcake (`cupcake.javazone.no`, `cupcake-backend.javazone.no`) is the first app running this way.

## Registry: Source of Truth for Memberships

The [registry](https://github.com/javaBin/registry) serves two purposes:

**Teams** (`teams/`) — app developer teams that need AWS access. Each team gets CI roles, an SSO permission set, a budget, a GitHub team, and a Google Group.

**Groups & Heroes** (`groups/`) — organizational membership for all javaBin volunteers:
- `groups.yaml` defines all groups (helter, styret, javazone, pkom, kodesmia, region, drift, admin, developers, etc.) with their properties (Google Workspace email, Cognito/Identity Center flags)
- `heros.yaml` defines all hero members with their group assignments

Changes to `groups/` trigger provisioning: Google Workspace account creation, group membership sync, email aliases, and Cognito/Identity Center sync where configured. Heroes are synced from a yearly Google Sheets application process.

## Tag Schema

Every AWS resource gets 7 tags — 5 static (Terraform-managed) and 2 dynamic (auto-applied by the resource-tagger Lambda):

| Tag | Source | Example | Purpose |
|-----|--------|---------|---------|
| `team` | app.yaml / default_tags | `core` | Access control, cost attribution, budgets |
| `service` | app.yaml / default_tags | `moresleep` | Cost breakdown within team |
| `repo` | app.yaml / default_tags | `javaBin/moresleep` | Link resource to source code |
| `environment` | default_tags | `production` | Environment label (one environment per app today) |
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
