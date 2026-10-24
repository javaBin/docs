# Authentication and Identity

How identity and access control work in the Javabin platform.

## Google Workspace Accounts

All javaBin heroes get a `@java.no` Google Workspace account, auto-created when their entry is added to `groups/heros.yaml` in the registry. An invite is sent to their personal email. Heroes can also request an email alias (e.g., `alex@java.no`) via PR.

## AWS Console and CLI: IAM Identity Center + Google Workspace

IAM Identity Center gives javaBin volunteers AWS console and CLI access, federated via Google Workspace SAML. Volunteers sign in with their `@java.no` account.

- Groups with `identity_center: true` in the registry's `groups/groups.yaml` are synced to Identity Center; `memberships` in `groups/heros.yaml` decides who is in them.
- General permission sets: `javabin-admin`, `javabin-developer`, `javabin-readonly`.
- Each registered team gets `javabin-team-<team>` for its members, scoped to the team's own resources. Use this one for app work (setting secrets, inspecting your services).

Set up the CLI with `javabin aws setup` (writes the SSO session and one profile per role you have, then signs in) and check it with `javabin aws doctor`. Install with `brew install javaBin/tap/javabin`.

## CI/CD: GitHub OIDC

App repos authenticate to AWS in CI with GitHub's OIDC provider. No long-lived credentials or repository secrets.

- The platform creates two CI roles per registry team: `javabin-ci-team-<team>` (Terraform) and `javabin-ci-deploy-<team>` (image push, ECS deploy). Teams not in the registry have none.
- Only the platform's own workflows can assume the broker roles. A platform Lambda verifies the job's OIDC token and that the repo belongs to the GitHub team named in `app.yaml` `team:` before handing out short-lived team credentials.
- Team roles are limited by a permission boundary to resources prefixed `<team>-`.
- Terraform apply gets its credentials from the apply-gate Lambda, and only after the plan's LLM risk review (run by the platform's `plan-review` Lambda) passes. HIGH or FAILED reviews need a board-member override via `approve-override.yml`. See [apply gate](https://github.com/javaBin/platform/blob/main/docs/apply-gate.md).

## App Authentication: Cognito User Pools

| Pool | Purpose | Users |
|------|---------|-------|
| **Internal** | javaBin volunteers, admin tools | `@java.no` accounts via Google Workspace SAML |
| **External** | Public-facing apps | Self-registration |

Apps declare auth in `app.yaml`:

```yaml
auth:
  pool: internal          # or external
  client: public          # confidential (default) | public | none
  callback_urls:          # optional; defaults derive from routing.host
    - https://cupcake.javazone.no/auth/callback
    - http://localhost:3000/   # localhost is allowed for local dev
  groups: [helter]        # optional: only these registry groups may sign in
  env:                    # optional: also inject values under your own names
    COGNITO_CLIENT_ID: NUXT_PUBLIC_OIDC_CLIENT_ID
```

`auth: internal` is shorthand for the defaults.

- `client: confidential` — server-side login; you get a client ID and secret.
- `client: public` — browser apps doing OIDC with PKCE; client ID, no secret.
- `client: none` — APIs that only verify tokens; you get the pool/issuer values, no client.

The container receives `COGNITO_USER_POOL_ID`, `COGNITO_DOMAIN`, `COGNITO_ISSUER_URL`, `COGNITO_WELL_KNOWN_URL`, and depending on `client`, `COGNITO_CLIENT_ID` and `COGNITO_CLIENT_SECRET`.

**Groups.** Internal-pool tokens carry the user's registry groups in `cognito:groups` (every `cognito: true` group they belong to, plus `team-<team>`), added at each sign-in and token refresh. `auth.groups` refuses sign-in to anyone outside the listed groups. Apps should still check `cognito:groups` for finer-grained authorization; a `client: none` API accepts tokens issued to any client in the pool, so it must check.

**How clients are created.** App CI only requests a client (an SSM parameter under the team's own path). The platform's `app-auth-sync` Lambda creates the Cognito client `{team}-{name}` and its credentials, so teams can never touch each other's clients.

**Google Workspace.** Sign-in to the internal pool goes through the Google Workspace SAML app "Javabin Apps (Cognito)". It must be turned on for a user's organizational unit; it is on for java.no.

Details: [app.yaml reference — auth](https://github.com/javaBin/platform/blob/main/docs/app-yaml-reference.md#auth). Cupcake (`javaBin/frosting`, `client: public`) and cupcake-backend (`javaBin/cupcake`, `client: none`) show the frontend/API split.

## Legacy: Auth0

Older JavaZone apps still use Auth0 at `login.javazone.no`. It stays as-is; new apps on the platform use Cognito.

## Developer CLI Authentication

The `javabin` CLI uses:

- **GitHub**: `gh auth token` or `GITHUB_TOKEN`/`GH_TOKEN`
- **AWS**: the standard credential chain; `javabin secret` picks your `javabin-team-<team>` SSO profile automatically

## Summary

| Context | Mechanism |
|---------|-----------|
| AWS console / CLI | Identity Center + Google Workspace SAML (`javabin aws setup`) |
| CI/CD | GitHub OIDC → platform broker → team CI roles |
| App users (internal) | Cognito internal pool + Google Workspace SAML |
| App users (external) | Cognito external pool |
| Legacy app auth | Auth0 (`login.javazone.no`) |
| Developer CLI | gh CLI + AWS SSO profiles |
