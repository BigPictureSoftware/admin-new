# Classic ASP Migration Guide

Last updated: July 25, 2026

This guide defines how legacy `.asp` admin tools move into the managed global
admin platform. It supersedes the earlier assumption that work remains an
unlinked WVBPS-only pilot.

Read with:

- [`admin-shell-platform.md`](admin-shell-platform.md)
- [`perl-eradication-plan.md`](perl-eradication-plan.md)
- [`security-quality-review-2026-07.md`](security-quality-review-2026-07.md)
- [`source-first-deployment-workflow.md`](source-first-deployment-workflow.md)

## Current direction

The project is no longer in stealth mode. WVBPS was the proving deployment, not
the permanent product location. Further development will move to a global admin
folder so the managed shell and rewritten tools can serve every client from one
maintained codebase.

The exact new physical and URL roots are not yet recorded in this repository.
Until they are selected:

- use `{GlobalManagedAdminRoot}` for the managed URL root;
- use `{GlobalManagedAdminPhysicalRoot}` for its physical folder;
- keep `E:\web\repos\admin-new` as the Git source of truth;
- treat the existing legacy `GLOBAL_6-next/admin` tree as read-only;
- do not infer that "move global" authorizes editing the existing legacy tree.

WVBPS paths and screenshots remain useful historical evidence, but they are no
longer the target architecture.

## Governing rule

> **If you have to touch a Classic ASP tool a lot, rebuild it in .NET.**

Do not invest new business logic, security mechanisms, database abstractions, or
service integrations in Classic ASP. Classic ASP is a temporary compatibility
surface, not the destination.

The amount of change is determined by responsibility, not line count. A change
is **small enough to preserve in ASP** only when all of the following are true:

- shell includes, title, route, or static asset paths are the only changes;
- the request and response contract stays identical;
- no authentication, authorization, database, filesystem, external-service, or
  business-rule behavior changes;
- no security defect requires restructuring;
- no ASP→Perl or ASP→unmanaged-service call is being added or materially changed;
- the tool has a documented .NET retirement path.

Rebuild in .NET when any of these is true:

- business logic or validation changes;
- a database query or transaction changes;
- an API/AJAX/form contract changes;
- output encoding, file handling, or security requires broad edits;
- the tool calls Perl for functional behavior that must be replaced;
- a new integration or endpoint is needed;
- tests cannot be added without extracting the logic;
- the ASP change would create new long-lived technical debt.

## Migration paths

### Path A — compatibility onboarding

Use only for a low-risk ASP tool that can run unchanged behind the managed
authentication, authorization, and shell.

Allowed edits:

- local/global managed shell include;
- page title;
- route registration;
- canonical ACL mapping;
- static include/asset path correction;
- an explicit compatibility comment and retirement record.

Do not refactor the tool while onboarding it. Characterize and test it, then
move it intact. If characterization exposes a high-risk defect, choose Path B.

### Path B — managed .NET rebuild

Use for any substantial change or any tool with functional Perl dependencies.
Rebuild as a managed page or zero-build Vue screen backed by narrow VB.NET JSON
handlers, following the Code Admin and Access Manager structure:

```text
App_Code/AdminShell/<Tool>/
  <Tool>Contracts.vb
  <Tool>Security.vb
  <Tool>Validation.vb
  <Tool>Repository.vb
  <Tool>Service.vb
  <Tool>ApiHandlers.vb
  <Tool>Page.vb                 # when server-side page authorization is needed

managed/<tool>/
  index.aspx
  api/*.ashx
  js/*.js
  <tool>.css
```

Keep handlers thin, enforce authorization again in the service layer, keep SQL
inside repositories, use typed commands/contracts, and share shell/API/dialog
code rather than copying it.

### Path C — retire

If usage evidence shows that a tool or action is no longer required, remove its
route and dependencies instead of rewriting it. Retirement still requires
owner approval, usage evidence, a rollback period, and confirmation that no
other ASP/Perl script invokes it.

## Per-tool workflow

### 1. Create an inventory record

Before editing, record:

- source path and public route;
- canonical ACL identity;
- menu/section registrations;
- users or roles with access;
- HTTP methods and accepted content types;
- query-string, form, cookie, header, and session inputs;
- response statuses, content types, bodies, redirects, downloads, and headers;
- database providers, tables, queries, procedures, and transaction boundaries;
- filesystem/share access;
- includes, COM objects, web services, AJAX targets, Perl calls, and scheduled
  jobs;
- side effects, logs, emails, cache/session writes, and downstream consumers;
- known defects and security findings;
- usage/owner evidence;
- selected migration path and reason.

Do not trust the entry script alone. Includes and server-side HTTP calls are
part of the implementation and contract.

### 2. Trace the complete dependency graph

Search the source and all includes for:

