# Perl Eradication Plan

Last updated: July 25, 2026

## Goal

Remove all Perl/CGI code from the admin platform without coupling the work to a
large redesign. The default migration unit is one required `.pl` entry point or
one cohesive shared Perl capability at a time.

The default strategy is **contract-preserving 1:1 replacement**:

- same form field names and submission encoding;
- same HTTP method, route/alias, status, response body/HTML/API shape, redirects,
  cookies, and headers;
- same database provider, queries/procedures, parameter semantics, transaction
  boundaries, ordering, null handling, and side effects;
- same authorization capability and client-context behavior;
- same downstream API and integration contracts;
- same user-visible workflow unless an approved correction is required.

This reduces migration risk. It does **not** authorize copying a vulnerability
or an obsolete implementation pattern. Each migration has an explicit
"mandatory correction" gate for defects that materially improve security,
integrity, operability, or the ability to retire Perl.

Read with:

- [`classic-asp-migration-guide.md`](classic-asp-migration-guide.md)
- [`security-quality-review-2026-07.md`](security-quality-review-2026-07.md)
- [`legacy-auth-bridge-hardening-plan.md`](legacy-auth-bridge-hardening-plan.md)
- [`vue-managed-screens.md`](vue-managed-screens.md)

## Non-negotiable direction

1. **No new Perl.**
2. **No substantial new Classic ASP.** If a migration requires meaningful
   changes to an ASP caller, rebuild that caller/capability in .NET.
3. **Do not move a Perl dependency sideways into ASP.** An ASP tool that calls a
   hairy Perl method must receive a .NET replacement for that method.
4. **Preserve external contracts by default, not internal structure.** Rewrite
   the implementation cleanly behind the old contract.
5. **Delete migrated Perl after a bounded rollback period.** A permanent
   fallback is not eradication.

## What 1:1 means

Contract parity is assessed at six boundaries.

### HTTP and form contract

- route and accepted method;
- query/form parameter names, repetition, omission, and encoding;
- content type and character encoding;
- response status, headers, content type, body, redirect, and download name;
- cookie and session reads/writes;
- error response behavior relied upon by callers.

### API contract

- field names, casing, types, null/empty behavior, arrays/maps, ordering;
- success and error envelopes;
- pagination/filter semantics;
- timeout and retry expectations;
- idempotency and duplicate-request behavior.

### Database contract

- provider and connection identity;
- tables/views/procedures and parameter order/types;
- joins, predicates, collation/case behavior, and ordering;
- transaction boundaries and lock/concurrency behavior;
- generated ids, audit columns, timestamps, and affected-row expectations;
- commits, rollbacks, triggers, and downstream database side effects.

The migration should use parameterized commands even when legacy SQL is
concatenated. "Same DB calls" means equivalent database behavior, not copying
unsafe string construction.

### Authorization and client-context contract

- canonical ACL identity;
- direct/group/section access semantics;
- organization/client scoping;
- record/object scope;
- read versus mutation capabilities;
- denial status and no-side-effect guarantee.

Preserve legitimate access. Do not preserve an authentication or authorization
bypass.

### Side-effect contract

- files, emails, SMS, jobs, cache, Redis/session, logs, external services;
- ordering and all-or-nothing behavior;
- retry and duplicate effects;
- observability and operator workflow.

### UI workflow contract

- action names and navigation;
- form defaults and validation;
- result ordering and labels;
- browser history, back behavior, and focus/accessibility;
- print/download behavior.

Pixel-for-pixel parity is not required. Functional and data parity is.

## Mandatory correction gate

For each script, create a short decision record with three sections:

1. **Preserved behavior** — contracts intentionally kept 1:1.
2. **Mandatory corrections** — changes included in the migration.
3. **Deferred redesign** — improvements deliberately postponed.

A correction is mandatory in the same migration when the legacy behavior has
any of these properties:

### Security

- authentication or authorization bypass;
- impersonation, session fixation, non-revocable bearer credential;
- SQL/command/template injection;
- path traversal or arbitrary file access;
- server-side request forgery;
- stored/reflected DOM or server-side XSS;
- CSRF on an administrative mutation;
- insecure deserialization;
- recoverable password or secret returned to a browser;
- disabled TLS verification or plaintext credential transport;
- dangerous dynamic code loading/evaluation;
- unrestricted upload or executable-file write.

### Data integrity

- partial multi-statement writes that need a transaction;
- cross-client data access;
- missing object ownership/scope check;
- duplicate/lost updates caused by a known race;
- destructive operation without target confirmation or affected-row check;
- silent error swallowing that can report success after failure.

