# Getting Started with the Javabin Platform

How to register an app and get automated CI/CD.

## Prerequisites

- Your repo must be in the `javaBin` GitHub organization
- Your team must be registered in [javaBin/registry](https://github.com/javaBin/registry) (or register it as part of this process)

## Option 1: Using the CLI

```bash
brew install javaBin/tap/javabin
javabin register
```

The wizard prompts for repo name, team, auth, and budget, then creates a registration PR automatically.

## Option 2: Manual PR

### Step 1: Register your team (if not already registered)

Create `teams/your-team.yaml` in [javaBin/registry](https://github.com/javaBin/registry):

```yaml
name: your-team
description: What your team works on
members:
  - github-username-1
  - github-username-2
repos:
  - your-repo-name
```

### Step 2: Register your app

Create `apps/your-service.yaml` in the same registry repo:

```yaml
name: your-service
team: your-team
repo: javaBin/your-service
```

### Step 3: Open a PR

A platform owner (`@javaBin/platform-owners`) reviews and merges it.

## What Happens After Registration

1. The platform provisions:
   - An **IAM role** your CI pipeline assumes via GitHub OIDC
   - An **ECR repository** for your Docker images
   - A **budget alert** for cost monitoring

2. Add an `app.yaml` to your repo root to configure compute, routing, and resources. See the [app.yaml reference](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md).

3. Push to `main`. The platform CI pipeline automatically:
   - Builds your code (Java/Maven or TypeScript/pnpm)
   - Builds and pushes a Docker image
   - Plans and applies Terraform infrastructure
   - Deploys to ECS Fargate
   - Sets up CloudWatch alarms

## Creating a New Repo from Template

Start from [javaBin/app-template](https://github.com/javaBin/app-template):

1. Click **"Use this template"** on GitHub
2. Customize `app.yaml` and `Dockerfile`
3. Register in the registry (see above)
4. Push — CI takes over

## No Per-Repo Workflow Files

The platform provides reusable workflows. Your repo does not need its own `.github/workflows/` directory for standard CI/CD. The entrypoint workflow (`javabin.yml`) detects your repo contents and runs the appropriate build and deploy steps.
