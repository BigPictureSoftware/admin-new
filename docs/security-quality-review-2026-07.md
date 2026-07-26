# Admin Shell Security and Engineering Review

Last updated: July 25, 2026

This is a source review of the managed admin shell, its legacy authentication
bridge, Access Manager, Code Admin, copied Classic ASP tools, shared JavaScript,
and data-access layers. It records findings; implementation sequencing for the
highest-risk authentication issue is in
[`legacy-auth-bridge-hardening-plan.md`](legacy-auth-bridge-hardening-plan.md).

## Executive summary

The migration from Perl to VB.NET materially improved the codebase:

- database values are parameterized throughout the new Access Manager and Code
  Admin repositories;
- state-changing managed APIs require CSRF tokens;
- authentication and authorization are centralized and rechecked per request;
- the new interfaces avoid the most common DOM-XSS patterns;
- destructive multi-statement repository operations generally use
  transactions;
- the legacy credential codec is isolated behind a replaceable interface.

The largest remaining risk is architectural rather than language-specific. The
managed shell accepts a legacy encrypted `username` cookie as proof of identity.
That preserves the old impersonation weakness: captured ciphertext is a
non-revocable bearer credential, and `authenticated=true` is attacker-supplied.
The rewrite reproduced the legacy trust contract instead of replacing it.

The repository is not ready to be treated as a finished security boundary until
the legacy bridge is hardened or disabled. Several additional high-priority
issues should be fixed before broad rollout: session fixation, disabled TLS
verification on a trusted server-to-server call, an open redirect, a
session-cookie-resettable login throttle, Code Admin's fail-open editable-class
probe, and plaintext PayPal credentials exposed through Code Admin.

## Scope and limits

Reviewed source:

- `App_Code/AdminShell/` and its Access Manager and Code Admin modules;
- `managed/` handlers, pages, shared browser code, and tests;
- copied Classic ASP tools in the repository;
- `global-bridge/pilot-bridge.asp`;
- `RedisService.vb` and `RedisSession.vb`;
- current project documentation and configuration templates.

This review did not include:

- a penetration test against IIS;
- database schema, indexes, constraints, or grants;
- parent-application `web.config`, IIS modules, headers, or network controls;
- the complete global Perl/Classic ASP source tree;
- dependency vulnerability scanning;
- verification of secrets or connection security in deployed configuration.

Severity describes plausible impact in an admin environment, not certainty of
exploitability in a specific deployment.

## Security findings

### Critical — legacy `username` ciphertext is a bearer credential

`PilotAuth.TryGetCurrentUser` falls back to legacy authentication whenever the
managed Forms Authentication ticket is missing, invalid, or expired
(`PilotSecurity.vb:419-433`). The fallback accepts:

1. `authenticated=true`; and
2. a `username` cookie that decrypts to a non-empty login name
   (`PilotLegacySession.vb:101-142`).

It does not require the password, verify `sessionIDadmin` in Redis, bind the
claim to a server-side session, verify an issuance time, or authenticate the
ciphertext with a MAC. It then issues a new managed ticket
(`PilotSecurity.vb:480-499`).

Consequences:

- captured ciphertext can mint a managed session as the captured user;
- browser cookie `Path=/admin` does not prevent an attacker from sending the
  cookie directly to a managed URL;
- logout and Redis deletion do not revoke the ciphertext;
- account lockout is checked only when the cookie is redeemed, not cryptographically
  bound to the credential;
- one shared-key disclosure compromises all cookies encrypted with that key.

Plan: [`legacy-auth-bridge-hardening-plan.md`](legacy-auth-bridge-hardening-plan.md).

### High — legacy session fixation

`PilotLegacySession.GetOrCreateAdminSessionId` adopts any non-empty
`sessionIDadmin` supplied by the browser (`PilotLegacySession.vb:179-187`).
After successful login, the pilot writes the authenticated user's `LoginName`
under that known identifier in Redis. A session identifier must be rotated at
every authentication or privilege transition.

### High — managed and legacy identities can diverge

