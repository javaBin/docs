# Authentication and Identity

How identity and access control work in the Javabin platform.

## Console Access: IAM Identity Center + Google Workspace

IAM Identity Center provides AWS console and CLI access for javaBin volunteers, federated via Google Workspace SAML.

- Google Workspace groups map to IAM Identity Center permission sets
- Volunteers sign in with their `@java.no` Google account
- Access is scoped by 3 permission sets: admin, developer, read-only
- 2FA is enforced on all Identity Center users

**Status**: Live. IAM Identity Center deployed with SAML federation to Google Workspace.

## CI/CD Access: GitHub OIDC

App repos authenticate to AWS in CI using GitHub's OIDC provider — no long-lived credentials.

- The platform provisions a per-app IAM role when a team is registered
- GitHub Actions assumes the role via OIDC with conditions on the repo name and branch
- Roles are scoped by a permission boundary that limits actions to the app's own resources
- The reusable workflow `javabin.yml` handles role assumption automatically

## App Authentication: Cognito User Pools

For apps that need user authentication, the platform provides Cognito user pools.

| Pool | Purpose | Users |
|------|---------|-------|
| **Internal** | javaBin volunteers, admin tools | Synced from Google Workspace |
| **External** | Public-facing apps (conference registration, etc.) | Self-registration |

Apps declare their auth needs in `app.yaml`:

```yaml
auth: internal    # or: external, both, none
```

The platform creates a Cognito app client and passes the client ID/secret via environment variables.

**Status**: Deployed. Both Cognito pools are managed in Terraform (`terraform/platform/identity/`). Apps that need auth can use Cognito or continue using Auth0 at `login.javazone.no`.

## Developer CLI Authentication

The `javabin` CLI (4 commands: init, register, status, whoami) authenticates via:

- **GitHub**: `gh auth token` (gh CLI) or `GITHUB_TOKEN` environment variable
- **AWS**: Standard credential chain (env vars, `~/.aws/credentials`, SSO via Identity Center)

## Summary of Auth Mechanisms

| Context | Mechanism | Status |
|---------|-----------|--------|
| AWS Console | Identity Center + Google SAML + 2FA | Deployed |
| CI/CD | GitHub OIDC | Deployed |
| App users (internal) | Cognito internal pool | Deployed |
| App users (external) | Cognito external pool | Deployed |
| Legacy app auth | Auth0 (`login.javazone.no`) | Active |
| Developer CLI | gh CLI + AWS credential chain | Implemented |
