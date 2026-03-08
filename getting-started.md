# Getting Started with the Javabin Platform

How to register a team, create a service, and get automated CI/CD.

## Prerequisites

- Your repo must be in the `javaBin` GitHub organization
- Your team must be registered in [javaBin/registry](https://github.com/javaBin/registry) (or register it as part of this process)

## Option 1: Using the CLI

```bash
brew install javaBin/tap/javabin

# Scaffold a new repo from the app-template
javabin init

# Register your team (opens a PR against javaBin/registry)
javabin register
```

The `init` command scaffolds a new service repo from [javaBin/app-template](https://github.com/javaBin/app-template), including `app.yaml` and `Dockerfile`.

The `register` command prompts for team details and creates a registration PR automatically.

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
```

- `google_group` is derived automatically as `team-{name}@java.no`
- Budget defaults to 500 NOK/mo
- All members become GitHub team maintainers

### Step 2: Open a PR

A platform owner (`@javaBin/platform-owners`) reviews and merges it. This triggers team provisioning (Google Group, GitHub team, Cognito group, IAM).

### Step 3: Add app.yaml to your repo

Add an `app.yaml` to your repo root to configure compute, routing, and resources. See the [app.yaml reference](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md).

## What Happens After Registration

1. The platform provisions:
   - A **Google Group** (`team-{name}@java.no`)
   - A **GitHub team** with all members as maintainers
   - An **IAM role** your CI pipeline assumes via GitHub OIDC
   - An **ECR repository** for your Docker images
   - A **budget alert** for cost monitoring

2. Push to `main`. The platform CI pipeline automatically:
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
4. Push — CI takes over

## Multi-Environment Support

`app.yaml` supports an `environments:` block for dev/prod separation. If omitted, a single production environment is created. See the [app.yaml reference](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md#environments) for details.

## No Per-Repo Workflow Files

The platform provides reusable workflows. Your repo does not need its own `.github/workflows/` directory for standard CI/CD. The entrypoint workflow (`javabin.yml`) detects your repo contents and runs the appropriate build and deploy steps.