During refresh, the bridge reuses the request's `username` cookie when present
instead of deriving it from the managed authenticated user
(`PilotLegacySession.vb:64-82`). A request can therefore carry a managed ticket
for one identity and cause the bridge to reissue a legacy credential for
another. Refresh must always derive outbound identity from `user.UserName`.

### High — unauthenticated legacy encryption

The compatibility codec is Blowfish-CBC without a MAC
(`PerlCryptCbcBlowfish.vb`). `RemoveSpacePadding` trims spaces but does not
validate padding (`:118-129`), so successful decryption is not evidence that a
ciphertext is authentic. Key and IV derivation use repeated uniterated MD5
blocks (`:77-105`). This is acceptable only as a temporary outbound
compatibility format; it must not be an inbound authentication primitive.

### High — recoverable user passwords in cookies

On managed login, `PilotLegacySession.Ensure` encrypts the user's actual
password and sends it in the `password` cookie (`PilotLegacySession.vb:33-51`,
`:195-205`). Anyone with the site-wide legacy key can recover admin passwords,
creating password-reuse risk outside this system. Confirm whether any remaining
legacy tool requires this cookie; stop issuing it at the earliest compatible
phase.

### High — TLS certificate verification disabled on identity-bearing call

`includes/ssi.inc:47` sets `MSXML2.ServerXMLHTTP.6.0` option `13056`, ignoring
certificate errors. That server-side request forwards session cookies to
`authorize.ashx`; `topshell.asp:38-40` trusts its `OK|username` response and
stores the returned identity in Classic ASP session state. Certificate
verification must be enabled. If private PKI is involved, trust the intended CA
instead of disabling validation.

### High — login throttling is reset by dropping a cookie

`managed/login.ashx:98-105` limits failures using
`context.Session("PilotFailedAttempts")` (`:175-190`). A client can discard the
ASP.NET session cookie between attempts and receive a fresh counter. Use an
account- and network-aware server-side throttle with bounded retention. Preserve
generic errors so it does not become a username-enumeration oracle.

### High — Code Admin editable-class probe fails open

`CodeAdminRepository.DetectClassEditColumn` catches every error and reports that
the column is unsupported (`CodeAdminRepository.vb:625-637`). In that mode,
`ListEditableClasses` returns every class, and `ReadClass` marks all of them
editable (`:44-48`, `:640-647`). A transient database/schema/permission error
therefore expands authorization. Authorization-relevant metadata must fail
closed.

### High — Code Admin has one broad capability for all editable classes

`CodeAdminAccess.CanOpenApp` is the mutation boundary for all code classes.
There is no per-class capability. Delete protection for `GROUP_TY_CD` and
`APPLICATION_DB` does not prevent create, update, activation, deactivation, or
reordering (`CodeAdminRules.vb:125-128`). Before global rollout, classify code
classes by sensitivity and require explicit capabilities for security,
connection, integration, and credential-bearing classes.

### High — PayPal password is treated as ordinary Code Admin data

For organization `825`, `optionValue4` is labeled `PayPal Password`
(`CodeAdminFieldMetadata.vb:134-138`). It is loaded and serialized like ordinary
text. This exposes a credential through database reads, API responses, browser
memory, screenshots, logs, and anyone with general Code Admin access. Replace
with a secret reference or write-only secret workflow; never return the stored
value.

### High/Medium — dynamic metadata selects a SQL column

The Code Admin in-use check concatenates
`membership_column_detail.column_desc` into a query after validating it as a
SQL identifier (`CodeAdminRepository.vb:566-574`). This blocks punctuation-based
SQL injection but does not prove the identifier is an approved membership
column. Treat database metadata as untrusted and resolve it through an explicit
allowlist/schema map.

### Medium — open redirect in the legacy bridge

`global-bridge/pilot-bridge.asp:10` reads `returnUrl`; line 40 redirects to it
without validation. `pilot-establish.ashx` validates its own local copy but
returns only `OK`, so the ASP layer still redirects to the original untrusted
value. Return the validated destination from the managed handler or enforce the
same configured-route allowlist in the bridge.

### Medium — exception details leak schema and provider information