- `<!--#include ... -->`;
- `Server.CreateObject`;
- `ServerXMLHTTP`, `WinHttpRequest`, and `XMLHTTP`;
- `.pl`, `/cgi-bin/`, `ajax.asp`, and dynamically assembled URLs;
- `Application(...)`, `Session(...)`, and cookies;
- database connection helpers and raw `Connection.Execute`;
- filesystem objects, shares, uploads, and downloads;
- client JavaScript `fetch`, XHR, jQuery AJAX, forms, and inline-edit URLs.

Classify each dependency:

| Class | Migration treatment |
|-------|---------------------|
| Managed shell/auth | replace with direct managed integration |
| ASP business helper | port into the .NET service/repository |
| Perl business helper | port directly to .NET; do not add an ASP replacement |
| Shared database helper | replace with parameterized repository code |
| External web service | wrap behind a typed .NET client with timeout and policy |
| Static include/asset | self-host or move under the shared managed asset tree |
| Dead/unused | prove and retire |

The global legacy source was unavailable during the July 25 documentation pass.
When it is mapped, this inventory must be rerun against the complete global tree
before selecting or estimating tools. The current repository is not a complete
ASP→Perl dependency inventory.

#### What the tracked repository proves today

The four copied ASP tools do **not** call Perl through `includes/ssi.inc`.
Current `pilotFetch`/`pilotWrite` callers reach only managed
`authorize.ashx`/`chrome.ashx`; `pilot-bridge.asp` calls
`pilot-establish.ashx`. Do not describe the shell helper itself as a Perl
gateway.

The tracked runtime dependencies are:

| Dependency | Type | Migration implication |
|------------|------|-----------------------|
| Code Admin → `/admin/admin/cgi-bin/classadminO.pl?action=showUpdate&code_class=...&Submit=Modify` (`managed/code-admin/js/components/workspace.js:32`) | Live browser navigation to Perl business UI | Characterize and rewrite Class Admin or remove the link |
| `cgi-bin/accessadmin.pl`, `codeadmin*.pl`, `sectionadmin.pl`, `scriptadmin.pl`, `section_scriptadmin.pl` in config | Canonical ACL identity strings | Do not mistake for HTTP calls; decide when/if neutral capability ids replace them |
| Access Manager `checkRoute` | Dynamic server-side HEAD probe that may target a CGI path | Constrain or remove; see the security review |
| Unmapped DB menu `script_name` | Dynamic browser navigation that may remain a `.pl` route | Include DB menu rows in the global inventory |
| `sms_logs.asp` global connection-information request | External web service, not Perl | Replace with managed secret/config provider |

The coworker-reported ASP→Perl business calls may exist in global ASP files not
copied into this repository. They remain a required discovery item, not a
confirmed property of these four copies.

### 3. Capture the observable contract

For every action, save sanitized fixtures for:

- successful request and response;
- validation failure;
- unauthorized/forbidden request;
- missing and malformed values;
- duplicate/concurrency case;
- empty and large result sets;
- external dependency failure;
- rollback or partial-write case.

Record exact status, content type, encoding, body shape, redirect location,
download filename, cookie/session effects, and database effects. For HTML forms,
record field names, repeated fields, checkbox omissions, submit-button names,
and encoding. Browser-visible similarity is not enough.

### 4. Establish the security boundary

Every migrated tool must:

- authenticate through the managed session authority;
- authorize its canonical capability on every request;
- enforce finer-grained capabilities in the service layer where applicable;
- require CSRF protection on state changes;
- use parameterized SQL and allowlist dynamic identifiers;
- encode output for its actual HTML/attribute/URL/JavaScript context;
- validate paths before filesystem access;
- set timeouts and validate TLS for outbound calls;
- avoid returning secrets and provider exception details;
- write an audit event for administrative mutations.

Do not preserve an insecure behavior merely because it is part of the legacy
implementation. See "contract parity versus mandatory correction" in the Perl
plan.

### 5. Implement behind a compatibility route

Preserve the old public/form/API contract unless an approved security correction
requires a break. Keep a route-level rollback switch or deploy the managed
replacement under a parallel path first.

Route identity and implementation path are different concepts. The replacement
may continue authorizing against the legacy canonical ACL identity during
migration so existing grants do not have to change in the same release.

### 6. Test parity and corrections

Tests must cover:

- request parsing and validation;
- service authorization and business rules;
- repository queries and transactions;
- response status/body/header contract;
- browser workflow and accessibility;
- database before/after state;
- each intentionally corrected security behavior;
- regression fixtures from the legacy implementation.

Use differential tests where safe: replay the same read-only request against
legacy and managed implementations, normalize timestamps/ids, and compare.
Never duplicate a destructive production request merely to compare behavior.

### 7. Cut over and observe

