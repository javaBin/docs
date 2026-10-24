# CLAUDE.md — Javabin Docs

Developer-facing documentation for the Javabin platform and JavaZone systems.
This is a standalone docs repo — no build system, just Markdown files.

## Files

| File | What |
|------|------|
| `README.md` | Index of all javaBin systems (platform, JavaZone apps, java.no) |
| `platform-overview.md` | Platform architecture, components, how apps get CI/CD |
| `getting-started.md` | Register a team, add repo, app.yaml + deploy.yml, AWS access, secrets |
| `auth.md` | Identity model: Identity Center, GitHub OIDC, Cognito app auth |

## Key Facts

- Teams are the registration unit. Register via `teams/{name}.yaml` in `javaBin/registry`.
- There is no separate app registration. An app needs `app.yaml` at repo root, `.github/workflows/deploy.yml` calling the platform's `javabin.yml`, and the repo in the GitHub team named in `app.yaml` `team:`.
- Platform Terraform and reusable workflows live in `javaBin/platform`. Its `docs/` (app-yaml-reference, reusable-workflows, lambda-functions, apply-gate) are the source of truth; keep these docs short and link there.
- Developer CLI: `javaBin/javabin-cli` (`brew install javaBin/tap/javabin`).
- Cupcake (`javaBin/frosting` + `javaBin/cupcake`) is the reference app on the platform.

## Related

- [javaBin/platform](https://github.com/javaBin/platform) — infrastructure code
- [javaBin/registry](https://github.com/javaBin/registry) — team registration