`PilotJsonApi.HandleServiceException` returns exception type and message
whenever the current host is an enabled pilot host
(`PilotJsonApi.vb:143-163`). That is the normal deployed case, not a development
check. ODBC/OleDb messages can reveal tables, columns, SQL fragments, provider
details, and filesystem paths. Log a correlation id server-side and return only
the generic response.

### Medium — Access Manager route probe is a same-host SSRF primitive

The `checkRoute` action constructs a URI from user-supplied `scriptName` and
issues a server-side HEAD request
(`AccessManagerApiHandlers.vb:254-261`, `:296-302`). It is available to a
workspace user without requiring script-management capability. Validation
blocks common traversal forms, but the feature still enumerates same-host IIS
paths and applications. Require `CanManageScripts` and constrain targets to
configured admin roots, or replace HTTP probing with a database/filesystem check.

### Medium — Access Manager capability is global, not object-scoped

Fine-grained capabilities distinguish sections, scripts, memberships, and
grants, but each capability applies to every object of that type. A grant
administrator can change any grant by id; a script administrator can delete any
script. This may be the intended full-admin model. If capabilities are delegated
more broadly, add tenant/section scope and verify target membership server-side.

### Medium — Access Manager accepts arbitrary section parents

`CreateSectionCommand.ParentId` is persisted without confirming the parent
exists or that nested sections are supported. The current workspace lists root
sections, so unexpected parents can hide records or interfere with deletion
rules. Validate the parent or force the supported root value.

### Medium — no Content Security Policy or frame protection

The shell has no CSP, `frame-ancestors`, or equivalent frame defense in source.
It loads jQuery, jQuery UI, Bootstrap, Font Awesome, and other assets from public
CDNs without Subresource Integrity (`PilotShell.vb:34-38`, `:89`). An admin
surface should self-host pinned assets where practical, add a nonce/hash-based
CSP, and deny framing unless a documented integration requires it.

### Medium — plaintext credentials leave the application during login

`PilotPasswordHasher.Verify` sends the entered password and salt to
`https://ws.ebigpicture.com/hash.aspx` (`PilotSecurity.vb:338-357`). This makes
the external service part of the credential trust boundary and availability
path. Move hash verification into the application with the exact legacy
algorithm, then migrate stored hashes to a modern adaptive scheme after login.

### Medium — no durable audit trail for administrative changes

The new Access Manager and Code Admin code does not record a security audit event
containing actor, action, target, before/after state, request correlation id,
result, and timestamp. Database `modify_by` fields are useful but not a complete,
tamper-resistant audit log. Add one before broader use.

### Medium — Classic ASP output encoding remains inconsistent

Copied tools render database values directly into HTML and attributes. Examples
include SMS recipient, message, status, and error text
(`sms_logs.asp:68-105`, `:147-151`). Stored values can become stored XSS. Every
Classic ASP migration must classify output context and use HTML, attribute, URL,
or JavaScript encoding accordingly; .NET rewrites should use templating that
encodes by default.

### Medium — `topshell.asp` embeds a query-derived URL in JavaScript

`topshell.asp:68-75` appends the raw query string to `pilotPrintUrl` and escapes
only the apostrophe before emitting a JavaScript string. It does not handle
backslashes or `</script>`. Use JSON/JavaScript-string encoding in a managed
template, or set the value through a data attribute with appropriate encoding.

### Low — logout is a state-changing GET

`managed/logout.ashx` signs out on any request without CSRF. This is usually
limited to logout CSRF (availability/nuisance), not account compromise. Prefer
POST with a CSRF token when the shell no longer needs the legacy GET contract.

### Low — host-name substring determines development banner

`PilotConfig.IsDevelopmentSite` treats any hostname containing `dev` as
development (`PilotSecurity.vb:113-122`). It currently controls presentation,
not authorization. Keep it away from future security or diagnostic-disclosure
decisions and use explicit configuration.

## Positive security controls

- Access Manager SQL uses positional ODBC parameters; no request-controlled
  `ORDER BY`, raw value concatenation, or dynamic `IN` values were found.
- Code Admin is not a generic table editor. CRUD targets fixed tables, request
  values are parameterized, and patch fields use a `Select Case` allowlist.
