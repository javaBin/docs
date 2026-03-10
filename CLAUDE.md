# CLAUDE.md — Javabin Docs

Developer-facing documentation for the Javabin platform and JavaZone systems.
This is a standalone docs repo — no build system, just Markdown files.

## Files

| File | What |
|------|------|
| `README.md` | Index of all javaBin systems (platform, JavaZone apps, java.no) |
| `platform-overview.md` | Platform architecture, components, how apps get CI/CD |
| `getting-started.md` | How to register a team, add repos, get automated CI/CD |
| `auth.md` | Identity model: Identity Center, GitHub OIDC, Cognito pools |

## Key Facts

- Teams are the registration unit. Register via `teams/{name}.yaml` in `javaBin/registry`.
- There is no separate app registration. Apps just need `app.yaml` at repo root and membership in a GitHub team.
- Platform Terraform and reusable workflows live in `javaBin/platform`.

## Related

- [javaBin/platform](https://github.com/javaBin/platform) — infrastructure code
- [javaBin/registry](https://github.com/javaBin/registry) — team registration