1. deploy the managed implementation without changing the existing route;
2. run authenticated smoke and contract tests;
3. enable a small explicit user/client cohort when possible;
4. compare errors, latency, row counts, and audit events;
5. switch the route/menu to managed;
6. retain a bounded rollback period;
7. remove rollback only after usage and dependency logs are quiet.

### 8. Delete legacy code

A migration is not complete while the old script remains an undocumented
fallback. After the rollback period:

- remove old menu/route registrations;
- remove ASP/Perl entry points and unused includes;
- remove obsolete configuration and secrets;
- remove compatibility aliases after consumers migrate;
- update the inventory and eradication status;
- verify that repository and deployment searches find no remaining callers.

## Known repository examples

### Views

`views.asp` is not self-contained. It uses global includes and posts inline-edit
updates to `Application("ADMIN_URL") & "/admin/ajax.asp?action=updateView"`
(`views.asp:884`). Migrating the page requires characterizing that AJAX contract
and either preserving it temporarily or rebuilding it as a managed endpoint.

The file also contains substantial data manipulation and dynamic UI generation.
Under the new rule, any meaningful feature or database change should trigger a
.NET rebuild rather than further ASP investment.

### SMS Logs

`sms_logs.asp:20-40` synchronously obtains database connection credentials from
`https://ws.ebigpicture.com/GlobalDBConnectionInformation/`, parses them in ASP,
and stores the resulting connection string in application state. A .NET rebuild
should use deployment-managed secrets or a typed credential provider, not copy
this pattern. Output such as message and error text is rendered without
contextual encoding and must be corrected.

### SQL Logs

`sql_logs.asp` accepts a filename through the query string and reads log
content. Its rebuild must canonicalize the requested path, constrain it to an
approved root, reject traversal/reparse-point escapes, and HTML-encode log
content. File route and download behavior should otherwise remain compatible.

### Login Log

`loginlog.asp` should be characterized for filtering, date/time behavior,
pagination, details, and database provider assumptions. A read-only log viewer
is a good early .NET migration candidate after security controls are standardized.

### Shell HTTP calls

`topshell.asp` and `bottomshell.asp` use `includes/ssi.inc` to call managed
authorization/chrome handlers. That bridge is transitional. Once tools live
inside the global managed admin application, render/authorize directly rather
than retaining blocking loopback HTTP.

## Global relocation checklist

Before moving development out of WVBPS:

- [ ] select and document `{GlobalManagedAdminRoot}` and physical root;
- [ ] decide whether it is a new IIS application or part of an existing one;
- [ ] document .NET Framework version, `App_Code`/compiled-project model, app
      pool identity, bitness, handlers, session provider, and connection strings;
- [ ] replace WVBPS host, banner, DSN, organization, and path values with
      environment/client configuration;
- [ ] remove the `PilotUsers` rollout allowlist in favor of production ACLs;
- [ ] define how client context is selected and validated;
- [ ] define the global managed login/session authority;
- [ ] resolve the legacy bridge findings before broad access;
- [ ] copy no client secret or client-specific data into the global source;
- [ ] create global deployment, rollback, and health-check manifests;
- [ ] run a complete route/include/ASP→Perl dependency inventory once the
      legacy source is available;
- [ ] update docs and tests that assert `/dev/adminshell` or WVBPS-specific paths;
- [ ] preserve WVBPS as a test client only if its configuration is externalized.

### Global path coexistence and promotion

Selecting the target is an architecture decision, not a file copy. Document:

1. whether managed admin is a sibling application, a subfolder, or the eventual
   replacement for `/admin/admin`;
2. how legacy and managed `default.asp`, `topshell.asp`, login, session, and
   static assets coexist without name or application-pool collisions;
3. whether `PilotRootPath` and `GlobalAdminRootPath` can overlap, and how cookie
   paths behave if they do;
4. which parent `global.asa`, `Application(...)` values, connection strings,
   Redis configuration, assemblies, and IIS handlers the managed app receives;
5. how menu and login entry points are promoted from legacy to managed;
6. how old public routes redirect, alias, or remain dual-served during rollback;
7. how the hardcoded `/dev/adminshell` bridge paths are removed before cutover;
8. how WVBPS-specific route assertions and browser URLs become parameterized.

Promotion is complete only when the managed route is linked in the real global
menu/login flow, its canonical ACL works for normal users, and the legacy route
is either an explicit rollback alias or deleted. The old "unlinked pilot"
acceptance criterion no longer applies.

## Definition of done

A Classic ASP migration is complete when:

- the managed implementation owns all required behavior;
- contracts are tested and intentional differences are documented;
- security gates pass;
- database writes are transactional and audited;
- no ASP-to-Perl dependency remains for the migrated tool;
- route/menu/ACL behavior is correct;
- global configuration contains no WVBPS-specific assumptions;
- rollback has expired and the legacy implementation is deleted;
- docs and the Perl/ASP eradication inventory are updated.
