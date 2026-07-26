# GitHub Repository

Last updated: July 25, 2026

## Remote

- **URL:** https://github.com/bigpicturesoftware/admin-new
- **Default branch:** `main`
- **Organization:** `bigpicturesoftware`

## Local working copy (git)

Use a **local disk** clone — not the mapped drive:

```text
E:\web\repos\admin-new
```

Clone or pull here, commit, and push. Sync changed files to the IIS mapped drive
for deployment.

```bash
git clone https://github.com/bigpicturesoftware/admin-new.git E:/web/repos/admin-new
```

## What is versioned

The repository contains:

1. **Pilot tree** — Classic ASP tools, managed endpoints, Access Manager SPA,
   shared shell assets, tests, and docs (synced to each client's IIS path).
2. **`App_Code/AdminShell/`** — shared admin-shell VB.NET classes, with
   Code Admin-specific classes in `App_Code/AdminShell/CodeAdmin/`. On IIS these
   deploy under application-root `App_Code\AdminShell`. Application-root
   `App_Code` is the special compilation root: ordinary same-language nested
   folders participate in its generated assembly under the current default
   configuration; explicit `codeSubDirectories` entries create separate
   compilation units.
3. **`App_Code/RedisService.vb` and `App_Code/RedisSession.vb`** — shared
   StackExchange.Redis helpers (DAPE-derived, CacheManager-compatible). On IIS
   these deploy to the application-root `App_Code\` folder beside any existing
   site services such as `FormSessionService.vb`.

## Live vs repo layout

| Environment | Pilot files | VB.NET classes |
|-------------|-------------|----------------|
| **IIS (live)** | `www/html/dev/adminshell/` | `www/html/App_Code/AdminShell/` (with Code Admin in `CodeAdmin/`) plus `www/html/App_Code/RedisService.vb` and `RedisSession.vb` |
| **Git repo** | repo root | `App_Code/AdminShell/` (with Code Admin in `CodeAdmin/`) plus `App_Code/RedisService.vb` and `RedisSession.vb` |

**Workflow:** edit and validate in `E:\web\repos\admin-new` → back up and
copy/sync the coordinated change to mapped deployment paths → verify hashes and
remote IIS behavior → commit/push when requested. See
[`source-first-deployment-workflow.md`](source-first-deployment-workflow.md).

Do **not** keep the git working tree on `A:\` (network mapped drive). Git is
slow and requires `safe.directory` workarounds there.

## Secrets

Do not commit:

- Production credentials or tokens
- `managed/web.config.local` (gitignored; holds `PilotMembershipEncryptionKey`)
- `.env` files

`managed/web.config` contains pilot **appSettings** only. Secrets load from
`web.config.local` via `<appSettings file="web.config.local">`. Commit
`web.config.local.example` with empty placeholders only.

Git commits: see [`.cursor/skills/commit-mapped-drive/SKILL.md`](../.cursor/skills/commit-mapped-drive/SKILL.md)
and [`admin-shell-platform.md`](admin-shell-platform.md).

## Agent reminder

**Put plans in `docs/`** before or as part of implementation work. Link new plan
documents from `agent-handoff.md` and update status checklists when steps
complete.