- Managed mutations use `AdminShellApiGuard.RequireAuthorizedMutation`, which
  performs authentication, authorization, and CSRF verification.
- Route authorization fails closed for unknown mappings.
- Forms Authentication cookies are `HttpOnly`, `Secure`, and `SameSite=Lax`.
- Return URL policy rejects absolute and protocol-relative URLs and requires a
  configured route; the gap is specifically the separate ASP bridge.
- Dynamic UI values in the managed screens generally use Vue interpolation,
  `textContent`, or explicit escaping; no Code Admin `v-html` was found.
- Secrets are excluded by `.gitignore`; the committed local-config template is
  empty. A history search found references to the key name but no key value.
- Multi-statement destructive operations in the new repositories generally use
  transactions.

## Correctness and data-integrity findings

### Access Manager reorder ignores optimistic concurrency

`ReorderSectionCommand.ExpectedUpdateNo` exists, but the service and repository
do not test it or increment `update_no`. Section-script reorder has no token.
Concurrent edits become silent last-write-wins operations. Apply the same
optimistic-concurrency contract used by lifecycle mutations.

### Position allocation can race

Section and membership creation/activation use `max(position)+1` inside a
transaction without a demonstrated uniqueness constraint or locking strategy.
Concurrent requests can assign duplicate positions. Use a protected sequence,
appropriate row/table lock, or a uniqueness constraint plus retry.

### Grant creation has a check-then-insert race

`FindGrant` occurs outside the insertion transaction. Concurrent requests can
create duplicates unless the database has a unique constraint. Enforce
uniqueness in the database and handle the constraint result.

### Effective-access response is incomplete

The model exposes `DirectSectionGrants`, but the repository initializes it and
never loads it. Remove the misleading property or implement the missing query.

### Code Admin rebuilds complete relation tables

For one organization-specific workflow,
`RebuildLicenseObjTypeTables` deletes all rows from two tables and rebuilds them
inside a transaction (`CodeAdminRepository.vb:496-501`). Atomicity is good, but
blast radius is large. Add parity tests, row-count safeguards, audit output, and
an induced-rollback test before relying on it globally.

### Broken Code Admin inline-save race guard

`managed/code-admin/js/app.js:166-176` compares against
`this.selectedClass` inside Composition API `setup()` under strict mode. It
should use reactive `state.selectedClass`. The intended stale-response/write
guard can throw or mis-evaluate.

### Code Admin lookup hydration covers only some option fields

`CodeAdminService.GetFieldValue` maps a subset of supported option fields. Other
lookup-backed fields do not pass selected inactive values into hydration. Make
field access metadata-driven rather than another hard-coded switch.

## Performance findings

### Repeated authorization queries

Authentication reloads the user from the database, route authorization executes
a substantial ACL query, and Access Manager capability resolution performs four
more ACL queries. Capability helpers can resolve the same set repeatedly during
one request. Cache the current user, route decision, and capability set in
`HttpContext.Items`; use short-lived cross-request caching only with explicit
invalidation.

### Two blocking HTTP round trips for Classic ASP chrome

Each copied ASP page calls `authorize.ashx` and `chrome.ashx` synchronously
through server-side HTTP. Both repeat authentication and authorization work.
Remove this architecture when tools move under the global managed shell; in the
interim, combine the result or memoize request-scoped decisions after measuring.

### Access Manager grant-list N+1 queries

Grant listing loads labels and principals per result. Join or batch these
lookups. The current pattern increases connection/command count linearly.

### Access Manager lists are unpaged

Section and script lists can load complete result sets. Add capped pagination
and server-side filtering before global scale broadens the data set.

### Code Admin paginates in memory

`CodeAdminService.ListValues` loads the entire matching class and slices a
200-row page in memory. Push paging and count operations into SQL.

### Code Admin delete checks are N+1

Bulk delete loads each record and opens additional queries/connections for each
referencing column. Cap batch size and perform set-based usage checks.

### Configuration and route parsing repeat

App settings and route mappings are parsed repeatedly during requests. Validate
configuration at application start and cache immutable parsed mappings.

## Maintainability and design findings