### Operational necessity

- dependency unavailable in the global managed environment;
- behavior cannot be tested or observed sufficiently to cut over safely;
- hardcoded client path/credential prevents global operation;
- response contract is ambiguous enough that preserving it would perpetuate
  caller breakage;
- copying the implementation would require new Perl or substantial ASP.

Do **not** combine unrelated UI redesign, schema redesign, or business-rule
changes with a parity migration. The drastic improvement must directly remove a
material risk or migration blocker. Everything else gets a follow-up issue.

When a mandatory correction changes a contract:

- document the old and new behavior;
- identify every caller;
- add compatibility handling where safe;
- obtain product/security owner approval;
- test the corrected denial/error behavior;
- communicate it in release notes.

## Target implementation pattern

Use the existing managed rewrites as references, not templates to copy blindly.

### Code Admin precedent

Code Admin replaced `codeadminO.pl`/`codeadmin.pl` with:

- typed contracts and validation;
- a service layer for business and authorization rules;
- an Oracle/OleDb repository preserving legacy data behavior;
- narrow `.ashx` APIs and a managed page;
- a zero-build Vue UI;
- the legacy canonical ACL identity during transition.

Its lessons include what to improve in future migrations: fail closed on schema
capability probes, use per-domain authorization instead of one broad route ACL,
push paging to SQL, and never expose credential-like option fields.

### Access Manager precedent

Access Manager combined the responsibilities of
`sectionadmin.pl`, `scriptadmin.pl`, `section_scriptadmin.pl`, and
`accessadmin.pl` into a cohesive managed domain. This is appropriate when
multiple Perl entry points are separate views over one data model and workflow.

Its lessons: keep granular capabilities, but add object scope when delegation
requires it; enforce optimistic concurrency consistently; avoid N+1 label
lookups; and audit every grant mutation.

### Standard shape

```text
App_Code/AdminShell/<Domain>/
  <Domain>Contracts.vb
  <Domain>Security.vb
  <Domain>Validation.vb
  <Domain>Repository.vb
  <Domain>Service.vb
  <Domain>ApiHandlers.vb
  <Domain>Page.vb

managed/<domain>/
  index.aspx
  api/*.ashx
  js/*.js
  <domain>.css

managed/App_Data/tests/
  <Domain>ValidationTests.vb
  <Domain>ServiceTests.vb
  <Domain>UiTests.js
  <Domain>ContractFixtures/
```

If the global relocation adopts a compiled application instead of source
`App_Code`, preserve the same logical layers in projects/libraries rather than
retaining the physical layout.

## Program phases

### Phase 0 — platform prerequisites

Complete once for the program:

- harden or disable the legacy auth bridge;
- select the global managed IIS/application layout;
- define client-context resolution and global authorization;
- add standard request correlation, audit logging, error mapping, CSRF, security
  headers, database helpers, and outbound HTTP policy;
- establish a repeatable VB/JS build and test command plus CI;
- define route aliases and rollback flags;
- define sanitized contract-fixture storage;
- create the migration inventory schema below.

### Phase 1 — full discovery

Inventory every:

- `.pl`, `.cgi`, Perl module/library, and scheduled Perl job;
- route/menu/ACL registration;
- Classic ASP, JavaScript, form, API, and external caller;
- `require`/`use` dependency;
- database and filesystem dependency;
- server-side HTTP call from ASP to Perl;
- shared method used by multiple scripts;
- secret/configuration dependency.

The global legacy source was not available on the machine during this plan's
creation. The current repository exposes known canonical routes but **cannot**
prove the complete Perl inventory. Rerun discovery against the mapped global
tree before estimates or eradication status are declared complete.

Recommended inventory fields:

| Field | Purpose |
|-------|---------|
| Entry point / shared module | Migration unit |
| Route and aliases | External contract |
| Canonical ACL | Authorization parity |
| Callers | Cutover dependency graph |
| Client/org scope | Globalization requirements |
| DB provider/tables/procedures | Data parity and test setup |
| Side effects | Transaction/idempotency design |
| ASP→Perl callers | Must become direct .NET capability |
| Usage/owner | Required, retire, or unknown |
| Risk findings | Mandatory correction gate |
| Target domain | New managed owner |
| Wave/status | Program tracking |

### Phase 2 — prioritize by dependency graph and risk

Do not migrate alphabetically.

