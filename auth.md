# Authentication and Identity

How identity and access control work in the Javabin platform.

## Console Access: IAM Identity Center + Google Workspace

IAM Identity Center provides AWS console and CLI access for javaBin volunteers, federated via Google Workspace SAML.

- Google Workspace groups map to IAM Identity Center permission sets
- Volunteers sign in with their `@java.no` Google account
- Access is scoped by permission sets (admin, read-only, developer)

**Status**: Planned. Blocked on Google Workspace admin access. Currently, console access is via IAM users.

## CI/CD Access: GitHub OIDC

App repos authenticate to AWS in CI using GitHub's OIDC provider — no long-lived credentials.

- The platform provisions a per-app IAM role when an app is registered
- GitHub Actions assumes the role via OIDC with conditions on the repo name and branch
- Roles are scoped by a permission boundary that limits actions to the app's own resources
- The reusable workflow `javabin.yml` handles role assumption automatically

## App Authentication: Cognito User Pools

For apps that need user authentication, the platform provides Cognito user pools.

Two pools are planned:

| Pool | Purpose | Users |
|------|---------|-------|
| **Internal** | javaBin volunteers, admin tools | Synced from Google Workspace |
| **External** | Public-facing apps (conference registration, etc.) | Self-registration |

Apps declare their auth needs in `app.yaml`:

```yaml
auth: internal    # or: external, both, none
```

The platform creates a Cognito app client and passes the client ID/secret via environment variables.

**Status**: Planned but not yet deployed. The `identity` module in the platform repo is a stub. Implementation is blocked on Google Workspace admin access (for internal pool sync). Apps that need auth today use Auth0 at `login.javazone.no`.

## Developer CLI Authentication

The `javabin` CLI authenticates via:

- **GitHub**: `gh auth token` (gh CLI) or `GITHUB_TOKEN` environment variable
- **AWS**: Standard credential chain (env vars, `~/.aws/credentials`, SSO)
- **Cognito**: TODO — device flow auth will be added when Cognito pools are deployed

## Summary of Auth Mechanisms

| Context | Mechanism | Status |
|---------|-----------|--------|
| AWS Console | Identity Center + Google SAML | Planned |
| CI/CD | GitHub OIDC | Implemented |
| App users (internal) | Cognito internal pool | Planned |
| App users (external) | Cognito external pool | Planned |
| Legacy app auth | Auth0 (`login.javazone.no`) | Active |
| Developer CLI | gh CLI + AWS credential chain | Implemented |
