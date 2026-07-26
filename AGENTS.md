# Agent guide — admin-new

**Product:** managed admin shell (client-neutral). **Repo:** `admin-new` on GitHub.

Read **[`docs/admin-shell-platform.md`](docs/admin-shell-platform.md)** first.
The WVBPS pilot proved the platform; development is now transitioning to a
global managed-admin location. Legacy `Pilot*` names remain in code until a
separate compatibility-safe rename.

| Doc | Use when |
|-----|----------|
| [`docs/agent-handoff.md`](docs/agent-handoff.md) | Boundaries, verification, troubleshooting, **Code Admin** status |
| [`docs/security-quality-review-2026-07.md`](docs/security-quality-review-2026-07.md) | Full security, correctness, performance, and maintainability review |
| [`docs/legacy-auth-bridge-hardening-plan.md`](docs/legacy-auth-bridge-hardening-plan.md) | Changing pilot↔legacy auth, cookies, or session trust |
| [`docs/classic-asp-migration-guide.md`](docs/classic-asp-migration-guide.md) | Moving Classic ASP tools into the managed global admin |
| [`docs/perl-eradication-plan.md`](docs/perl-eradication-plan.md) | Contract-preserving Perl rewrites, correction gates, cutover, and deletion |
| [`docs/vue-managed-screens.md`](docs/vue-managed-screens.md) | Building or changing zero-build Vue managed screens; Code Admin reference |
| [`docs/source-first-deployment-workflow.md`](docs/source-first-deployment-workflow.md) | Edit, validate, deploy, hash-check, browser-test, roll back, commit, and push workflow |
| [`docs/aspnet-web-site-vb48-workflow.md`](docs/aspnet-web-site-vb48-workflow.md) | Repo-agnostic staff guide for VB.NET 4.8 Web Sites, App_Code, code-behind, local Git, and workstation compilation |
| [`.cursor/rules/browser-e2e.mdc`](.cursor/rules/browser-e2e.mdc) | Browser MCP protocol for pilot UI verification |
| [`docs/github-repo.md`](docs/github-repo.md) | Git clone, sync, secrets |
| [`docs/legacy-credential-encoder.md`](docs/legacy-credential-encoder.md) | Legacy cookie bridge |
| [`.cursor/skills/commit-mapped-drive/SKILL.md`](.cursor/skills/commit-mapped-drive/SKILL.md) | Commits from mapped-drive workspaces |

**Hard rules:** the existing `GLOBAL_6-next/admin` legacy tree remains read-only
until an explicit new global managed target is selected. Commit from
`E:\web\repos\admin-new`. Never commit `managed/web.config.local`. No new Perl;
if a Classic ASP change is substantial, rebuild the capability in .NET.