1. shared authentication/session and configuration dependencies;
2. shared Perl libraries with many callers, behind .NET facades;
3. read-only, high-use tools that prove the migration factory;
4. cohesive domains such as Access Manager;
5. write workflows with well-characterized transactions;
6. file/process/external-integration scripts;
7. low-use scripts that may be retired;
8. final compatibility and login/bridge code.

Prefer leaf callers only when their Perl dependency is unique. If ten ASP tools
call one Perl helper, migrate the shared capability first and switch callers in
controlled waves.

### Phase 3 — characterize one migration unit

For each entry point or domain:

1. freeze unrelated feature work;
2. trace all entry points, includes, modules, and callers;
3. enumerate contract actions;
4. capture sanitized request/response/database fixtures;
5. map authorization and client scope;
6. document side effects and transaction boundaries;
7. run the mandatory correction gate;
8. choose 1:1 entry-point replacement or cohesive domain consolidation;
9. estimate tests, cutover, and deletion together.

No implementation begins while material callers or side effects remain
unknown.

### Phase 4 — implement parity behind a seam

- keep old routes through rewrite/alias when practical;
- put request translation at the handler boundary;
- use typed internal contracts even if external field names are awkward;
- implement business rules once in the service layer;
- use parameterized repository commands;
- preserve provider-specific behavior deliberately;
- make multi-step writes atomic;
- wrap external services behind typed clients;
- add correlation and audit records;
- implement approved mandatory corrections;
- do not expand scope into unrelated redesign.

For an ASP→Perl business call, the target sequence is:

```text
Classic ASP caller -> Perl helper
                 becomes
Classic ASP caller -> managed .NET endpoint -> .NET service/repository
                 then
managed .NET UI/page -> same .NET service/repository
```

The ASP bridge is temporary. Do not duplicate the migrated logic in ASP.

### Phase 5 — parity verification

Use three layers:

#### Characterization tests

Run against legacy to lock observable behavior before rewrite. Store sanitized
fixtures and expected database effects.

#### Managed unit/integration tests

Test validation, authorization, business rules, repository mapping,
transactions, external-client behavior, and error mapping. Include every
mandatory correction.

#### Differential tests

For safe read-only actions, submit equivalent requests to legacy and managed
implementations and compare normalized results. For writes, use isolated test
data or compare sequentially with a reset; never double-submit destructive
production actions.

Compare:

- status, headers, content type, redirect;
- normalized body/JSON/HTML semantics;
- rows selected/changed;
- generated audit/session/cache/file effects;
- authorization outcomes;
- latency and query count.

Every difference is classified as:

- expected correction;
- harmless presentation/serialization difference;
- defect blocking cutover.

### Phase 6 — controlled cutover

1. deploy managed code dark;
2. run health and authenticated contract checks;
3. shadow read-only traffic when feasible without exposing data;
4. enable an explicit user/client cohort;
5. switch route/menu/API callers behind a flag or alias;
6. monitor errors, denials, DB effects, latency, and audit events;
7. retain rollback for a defined short period;
8. prohibit feature changes to the retired Perl implementation.

Rollback returns routing to Perl; it must not require reverting data schema or
managed code. Any migration requiring irreversible schema change needs a
separate migration/rollback design and is not a simple 1:1 wave.

### Phase 7 — delete and prove eradication

After the observation window:

- delete migrated `.pl` entry points and unreferenced modules;
- remove ASP/JS/API callers and compatibility aliases;
- remove menu/ACL registrations that exist only for deleted paths, or remap
  canonical identity deliberately;
- remove Perl runtime/configuration/secrets when no users remain;
- search deployment and repository for route, filename, module, and method
  references;
- run the full contract and browser suites;
- update inventory status with deletion commit and deployment evidence.

The program is complete only when production no longer requires a Perl
interpreter or Perl-owned scheduled job.

## ASP-to-Perl dependencies

The coworker-reported pattern — ASP tools making server-side calls to Perl for
business functionality — is a first-class migration dependency, not an edge
case.

The tracked repository does not contain such a call: its Classic ASP
`ssi.inc` callers reach managed `.ashx` handlers only. The only confirmed live
business hop to Perl in this repository is Code Admin's browser link to
`classadminO.pl`. The reported ASP→Perl calls therefore belong to global source
that was unavailable during this documentation pass and must be inventoried
after it is mapped.

For each call, capture:

- ASP caller and action;
- exact Perl target (including dynamic URL construction);
- method, encoding, parameters, cookies/session, timeout, and TLS behavior;
- response status/body/headers parsed by ASP;
- database/side effects caused by Perl;
- retry and error behavior;
- other callers of the same Perl method.