- The handler → guard → service → repository → contract structure is consistent
  and should be retained.
- `ILegacyMembershipCredentialEncoder` is a good anti-corruption boundary and
  gives the legacy crypto an explicit deletion path.
- `AccessManagerRepository.vb`, `CodeAdminRepository.vb`, and
  `sections-view.js` have become large multi-responsibility modules. Split by
  aggregate/workflow, not merely by line count.
- MySQL/ODBC and Oracle/OleDb require different provider adapters, but common
  transaction/error/command patterns can still be factored behind small
  provider-neutral helpers.
- `PilotShell.vb` builds large HTML and inline JavaScript strings. Move toward a
  managed template/component so context-sensitive encoding is the default.
- `PilotLegacyMembershipCrypto` is a pass-through facade over another facade and
  can disappear with the bridge.
- `global-bridge/pilot-bridge.asp` hardcodes `/dev/adminshell`, contradicting the
  relocatable-platform design.
- Domain constants (`SCRI`, `SECT`, `GROU`, `G`, `Y/N`) are inconsistently
  centralized.
- Organization-specific Code Admin behavior is encoded as hard-coded ids and
  branches. Move it to explicit tested policy/configuration where feasible.
- Error handling is inconsistent: some paths return typed service errors,
  others swallow exceptions, and deployed APIs expose provider details.

## Test and delivery findings

The test suite contains valuable service fakes and behavior checks, but it is
not a unified automated test system:

- VB tests are hand-built console programs compiled with documented `vbc`
  commands;
- JavaScript tests are standalone Node scripts;
- several UI tests assert that source files contain literal strings, which tests
  implementation text rather than behavior;
- there is no repository CI workflow;
- deployed behavior depends on IIS, App_Code dynamic compilation, ODBC/OleDb,
  Redis, parent application settings, and remote services that local tests do
  not reproduce.

Highest-priority missing tests:

1. valid legacy ciphertext with and without a matching Redis session;
2. session-id rotation on login and Redis teardown on logout;
3. managed/legacy identity mismatch;
4. brute-force throttle across new ASP.NET sessions;
5. bridge return URL rejection;
6. TLS failure on the internal authorization hop;
7. Code Admin edit-column probe failure must deny all;
8. per-class authorization for sensitive Code Admin classes;
9. Access Manager reorder concurrency conflict;
10. stored-XSS payloads through copied ASP and managed APIs;
11. transaction rollback for organization-specific table rebuilds;
12. authorization and CSRF tests for every mutation endpoint.

Create one repeatable command for unit tests and one IIS integration suite. CI
should compile every VB source combination used in deployment, run JS tests,
scan dependencies/secrets, and publish results.

## Remediation order

### Before broader rollout

1. Implement Phase 0 and Phase 1 of the legacy bridge hardening plan, or disable
   inbound legacy trust entirely.
2. Re-enable TLS verification in `ssi.inc`.
3. fix bridge return URL validation.
4. replace the session-only login throttle.
5. make Code Admin editable-class detection fail closed.
6. remove PayPal and any other credentials from generic Code Admin responses.
7. stop exposing exception details and add audit logging.

### Before moving into the global admin path

1. Define the global authentication/session authority and remove client-specific
   host/user allowlists.
2. add per-class Code Admin authorization.
3. add security headers and self-host/pin browser dependencies.
4. establish automated build/test/deploy gates.
5. inventory every ASP, Perl, AJAX, include, web-service, and database contract.
6. classify which copied ASP tools remain temporarily and which are rebuilt.

### Engineering follow-up

1. enforce optimistic concurrency for reorder operations;
2. add server-side paging and remove N+1 query patterns;
3. split oversized repository/UI modules;
4. replace HTML string construction and context-unsafe ASP output;
5. centralize immutable configuration and domain constants.

## Status

- [x] Source review recorded
- [x] Legacy auth hardening plan written
- [ ] Critical/high findings remediated
- [ ] IIS security headers and parent configuration reviewed
- [ ] Complete global ASP/Perl dependency inventory completed
- [ ] Automated CI and integration test harness established
- [ ] Follow-up review after global-path migration
