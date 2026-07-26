# Managed Admin Shell (`admin-new`)

Client-neutral managed admin platform: HTML5 login, unified chrome, VB.NET APIs,
Access Manager, Code Admin, compatibility-hosted Classic ASP tools, and a
temporary session bridge to legacy `/admin/admin` tools.

The WVBPS pilot proved the platform. The project is no longer in stealth mode;
further work is moving to a global managed-admin folder so Classic ASP and Perl
admin tools can be replaced for every client. The exact global URL/physical
root is still to be selected. Existing `Pilot*` code/config names remain for
compatibility until a deliberate rename.

Files under the existing `GLOBAL_6-next\admin` legacy tree remain read-only.
Do not infer a deployment target from the phrase "global admin"; select and
document the new managed location first.

See [`docs/managed-admin-shell-plan.md`](docs/managed-admin-shell-plan.md) for
the current implementation status, IIS setup, verification, and later waves.
Before continuing work, read [`docs/admin-shell-platform.md`](docs/admin-shell-platform.md)
and [`docs/agent-handoff.md`](docs/agent-handoff.md) for architecture, environment
boundaries, and the tool-migration checklist.

Plans and status:

- [`docs/admin-shell-platform.md`](docs/admin-shell-platform.md) — **central overview** (legacy vs pilot, auth bridge, relocatable config).
- [`docs/security-quality-review-2026-07.md`](docs/security-quality-review-2026-07.md) — full security, correctness, performance, and quality review.
- [`docs/legacy-auth-bridge-hardening-plan.md`](docs/legacy-auth-bridge-hardening-plan.md) — remove the legacy cookie impersonation/session trust weakness.
- [`docs/classic-asp-migration-guide.md`](docs/classic-asp-migration-guide.md) — compatibility onboarding versus .NET rebuild rules for `.asp` tools.
- [`docs/perl-eradication-plan.md`](docs/perl-eradication-plan.md) — contract-preserving Perl rewrites, mandatory corrections, cutover, and deletion.
- [`docs/shell-unification-plan.md`](docs/shell-unification-plan.md) — historical
  completed WVBPS shell-unification wave.
- [`docs/github-repo.md`](docs/github-repo.md) — GitHub repo layout and deploy notes.
- [`docs/source-first-deployment-workflow.md`](docs/source-first-deployment-workflow.md) — source-first validation, deployment, rollback, and commit workflow.
- [`docs/aspnet-web-site-vb48-workflow.md`](docs/aspnet-web-site-vb48-workflow.md) — reusable staff guide for source-deployed ASP.NET Web Sites using VB.NET and .NET Framework 4.8.
- [`AGENTS.md`](AGENTS.md) — short index for coding agents.

## Direction and engineering rule

- Preserve form, API, database, authorization, and side-effect contracts by
  default when replacing legacy code.
- Do not copy known security or integrity defects merely for parity.
- Add no new Perl.
- If a Classic ASP capability must be touched substantially, rebuild it in .NET.
- If ASP calls Perl for business functionality, implement that capability once
  in .NET; use a temporary ASP-compatible endpoint only until the caller is
  rebuilt.
- Remove old implementations after a bounded rollback period. A permanent
  fallback is not a completed migration.

## IIS layout (historical WVBPS proving deployment)

Values below document the proving deployment and current source assumptions.
They are not the future global target.

- Pilot tree lives under the client's front-end IIS app (example: `/dev/adminshell`).
- Managed endpoints inherit the parent .NET Framework 4.8 configuration.
- Application-root `App_Code` is the special source-compilation root. Shared admin-shell VB classes deploy under `App_Code/AdminShell/`; Code Admin-specific classes deploy under `App_Code/AdminShell/CodeAdmin/`. Under the current default compilation configuration, ordinary same-language nested folders participate in the generated `App_Code` assembly. Explicit `codeSubDirectories` entries would create separate compilation units.
- `managed\web.config` contains only pilot app settings; it must not contain
  application-level sections such as `authentication`, `compilation`, or
  `sessionState`.

Pilot entry point:

`https://dev.services.wvbps.wv.gov/dev/adminshell/managed/login.html`

Copied pilot tools:

