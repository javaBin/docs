# Getting Started with the Javabin Platform

How to register a team, add repos, and get automated CI/CD.

## Prerequisites

- Your repo must be in the `javaBin` GitHub organization
- Your team must be registered in [javaBin/registry](https://github.com/javaBin/registry) (or register it as part of this process)

## Option 1: Using the CLI

```bash
brew install javaBin/tap/javabin

# Scaffold a new repo from the app-template
javabin init

# Register your team (opens a PR against javaBin/registry)
javabin register-team
```

The `init` command scaffolds a new service repo from [javaBin/app-template](https://github.com/javaBin/app-template), including `app.yaml` and `Dockerfile`.

The `register-team` command prompts for team details and creates a team registration PR automatically.

Other commands: `javabin status` (check platform status), `javabin whoami` (show current identity).

## Option 2: Manual PR

### Step 1: Register your team (if not already registered)

Create `teams/your-team.yaml` in [javaBin/registry](https://github.com/javaBin/registry):

```yaml
name: your-team
description: What your team works on
members:
  - google: firstname.lastname
    github: github-username
budget_nok: 5000  # Optional, defaults to 500 NOK/mo
```

- `google_group` is derived automatically as `team-{name}@java.no`
- Budget defaults to 500 NOK/mo
- All members become GitHub team maintainers

### Step 2: Open a PR

A platform owner (`@javaBin/platform-owners`) reviews and merges it. This triggers team provisioning (Google Group, GitHub team, Cognito group, IAM).

### Step 3: Add repos to your team

After your team is provisioned, add your repos to the GitHub team:

**Via GitHub UI:**
1. Go to [github.com/orgs/javaBin/teams](https://github.com/orgs/javaBin/teams)
2. Select your team
3. Go to the **Repositories** tab
4. Click **Add repository** and select your repo

**Via CLI:**
```bash
gh api orgs/javaBin/teams/TEAM/repos -f owner=javaBin -f repo=REPO -f permission=push
```

Replace `TEAM` with your team name and `REPO` with the repository name.

### Step 4: Add app.yaml to your repo

Add an `app.yaml` to your repo root to configure compute, routing, and resources. See the [app.yaml reference](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md).

For CI/CD to work on a repo, the repo needs:
- To be under a registered team (GitHub team membership)
- An `app.yaml` at the repo root

## What Happens After Registration

1. The platform provisions:
   - A **Google Group** (`team-{name}@java.no`)
   - A **GitHub team** with all members as maintainers
   - A **Cognito group** for identity management
   - **IAM resources** your CI pipeline assumes via GitHub OIDC
   - A **budget alert** for cost monitoring

2. After you add a repo to your team and include an `app.yaml`, every push to `main` triggers the full pipeline:
   - Builds your code (Java/Maven or TypeScript/pnpm)
   - Builds and pushes a Docker image
   - Plans and applies Terraform infrastructure
   - Deploys to ECS Fargate
   - Sets up CloudWatch alarms

## Creating a New Repo from Template

Use the CLI or start from [javaBin/app-template](https://github.com/javaBin/app-template):

1. Run `javabin init` (recommended), or click **"Use this template"** on GitHub
2. Customize `app.yaml` and `Dockerfile`
3. Register your team in the registry (see above)
4. Add the repo to your GitHub team (see Step 3 above)
5. Push — CI takes over

## Multi-Environment Support

`app.yaml` supports an `environments:` block for dev/prod separation. If omitted, a single production environment is created. See the [app.yaml reference](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md#environments) for details.

## Hero Membership

The registry is also the source of truth for all javaBin hero volunteers (~70 people/year). This is separate from team registration — teams are for app developers who need AWS access, heroes are all active volunteers.

### How hero membership works

1. Board sends yearly Google Forms for hero applications
2. Responses go to a Google Sheet, board marks approved heroes
3. Board triggers a sync (Apps Script button or GitHub Actions workflow)
4. Sync creates a PR on the registry updating `groups/heros.yaml`
5. PR reviewed and merged — triggers provisioning:
   - Google Workspace accounts auto-created for new heroes
   - All heroes added to `helter@java.no`
   - Heroes added to their membership groups (styret, javazone, pkom, kodesmia, region, etc.)

### Registry structure

```
registry/
  teams/              # App developer teams (AWS IAM, budgets, GitHub teams)
  groups/
    groups.yaml       # All group definitions (Google email, Cognito/IC flags)
    heros.yaml        # All hero members (name, email, alias, memberships)
```

- `groups.yaml` defines what groups exist and their properties
- `heros.yaml` defines who the heroes are and which groups they belong to
- Being in `heros.yaml` = member of `helter` (implicitly)
- The `memberships` field controls additional group assignments
- Heroes can PR an email alias (e.g., `alex@java.no`)

## No Per-Repo Workflow Files

The platform provides reusable workflows. Your repo does not need its own `.github/workflows/` directory for standard CI/CD. The entrypoint workflow (`javabin.yml`) detects your repo contents and runs the appropriate build and deploy steps.
