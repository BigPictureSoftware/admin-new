# Shell Unification Plan

Last updated: July 25, 2026

> **Historical/completed WVBPS wave.** The unified shell remains the reference
> implementation, but future work moves to the global managed-admin location.
> Use [`classic-asp-migration-guide.md`](classic-asp-migration-guide.md) for new
> ASP onboarding/rebuild decisions.

Read with [`agent-handoff.md`](agent-handoff.md) and
[`managed-admin-shell-plan.md`](managed-admin-shell-plan.md).

## Problem

The pilot currently has **two shells**:

| Shell | Used by | Left nav | Top tabs | Chrome source |
|-------|---------|----------|----------|---------------|
| **New** | Access Manager SPA | ACL-filtered section accordion (`shell.js`) | JS `renderNav` from session | `managed/access-manager/index.aspx` + `managed/shared/shell.{css,js}` |
| **Legacy** | `views.asp`, `loginlog.asp`, `sql_logs.asp`, `sms_logs.asp` | Stub “Configuration” link only | Server-rendered `PilotRoutes` tabs | `topshell.asp` → `chrome.ashx` → `PilotShell.vb` |

Access Manager already shows the desired UX: modern header, pilot tool tabs, and
the legacy-style section/script left menu. Classic ASP tools still use the old
Bootstrap banner and table layout.

**Goal:** one shared shell for every pilot route. Tool business logic stays
unchanged; only the chrome wrapper changes.

## Target layout

```text
┌─────────────────────────────────────────────────────────┐
│  Header (brand, user, sign out)                         │
│  Top tabs: Views | Login Log | SQL Logs | SMS | Access  │
├──────────┬──────────────────────────────────────────────┤
│ Section  │  Tool content                                │
│ menu     │  (views.asp body, Access Manager SPA, logs)  │
│ (ACL)    │                                              │
└──────────┴──────────────────────────────────────────────┘
```

## What does NOT change

- `views.asp` (and other copied tools) — no rewrite of CRUD, grids, or Ajax.
  Views still calls global `/admin/admin/...` endpoints.
- Perl rollback paths under `/admin/admin/cgi-bin/...`.
- `PilotRoutes` ACL model — each pilot path still maps to a canonical identity.
- Access Manager SPA internals — it already uses the target shell.

## Implementation steps

### Step 1 — Pilot-wide session API

**Today:** `menuSections` is returned only from
`managed/access-manager/api/session.ashx`, gated on Access Manager capability
(`CanUseAccessManager`).

**Change:** add `managed/api/session.ashx` (or equivalent pilot-wide handler)
that returns, for any authenticated pilot user:

```json
{
  "userName": "...",
  "menuSections": [ ... ],
  "paths": {
    "pilotRoot": "/dev/adminshell",
    "globalAdminRoot": "/admin/admin",
    "logoutUrl": "...",
    "routes": [ { "path": "...", "label": "..." } ]
  }
}
```

Reuse `PilotRepository.ListMenuSections(memberId)` — same query Access Manager
already uses (`PilotSecurity.vb`).

Access Manager may keep its own `session.ashx` for capabilities and CSRF, or
call the shared menu builder internally.

### Step 2 — Extend `PilotShell.vb` chrome (one place → all ASP tools)

**Today:** `RenderHeader` emits Bootstrap banner, stub left column, and
horizontal `#sectionMenu` tabs. `RenderFooter` closes `.mainbody` wrappers.

**Change `App_Code/AdminShell/PilotShell.vb`:**

1. Link `managed/shared/shell.css` (keep `bpstyles.css` for legacy tool grids).
2. Replace `#col-left` stub with `<aside id="adminMenu" class="admin-menu">`.
3. Wrap tool content in `<main class="shell-main">` inside `.admin-layout`.
4. Include `managed/shared/{api-client,session,shell}.js`.
5. Inline bootstrap on load:
   - `PilotSession.load()` against the pilot-wide session API
   - `PilotShell.renderNav(#pilotToolNav, routes, currentPath)`
   - `PilotShell.renderSectionMenu(#adminMenu, menuSections, currentPath)`

**No per-tool ASP edits** if `topshell.asp` / `bottomshell.asp` stay the entry
point — all four log/view tools pick up the new chrome automatically.

Match Access Manager header markup where practical (`shell-header`,
`#pilotToolNav`, `#logoutButton`).

### Step 3 — Remap menu links to pilot URLs

**Today:** `ListMenuSections` returns DB `script_name` values such as
`/admin/admin/views.asp`. The left menu in Access Manager can link users to
**legacy** admin instead of the pilot copy.

**Change:** when building menu items, if a script's canonical path matches a
`PilotRoutes` entry, emit the **pilot path** instead:

```text
/admin/admin/views.asp  →  /dev/adminshell/views.asp
```

Add reverse lookup in `PilotPolicy.vb` (e.g. `TryResolvePilotPath`) or remap in
the menu serializer. `PilotRoutes` in `managed/web.config` is the source of
truth.

### Step 4 — Adjust `RenderFooter`

Close the new `.admin-layout` / `.shell-main` wrappers cleanly. Verify
`bottomshell.asp` still pairs with the updated header.

### Step 5 — Tests and remote verification

**Local / CI:**

- Extend `AccessManagerWorkspaceUiTests.js` or add `PilotShellUiTests.js` for
  chrome markup expectations.
- Add `PilotPolicy` test for canonical → pilot path reverse lookup.
- VB compile check on `App_Code/AdminShell`.

**Remote (IIS):**

1. Open Views from the section menu — confirm pilot URL, not legacy.
2. Confirm same left nav + top tabs as Access Manager (collapse, search,
   accordion).
3. Confirm Views CRUD and Ajax still work.
4. Repeat spot-check for Login Log, SQL Logs, SMS Logs.
5. Confirm user without a route ACL still gets 403 on direct navigation.

## File touch list

| File | Change |
|------|--------|
| `managed/api/session.ashx` | New pilot-wide session handler |
| `App_Code/AdminShell/PilotShell.vb` | New chrome layout + script includes |
| `App_Code/AdminShell/PilotPolicy.vb` | Optional reverse route lookup |
| `App_Code/AdminShell/PilotJsonApi.vb` | Shared menu serialization helper |
| `managed/shared/session.js` | Point at pilot session path for ASP pages |
| `managed/App_Data/tests/*` | Policy + UI string tests |

## Optional later

- Retire Bootstrap banner in favor of Access Manager `shell-header` styling.
- Apply the same shell to any future copied ASP tools (add route only).
- Combine `authorize.ashx` + `chrome.ashx` into one round trip (measure first).

## Status

- [x] Step 1 — pilot-wide session API (`managed/api/session.ashx`)
- [x] Step 2 — `PilotShell.vb` chrome
- [x] Step 3 — menu path remap (`PilotPolicy.TryResolvePilotPathByCanonical`)
- [x] Step 4 — footer / layout close
- [ ] Step 5 — tests and remote verification