| Pilot route | Canonical ACL identity |
|-------------|------------------------|
| `/dev/adminshell/views.asp` | `/admin/admin/views.asp` |
| `/dev/adminshell/loginlog.asp` | `/admin/admin/loginlog.asp` |
| `/dev/adminshell/sql_logs.asp` | `/admin/admin/sql_logs.asp` |
| `/dev/adminshell/sms_logs.asp` | `/admin/admin/sms_logs.asp` |
| `/dev/adminshell/managed/access-manager/index.aspx` | `/admin/admin/cgi-bin/accessadmin.pl` |
| `/dev/adminshell/managed/code-admin/index.aspx` | `/admin/admin/cgi-bin/codeadminO.pl` |

Access Manager SPA:

`https://dev.services.wvbps.wv.gov/dev/adminshell/managed/access-manager/index.aspx`

Code Admin (.NET rewrite of `codeadminO.pl`):

`https://dev.services.wvbps.wv.gov/dev/adminshell/managed/code-admin/index.aspx`

The Access Manager SPA uses document-relative JSON APIs under `managed/access-manager/api/`
and shared shell assets under `managed/shared/`. It now uses one section-centered
workspace: section names and assigned scripts are editable inline, script CRUD
is available from the section, and modal lookups replace raw script/principal
IDs. Principal-centric access search is launched from the same workspace.
The shell also renders the user's access-filtered sections and scripts in a
searchable, collapsible left menu following the legacy navigation hierarchy.
Perl admin tools currently remain at their canonical
`/admin/admin/cgi-bin/...` paths as rollback. The program goal is to remove all
of them; see the eradication plan.

Its seven production server files are under `App_Code/AdminShell/AccessManager/`:
`AccessManagerContracts.vb`, `AccessManagerSecurity.vb`,
`AccessManagerApiHandlers.vb`, `AccessManagerPage.vb`,
`AccessManagerRepository.vb`, `AccessManagerService.vb`, and
`AccessManagerValidation.vb`.

Route mappings, nav labels, and the default post-login route are configured in
`managed/web.config` (`PilotRoutes`, `PilotDefaultRoute`). Unknown routes are
denied. The HTML5 login still defaults to Views.

## Historical pilot isolation

- The managed login uses the separate `bp_admin_next` cookie.
- The cookie contains no password and does not satisfy legacy Perl auth.
- The login UI is semantic HTML5 and JavaScript backed by a VB.NET JSON API;
  it does not use Web Forms, server controls, postbacks, or view state.
- Only the configured host and `PilotUsers` allowlist can sign in.
- Each copied route is authorized against its own canonical
  `/admin/admin/...` ACL identity from `PilotRoutes`.
- Pilot shell navigation is generated from the same route configuration.
- The original WVBPS rollout was not linked from the existing menu. That stealth
  constraint is retired for future global development.

## Verification

The managed pages and application-root classes compile as part of the parent
.NET Framework 4.8 application.
`managed/App_Data/tests/PilotPolicyTests.vb` covers host/user gating, config-
driven route mapping (happy path, unknown route, malformed config, case-
insensitive matching), return URL validation, and hash comparison.
`managed/App_Data/tests/AccessManagerValidationTests.vb` and
`AccessManagerServiceTests.vb` cover Access Manager validation and service
rules with an in-memory repository fake.
`managed/App_Data/tests/AccessManagerStateTests.js` covers pure client reorder
helpers (`node AccessManagerStateTests.js` when Node is available).
Code Admin tests: `CodeAdminViewModelTests.js`, `CodeAdminWorkspaceUiTests.js`,
`CodeAdminValidationTests.vb`, `CodeAdminServiceTests.vb`, and
`Test-CodeAdminBrowser.ps1`. See [`docs/agent-handoff.md`](docs/agent-handoff.md)
for Code Admin status, Oracle connection requirements, and UI styling notes.
`managed/App_Data/tests/Test-PilotData.ps1` verifies the pilot user and
canonical Views ACL when run on a machine with the WVBPS ODBC DSN.
`managed/App_Data/tests/Test-AccessManagerData.ps1` verifies the pilot user
has the Access Manager grant capability ACL. This mapped-
drive workstation is not the IIS host; deployed behavior must be verified
through the remote development URL or on the actual server.
