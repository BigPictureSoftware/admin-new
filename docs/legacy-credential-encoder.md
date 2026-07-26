# Legacy membership credential encoding

The pilot bridges sign-in to legacy `/admin/admin` cookies through a small,
replaceable encoder layer. **Call sites use `LegacyMembershipCredentialCodec` /
`PilotLegacyMembershipCrypto`, not Blowfish directly.**

## Why this exists

Legacy admin stores `username` and `password` cookies as Perl
`membershipEncryption` values (Crypt::CBC + Blowfish). Legacy topshell decrypts
those cookies on every page load. The pilot must emit the same format until
global admin moves to a modern auth contract.

## Pluggable design

| Type | Role |
|------|------|
| `ILegacyMembershipCredentialEncoder` | Encode/decode contract |
| `LegacyMembershipCredentialEncoderFactory` | Chooses implementation from config |
| `LegacyMembershipCredentialCodec` | App-facing facade |
| `PerlCryptCbcBlowfishCredentialEncoder` | **Legacy adapter** — delete when global modernizes |
| `PerlCryptCbcBlowfish` / `BlowfishBlockCipher` | Implementation details of the adapter |

### Configuration

Committed settings live in `managed/web.config`. **Do not commit secrets there.**

Copy `managed/web.config.local.example` to `managed/web.config.local` (gitignored) on each server:

```xml
<add key="PilotMembershipEncryptionKey" value="..." />
```

`web.config` loads overrides via `<appSettings file="web.config.local">`.

```xml
<add key="PilotLegacyCredentialEncoder" value="perl-crypt-cbc-blowfish" />
```

| `PilotLegacyCredentialEncoder` | Behavior |
|--------------------------------|----------|
| `perl-crypt-cbc-blowfish` (default) | Perl-compatible Blowfish cookies |
| `disabled` | Skip credential cookies; Redis bridge only |

### Moving to a modern scheme

1. Implement `ILegacyMembershipCredentialEncoder` (e.g. signed JWT fragment, opaque ticket, etc.).
2. Register it in `LegacyMembershipCredentialEncoderFactory`.
3. Set `PilotLegacyCredentialEncoder` to the new scheme id.
4. Delete `PerlCryptCbcBlowfish*` and `BlowfishBlockCipher.vb`.
5. When global admin no longer reads legacy cookies, remove `PilotLegacySession` entirely.

## Redis session bridge (separate concern)

`PilotLegacySession` writes CacheManager-compatible Redis keys via `RedisSession`
and sets `/admin` cookies. Credential encoding is only the `username` /
`password` cookie values; Redis `LoginName` does not use Blowfish.

`Ensure` always writes `sessionIDadmin`, `authenticated`, and Redis `LoginName`
even if credential encoding fails (missing key).

## Bidirectional bridge

| Direction | Mechanism |
|-----------|-----------|
| Pilot → legacy | `PilotAuth.SignIn` calls `PilotLegacySession.Ensure` |
| Legacy → pilot | Browser hits `/admin/admin/pilot-bridge.asp`, which forwards legacy cookies to `managed/pilot-establish.ashx` and sets `bp_admin_next` |

Legacy auth cookies (`username`, `authenticated`, etc.) use `Path=/admin`. The
browser does **not** send them to `/dev/adminshell/...`, so pilot Classic ASP
cannot read them directly. `topshell.asp` redirects through `pilot-bridge.asp`
(under `/admin/admin`) before showing `login.html`.

`PilotAuth.TryGetCurrentUser` also accepts legacy cookies when they are present
on the request (e.g. `pilot-establish.ashx` and direct managed API calls).

> **Security note.** That inbound acceptance treats the `username` ciphertext as
> a bearer credential: no password, no Redis check, no MAC, and no server-side
> state to revoke. Cookie `Path=/admin` does not restrict it, because the handler
> reads the cookie from any URL. See
> [`legacy-auth-bridge-hardening-plan.md`](legacy-auth-bridge-hardening-plan.md)
> before relying on or extending this direction of the bridge.

## Tests

- `managed/App_Data/tests/PerlCryptCbcBlowfishTests.vb` — padding, KDF, round-trip
- `managed/App_Data/tests/PilotLegacySessionTests.vb` — session id + Redis key shape

## Server setup

1. Copy `managed/web.config.local.example` → `managed/web.config.local` on the IIS host.
2. Set `PilotMembershipEncryptionKey` to the legacy `CONF{'encryption_key'}` value (never commit this file).
3. Recycle the app pool after deploying new `App_Code` files.