Migration rules:

1. implement the capability once in a .NET service;
2. expose a narrow compatibility endpoint only while ASP callers remain;
3. authorize the endpoint independently — do not trust that the ASP caller
   already authorized;
4. require CSRF for browser-originated state changes;
5. preserve the ASP-parsed response contract until the caller is rebuilt;
6. add contract tests on both sides;
7. remove the endpoint or simplify it when the last ASP caller migrates.

Do not add new Classic ASP implementation as an intermediate step. If the
existing ASP caller itself needs substantial changes to consume the .NET
capability, rebuild the caller under the managed shell.

## Known migration units from this repository

| Legacy Perl identity | Managed status | Notes |
|----------------------|----------------|-------|
| `codeadminO.pl` / `codeadmin.pl` | Code Admin rewrite exists | Preserve as reference; fix fail-open class probe and broad class authorization before global rollout. |
| `accessadmin.pl` | Access Manager rewrite exists | Consolidated with related section/script/membership workflows. |
| `sectionadmin.pl` | Access Manager rewrite exists | Capability retained separately. |
| `scriptadmin.pl` | Access Manager rewrite exists | `checkRoute` probing needs hardening. |
| `section_scriptadmin.pl` | Access Manager rewrite exists | Add complete concurrency coverage. |
| `classadminO.pl` | Not migrated | Code Admin UI still navigates by GET to `action=showUpdate&code_class=<selected>&Submit=Modify` from `managed/code-admin/js/components/workspace.js:32`; response/auth/error contract is not documented here. |
| `login.pl` | Legacy bridge dependency | Remove through auth-bridge sunset; do not reproduce its cookie trust model. |

This table is not a complete global inventory.

## Per-script migration checklist

### Discovery

- [ ] entry point, modules, includes, callers, aliases recorded
- [ ] route/menu/canonical ACL recorded
- [ ] query/form/API/session/cookie contract recorded
- [ ] DB queries, provider, transactions, and side effects recorded
- [ ] ASP→Perl and Perl→external dependencies recorded
- [ ] usage and owner confirmed

### Decision

- [ ] required versus retire decided
- [ ] 1:1 entry point versus cohesive domain decided
- [ ] preserved behavior listed
- [ ] mandatory corrections approved
- [ ] deferred redesign listed

### Implementation

- [ ] managed handler/page and service authorization
- [ ] typed validation/contracts
- [ ] parameterized repository and atomic writes
- [ ] external clients with TLS/timeouts
- [ ] audit and correlation
- [ ] no secrets in response/source/logs
- [ ] temporary ASP compatibility endpoint only if required

### Verification

- [ ] characterization fixtures
- [ ] unit/service/repository tests
- [ ] authorization/CSRF/security tests
- [ ] differential parity result
- [ ] database/side-effect comparison
- [ ] browser/accessibility verification
- [ ] rollback tested

### Completion

- [ ] callers switched
- [ ] observation window passed
- [ ] Perl entry/module deleted
- [ ] ASP compatibility call deleted
- [ ] config/secret/runtime dependency removed
- [ ] inventory and docs updated

## Program metrics

Track eradication by dependency, not only file count:

- Perl entry points remaining;
- shared Perl modules with live callers;
- ASP→Perl calls remaining;
- JS/form/API callers of Perl remaining;
- scheduled Perl jobs remaining;
- routes still served by Perl;
- deployments requiring Perl runtime;
- migrations in characterization, parity, cohort, rollback, and deleted states;
- mandatory security corrections open.

## Immediate next steps

1. Map the read-only global legacy source on this machine.
2. Run the complete Perl/ASP dependency inventory.
3. Choose and document the new global managed physical/URL roots.
4. Resolve the critical authentication bridge findings.
5. Fix global-rollout blockers in the two completed rewrites.
6. Select one read-only ASP tool and one small Perl entry point as migration
   factory exercises.
7. Migrate `classadminO.pl` or remove the remaining Code Admin link to it.
8. Establish CI and contract fixtures before parallel migration waves begin.

## Status

- [x] Migration principles and gates defined
- [x] Existing Code Admin and Access Manager precedents recorded
- [ ] Complete global Perl/ASP dependency inventory
- [ ] Global managed target root selected
- [ ] Platform prerequisites complete
- [ ] First new parity migration selected
- [ ] All ASP→Perl business calls removed
- [ ] Perl runtime removed from production
