# Getting Started with the Javabin Platform

How to register a team, deploy an app with `app.yaml`, and get AWS access as a developer.

[Cupcake](https://github.com/javaBin/frosting) (`cupcake.javazone.no`, frontend) and [cupcake-backend](https://github.com/javaBin/cupcake) (`cupcake-backend.javazone.no`, API) are the first apps running on the platform from `app.yaml`. Their repos are good working examples.

## 1. Register your team

Teams are the registration unit; there is no separate app registration.

```bash
brew install javaBin/tap/javabin   # releases are automatic; update with: brew upgrade javabin
javabin register-team              # opens a PR against javaBin/registry
```

Or create `teams/<team>.yaml` in [javaBin/registry](https://github.com/javaBin/registry) by hand:

```yaml
name: yourteam          # lowercase letters only, max 12 chars
description: What your team works on
members:
  - google: firstname.lastname
    github: github-username
budget_nok: 5000        # optional, default 500 NOK/month
```

A platform owner (`@javaBin/platform-owners`) reviews and merges. The merge provisions:

- a Google Group `team-<team>@java.no` and a GitHub team (members become maintainers)
- the team's AWS CI roles `javabin-ci-team-<team>` and `javabin-ci-deploy-<team>` (only registry teams get them, so an unregistered team cannot deploy)
- an IAM Identity Center permission set `javabin-team-<team>` for the members
- a monthly AWS budget (alert at 80%, services scaled to zero at 200%)

## 2. Add the repo to your team

The repo must belong to the GitHub team named in `app.yaml` `team:`. A repo may be in several teams; the `app.yaml` one is used.

```bash
gh api -X PUT orgs/javaBin/teams/<team>/repos/javaBin/<repo> -f permission=push
```

Or in the GitHub UI: **Teams → your team → Repositories → Add repository**.

## 3. Add `app.yaml` and `deploy.yml`

`javabin init` scaffolds a new repo from [javaBin/app-template](https://github.com/javaBin/app-template) with `app.yaml`, `Dockerfile` and the deploy workflow. For an existing repo, add them yourself.

`app.yaml` at the repo root declares compute, routing, resources and auth. It is validated strictly: unknown or unsupported fields fail the pipeline with a message. See the [app.yaml reference](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md).

```yaml
name: myapp
team: yourteam
routing:
  host: myapp.javazone.no
  priority: 120
```

`.github/workflows/deploy.yml` calls the platform pipeline. No repository or organization secrets are needed; AWS access is GitHub OIDC.

```yaml
name: Deploy
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch:      # manual deploys

permissions:
  id-token: write
  contents: write         # the pipeline commits generated Terraform to the branch
  pull-requests: write    # plan + review comment on PRs

jobs:
  javabin:
    uses: javaBin/platform/.github/workflows/javabin.yml@main
```

Pull requests get a Terraform plan and an LLM risk review as a PR comment. Pushes to `main` build (Maven, Gradle or pnpm), apply Terraform, push the Docker image and deploy to ECS Fargate. Plans the review rates HIGH (or that it could not review, FAILED) are blocked until a board member approves an override. See [reusable workflows](https://github.com/javaBin/platform/blob/main/docs/reusable-workflows.md).

## 4. Developer AWS access

Sign in with your `@java.no` Google account through IAM Identity Center. Needs the AWS CLI v2 (`brew install awscli`).

```bash
javabin aws setup     # writes the SSO session + one profile per role you have, then signs in
javabin aws doctor    # checks config, token, and which profile javabin would use
```

Use the `javabin-team-<team>` profile for your team's resources (`--profile javabin-team-<team>` or `AWS_PROFILE`). Sessions last about 8 hours; renew with `aws sso login --sso-session javabin`.

## 5. Secrets

Declare secrets in `app.yaml`:

```yaml
resources:
  secrets:
    - name: api-key
      env: API_KEY
```

The first deploy creates each one in SSM with the value `PLACEHOLDER`. Set the real value from the app repo:

```bash
javabin secret list           # which secrets still hold PLACEHOLDER
javabin secret set api-key    # prompts without echo; --stdin to pipe
```

The platform restarts the service within a minute so the new value is picked up.

## Not supported

These fail the pipeline with an explanation (see [Not supported](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md#not-supported)):

- `environments:` — one `app.yaml` deploys one environment. For a dev environment, deploy a second app (e.g. `name: myapp-dev`) from its own repo.
- `budget_alert_nok` — budgets are per team: set `budget_nok` in the registry team file.
- `auth.pool: both` — run two services, one per pool.

## Hero membership

The registry is also the source of truth for all javaBin hero volunteers (~70 people/year). This is separate from teams: teams are for app developers who need AWS access, heroes are all active volunteers.

1. The board sends yearly Google Forms for hero applications; approved heroes are marked in the response sheet.
2. A sync opens a PR on the registry updating `groups/heros.yaml`.
3. Merging provisions Google Workspace accounts for new heroes, adds everyone to `helter@java.no`, and adds heroes to their membership groups (styret, javazone, pkom, kodesmia, region, etc.).

```
registry/
  teams/              # app developer teams (GitHub team, AWS roles, budget)
  groups/
    groups.yaml       # group definitions (Google email, Cognito/Identity Center flags)
    heros.yaml        # heroes (name, email, alias, memberships)
```

Being in `heros.yaml` implies membership of `helter`; the `memberships` field adds other groups. Heroes can PR an email alias (e.g. `alex@java.no`).
