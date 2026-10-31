# Authentication and Identity

How identity and access control work in the Javabin platform.

## Google Workspace Accounts

All javaBin heroes get a `@java.no` Google Workspace account, auto-created when their entry is added to `groups/members.yaml` in the registry. An invite is sent to their personal email. Heroes can also request an email alias (e.g., `alex@java.no`) via PR.

## AWS Console and CLI: IAM Identity Center + Google Workspace

IAM Identity Center gives javaBin volunteers AWS console and CLI access. Its identity source is Google Workspace: volunteers sign in with their `@java.no` Google account (SAML), and Google's AWS app provisions users into Identity Center (SCIM). It provisions only the **direct** members of `drift@`, `styret@` and `javabin-developer@java.no`; nested groups are not expanded.

- Each registered team gets the permission set `javabin-team-<team>`, assigned to the Identity Center group `team-<team>` (the members of `teams/<team>.yaml`). It is scoped to the team's own resources. Use this one for app work (setting secrets, inspecting your services).
- Groups with `identity_center: true` in the registry's `groups/groups.yaml` are synced to Identity Center with the `permission_set` they name; `memberships` in `groups/members.yaml` decides who is in them. Today: `styret` gets `javabin-readonly`, `drift` gets `javabin-admin`, `kodesmia` gets `javabin-developer`.
- `javabin-developer` cannot reach team resources. It relies on a `team` session tag that Identity Center never sets for humans, so its team-scoped rules match nothing. Use `javabin-team-<team>` instead.
- Sessions last 4 hours (every permission set uses `PT4H`).

### How sign-in works

1. A registry change creates your `@java.no` Google account (`groups/members.yaml`) and puts you in your team (`teams/<team>.yaml`) or in `kodesmia`.
2. The team provisioner adds team members and kodesmia members (by their `@java.no` address) as direct members of `javabin-developer@java.no`. Personal addresses can't be provisioned.
3. Google's SCIM sync then creates your Identity Center user, some time later.
4. A provisioner run every 3 hours adds your Identity Center user to `team-<team>`, which gives you the `javabin-team-<team>` role (and to `kodesmia` if you are in it). The registry run that added you to `javabin-developer@` skipped you because your user did not exist yet.
5. You sign in at `https://aws.javabin.no` (redirects to `https://javabin.awsapps.com/start/`), or from the CLI after `javabin aws setup`. The access portal lists one role per permission set you have.

Set up the CLI with `javabin aws setup` (writes the SSO session and one profile per role you have, then signs in) and check it with `javabin aws doctor`. Install with `brew install javaBin/tap/javabin`. Renew an expired session with `aws sso login --sso-session javabin`.

### When AWS sign-in fails

- **Google signs you in, but AWS refuses you or shows no account.** Identity Center has no user for your Google account. Google's SCIM sync may not have run yet after you were added to `javabin-developer@`; if it persists, you are not provisioned. You cannot fix this yourself: ask a platform admin.
- **Google says the app is not configured for you.** The AWS app is not turned on for your account. A Google Workspace admin has to enable it.
- **AWS sees the wrong Google account.** Google's SAML sign-in uses the browser's active Google session and shows no account chooser. Switch to your `@java.no` account first, or use a private window.
- **You sign in but `javabin-team-<team>` is missing.** Check that you are in `teams/<team>.yaml`. If you are, your Identity Center user was probably created after the last team sync: the 3-hourly provisioner run adds you to `team-<team>`. If it is still missing after that, ask a platform admin.

## CI/CD: GitHub OIDC

App repos authenticate to AWS in CI with GitHub's OIDC provider. No long-lived credentials or repository secrets.

- The platform creates two CI roles per registry team: `javabin-ci-team-<team>` (Terraform) and `javabin-ci-deploy-<team>` (image push, ECS deploy). Teams not in the registry have none.
- Only the platform's own workflows can assume the broker roles. A platform Lambda verifies the job's OIDC token and that the repo belongs to the GitHub team named in `app.yaml` `team:` before handing out short-lived team credentials.
- Team roles are limited by a permission boundary to resources prefixed `<team>-`.
- Terraform apply gets its credentials from the apply-gate Lambda, and only after the plan's LLM risk review (run by the platform's `plan-review` Lambda) passes. HIGH or FAILED reviews need a board-member override via `approve-override.yml`. See [apply gate](https://github.com/javaBin/platform/blob/main/docs/apply-gate.md).

## App Authentication: Cognito User Pools

| Pool | Purpose | Users |
|------|---------|-------|
| **Internal** | javaBin volunteers, admin tools | `@java.no` accounts via Google sign-in (OIDC) |
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

**Google sign-in.** The internal pool signs users in through Google OIDC (Cognito identity provider `Google`). Google shows its account chooser when the app (or the platform's ALB sign-in) asks for it, and its consent screen is Internal, so only `@java.no` accounts complete sign-in; Google shows anyone else an error page with an account switcher. Apps that pass `identity_provider` themselves should read `COGNITO_IDENTITY_PROVIDER` instead of hardcoding a name. App sign-in does not depend on Identity Center provisioning. See [cognito-google-setup](https://github.com/javaBin/platform/blob/main/docs/cognito-google-setup.md).

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
| AWS console / CLI | Identity Center, Google Workspace SAML + auto-provisioning (`javabin aws setup`) |
| CI/CD | GitHub OIDC → platform broker → team CI roles |
| App users (internal) | Cognito internal pool + Google OIDC |
| App users (external) | Cognito external pool |
| Legacy app auth | Auth0 (`login.javazone.no`) |
| Developer CLI | gh CLI + AWS SSO profiles |
