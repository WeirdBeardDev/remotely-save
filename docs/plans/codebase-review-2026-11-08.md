# Remotely Save Codebase Review (2026-11-08)

## Scope

Comprehensive review of the Obsidian plugin codebase with focus on:

- Security
- Obsidian plugin behavior and UX
- Reliability/quality issues likely to impact Android ↔ Windows Dropbox sync workflows

## Executive summary

- **No immediately exploitable vulnerabilities** were found in the dedicated security scan pass.
- I found multiple **important correctness and hardening issues** that should be addressed, including one high-impact token-expiry bug, settings/features that appear wired in UI but not enforced in sync logic, and sensitive-data handling footguns.
- Current tests are mostly healthy, but there is **1 failing test** on this environment.

---

## Security-focused findings

### 1) Sensitive settings export includes long-lived credentials (Medium)

The QR/URI export logic includes full credential-bearing config for several providers:

- S3 full config export: [`src/importExport.ts:28`](../../src/importExport.ts)
- Dropbox full config export: [`src/importExport.ts:30`](../../src/importExport.ts)
- WebDAV full config export: [`src/importExport.ts:34`](../../src/importExport.ts)
- Webdis full config export: [`src/importExport.ts:36`](../../src/importExport.ts)

This is useful for migration, but creates high leakage risk (screenshots, clipboard history, logs, chat paste, etc.).

**Recommendation**

- Add an explicit warning/confirmation modal before export.
- Offer two export modes per service: **safe (no secrets/tokens)** and **full migration (with secrets)**.
- Consider short-lived encrypted export payloads when possible.

### 2) Obfuscation is reversible encoding, not encryption (Medium)

- Default is enabled: [`src/main.ts:99`](../../src/main.ts)
- Implementation is reversible base64url + reverse-string: [`src/configPersist.ts:30`](../../src/configPersist.ts), [`src/configPersist.ts:58`](../../src/configPersist.ts)

This can create a false sense of security for `data.json`.

**Recommendation**

- Keep current behavior for compatibility, but rename UX wording from “obfuscate” to “light obfuscation (not encryption)”.
- Add a stronger warning in settings/README for local credential exposure.

### 3) Encryption mode uses AES-CBC without authenticity (Hardening)

OpenSSL-compatible crypto uses PBKDF2 + AES-CBC:

- [`src/encryptOpenSSL.ts:4`](../../src/encryptOpenSSL.ts)
- [`src/encryptOpenSSL.ts:58`](../../src/encryptOpenSSL.ts)

CBC alone does not provide integrity/authenticity (tampering not strongly detectable).

**Recommendation**

- Keep for compatibility, but add an authenticated option (e.g., AES-GCM) for new profiles.
- Document the integrity limitation of legacy compatibility mode.

---

## Functional / Obsidian plugin findings

### 4) OneDrive token expiry timestamp bug (High)

Initial auth computes expiry with seconds added directly to milliseconds:

- [`src/fsOnedrive.ts:197`](../../src/fsOnedrive.ts)

Current code:
- `Date.now() + authRes.expires_in - 5 * 60 * 1000`

Expected unit handling:
- `Date.now() + authRes.expires_in * 1000 - ...`

This can force near-immediate refresh loops and unstable auth behavior.

**Recommendation**

- Multiply `expires_in` by `1000` in initial auth path (refresh path already does this correctly at [`src/fsOnedrive.ts:522`](../../src/fsOnedrive.ts)).

### 5) `onlyAllowPaths` appears in settings but is not enforced in sync logic (High)

Configured in settings/UI and defaults:

- [`src/settings.ts:2103`](../../src/settings.ts)
- [`src/main.ts:91`](../../src/main.ts)

But sync logic only references `ignorePaths`, not `onlyAllowPaths`:

- [`src/sync.ts:116`](../../src/sync.ts)
- [`src/sync.ts:1470`](../../src/sync.ts)

This is a major behavior mismatch between UX and actual sync behavior.

**Recommendation**

- Implement `onlyAllowPaths` filtering in the same decision path as `ignorePaths`.
- Add tests verifying allowlist semantics.

### 6) Async config persistence not awaited in protocol import/auth callback flows (Medium)

`saveSettings()` is called without `await` in callback/import code paths:

- [`src/main.ts:541`](../../src/main.ts)
- Auth config updates called without `await`: [`src/main.ts:596`](../../src/main.ts), [`src/main.ts:687`](../../src/main.ts)

Can lead to race conditions/unhandled persistence failures.

**Recommendation**

- `await` all settings persistence/update calls in protocol and auth handlers.

### 7) Metadata serialization typo prevents version assignment (Medium)

In remote metadata serialization:

- [`src/metadataOnRemote.ts:44`](../../src/metadataOnRemote.ts)

Uses comparison instead of assignment:
- `y["version"] === DEFAULT_VERSION_FOR_METADATAONREMOTE;`

**Recommendation**

- Replace `===` with `=`.
- Add a regression test for defaulted `version` field.

### 8) Sensitive settings can be dumped to console by debug action (Low)

- [`src/settings.ts:2535`](../../src/settings.ts)

Useful for support, but risky for shared-device/log collection scenarios.

**Recommendation**

- Add redaction for tokens/passwords before console output.
- Gate behind explicit confirmation.

---

## Test/quality findings

### 9) Test suite has one failure in this environment (Medium)

`npm test` result:
- 52 passing
- 1 failing: `Encryption OpenSSL tests -> should encrypt text file and get the same result as openssl`
- Assertion location: [`tests/encryptOpenSSL.test.ts:77`](../../tests/encryptOpenSSL.test.ts)

Likely cross-environment incompatibility (text normalization/toolchain differences) since binary OpenSSL compatibility test still passes.

**Recommendation**

- Make this assertion robust across environments (normalize fixture or compute expected vector in a deterministic environment).

---

## Documentation findings

### 10) Service matrix has conflicting/duplicate entries (Low)

In service compatibility doc:

- Seafile listed twice with conflicting status: [`docs/services_connectable_or_not.md:25`](../services_connectable_or_not.md), [`docs/services_connectable_or_not.md:29`](../services_connectable_or_not.md)
- Azure Blob appears in two forms with conflicting states: [`docs/services_connectable_or_not.md:20`](../services_connectable_or_not.md), [`docs/services_connectable_or_not.md:37`](../services_connectable_or_not.md)

**Recommendation**

- De-duplicate rows and clarify “officially supported now” vs “planned/pro”.

---

## Prioritized action plan

1. Fix OneDrive expiry unit bug.
2. Implement and test `onlyAllowPaths` enforcement.
3. Fix metadata version assignment typo + add tests.
4. Await and harden settings/auth persistence flows.
5. Reduce credential leakage risk in export/debug tooling.
6. Clean up service matrix documentation inconsistencies.
