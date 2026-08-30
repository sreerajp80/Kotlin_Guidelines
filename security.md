# Security

Use this document when the repository handles secrets, protected personal data, health data,
financial data, private files, or any local encrypted store.

If the app is not security-sensitive, keep this file short and document that decision explicitly.

---

## 1. Security Scope

- App: `<app name>`
- Data sensitivity level: `low`, `moderate`, or `high`
- Engineering standard profiles in force:
  - `Core Baseline`
  - `Sensitive Data Extension` if applicable
- Platform: `Android` (minSdk `<NN>`, targetSdk `<NN>`)

---

## 2. Security Objectives

- `<objective 1>`
- `<objective 2>`
- `<objective 3>`

Example objectives:
- Protect locally stored data from casual extraction on a lost or stolen device.
- Prevent accidental disclosure through logs, backups, screenshots, or exports.
- Preserve recoverability and migration without weakening encryption.
- Comply with OWASP Mobile Top 10 controls for all applicable risk categories.

---

## 3. Threat Model Summary

Document the threats the product is designed to address and those it explicitly does not address.

### In Scope Threats

- `<lost or stolen device>`
- `<casual local access by another user>`
- `<accidental plaintext export>`
- `<log leakage of sensitive data>`
- `<reverse engineering of application logic from release binary>`

### Out Of Scope Threats

- `<fully compromised/rooted device>`
- `<physical hardware attacks>`
- `<attacks requiring OS-level compromise>`
- `<nation-state adversaries>`

---

## 4. Sensitive Data Inventory

| Data Type | Example | Where It Exists | Protection Required |
|-----------|---------|-----------------|---------------------|
| `<secret>` | `<example>` | `<memory/db/export>` | `<control>` |
| `<token>` | `<example>` | `<memory/storage>` | `<control>` |
| `<user data>` | `<example>` | `<storage/logs?>` | `<control>` |

---

## 5. Storage Model

### At Rest

- Primary local storage: `<Room / files / etc.>`
- Secure key storage: `<EncryptedSharedPreferences / AndroidKeystore / etc.>`
- Backup behavior: `<disabled/restricted/encrypted/plaintext not allowed>`

### In Memory

- Sensitive values are kept in memory: `<briefly / cached / long-lived>`
- Memory clearing strategy: `<lock/background/manual clear>`

### In Transit

- Network use: `<none / https api / internal network>`
- Transport protections: `<tls/pinning/none>`

---

## 6. Cryptography Design

Document only the design, not the secrets.

- Encryption algorithm: `<AES-256-GCM/etc.>`
- Key derivation: `<PBKDF2/Argon2/etc.>`
- Nonce or IV strategy: `<random per record>`
- Format versioning: `<how versioning works>`
- Legacy format support: `<yes/no and why>`

### Rules

- Keys, IVs, salts, and passwords must never be hardcoded.
- Randomness must use cryptographically secure generation (`SecureRandom`).
- Encrypted formats must be versioned.

---

## 7. Authentication And Access Control

- App-lock strategy: `<none / biometric / pin / password / device credential>`
- Biometric implementation: `<BiometricPrompt with BIOMETRIC_STRONG / DEVICE_CREDENTIAL fallback>`
- Fallback behavior: `<behavior>`
- Session-expiry rule: `<rule>`
- Background lock rule: `<triggered on Activity.onPause or ProcessLifecycleOwner>`
- Lock screen implementation: `<path to the lock screen Composable>`

---

## 8. Binary Protections

### 8.1 R8 / ProGuard Code Shrinking

Production release builds SHOULD enable R8 code shrinking:

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro",
        )
    }
}
```

R8 performs dead code elimination, obfuscation (renames classes and methods to short meaningless
names), and optimization. The `mapping.txt` file produced in `app/build/outputs/mapping/release/`
is required to decode stack traces from crash reports.

**Mapping file archive policy:**
- The `mapping.txt` MUST be archived securely after every production release build.
- Mapping files MUST be retained for the lifetime of the released version.
- Mapping files MUST NOT be committed to source control (add `build/` to `.gitignore`).
- Without the mapping file, stack traces from that version are permanently unreadable.

### 8.2 Debuggable Verification

Verify that `android:debuggable` is `false` in the merged release manifest before every
production release. A debuggable production build is a security vulnerability and a Google Play
policy violation.

Check via `aapt2`:

```bash
aapt2 dump badging app/build/outputs/apk/release/app-release.apk | grep -i debuggable
```

```powershell
aapt2 dump badging app\build\outputs\apk\release\app-release.apk | Select-String -Pattern debuggable
```

Expected output: no `application-debuggable` line present. If it appears, investigate
`buildTypes.release` in `build.gradle.kts`.

---

## 9. Logging And Telemetry Policy

### Never Log

- Secrets
- Tokens
- Recovery codes
- Decrypted payloads
- Full database row content that may contain user data
- Sensitive personal data unless explicitly approved

### Allowed Diagnostic Context

- Operation name
- Screen or flow name
- Error category (not raw exception message if it may contain user data)
- Non-sensitive identifiers where justified

### Logging Controls

- Verbose logging gate: `<BuildConfig.DEBUG flag>`
- Log level in production: `info` and above only (no `Log.d` or `Log.v` output)
- Redaction strategy: `<e.g. user-provided field values replaced with [REDACTED]>`

---

## 10. Platform Security Controls

### Android

- `android:allowBackup`: `<true/false and why>`
  - For sensitive-data apps: set to `false` or use `android:fullBackupContent` to explicitly
    exclude sensitive directories.
- `android:fullBackupContent`: `<value>`
- Screenshot protection:
  - Set `FLAG_SECURE` on the window to prevent screenshots and screen recording:
    ```kotlin
    window.setFlags(
        WindowManager.LayoutParams.FLAG_SECURE,
        WindowManager.LayoutParams.FLAG_SECURE,
    )
    ```
  - Apply on all screens showing sensitive data; remove on non-sensitive screens if UX requires.
- `android:debuggable`: MUST be `false` in release builds (verify per section 8.2).
- Root detection: `<if any — e.g. Play Integrity API>`
- Network security config: `<res/xml/network_security_config.xml if applicable>`

---

## 11. Permissions

| Permission | Why It Is Needed | Requested When | Denial Handling |
|------------|------------------|----------------|-----------------|
| `<permission>` | `<reason>` | `<point of use>` | `<behavior>` |
| `<permission>` | `<reason>` | `<point of use>` | `<behavior>` |

Permission review rules:
- Request only permissions the app currently uses. Remove unused permissions promptly.
- For offline apps: verify `INTERNET` permission is absent from the merged release manifest.
- Dangerous permissions MUST be requested at the point of use with a rationale, not at startup.
- The app MUST function in a degraded but safe state if a non-critical permission is denied.

---

## 12. OWASP Mobile Top 10 Compliance

Review and sign off each item before every production release.

| ID | Risk | Control | Status |
|----|------|---------|--------|
| M1 | Improper Credential Usage | No hardcoded secrets; platform secure storage used | `verified / n/a / risk-accepted` |
| M2 | Inadequate Supply Chain Security | `libs.versions.toml` pinned; dependency audit performed; licenses verified | `verified / n/a / risk-accepted` |
| M3 | Insecure Authentication | App lock enforced; background lock on `onPause` | `verified / n/a / risk-accepted` |
| M4 | Insufficient Input/Output Validation | All user input validated; DB writes via parameterized Room queries | `verified / n/a / risk-accepted` |
| M5 | Insecure Communication | No network traffic (offline) OR TLS-only with valid certificates | `verified / n/a / risk-accepted` |
| M6 | Inadequate Privacy Controls | Data inventory reviewed; no PII in logs; sensitive fields excluded from backup | `verified / n/a / risk-accepted` |
| M7 | Insufficient Binary Protections | R8 `isMinifyEnabled = true`; `android:debuggable=false` verified | `verified / n/a / risk-accepted` |
| M8 | Security Misconfiguration | Permissions minimal; backup config explicit; debug features disabled in prod | `verified / n/a / risk-accepted` |
| M9 | Insecure Data Storage | No sensitive data in plain `SharedPreferences`; no sensitive data in unencrypted files | `verified / n/a / risk-accepted` |
| M10 | Insufficient Cryptography | Versioned encrypted formats; secure key derivation; no hardcoded keys | `verified / n/a / risk-accepted` |

For each `risk-accepted` item, document the justification and owner below the table.

---

## 13. Data Retention And Purge Policy

Define what data is stored, how long it lives, and what triggers deletion.

### Retention Schedule

| Data Type | Retention / Rotation Policy | Deletion Trigger |
|-----------|-----------------------------|------------------|
| `<user content>` | `<indefinite / N days>` | `<user delete action / account purge>` |
| `<log files>` | `<max 5 MB per file, 3 rotated files>` | `<size limit reached / app uninstall>` |
| `<temp export files>` | `<session only>` | `<export complete or app close>` |
| `<cache>` | `<OS discretion>` | `<stored in cache dir — cleared by OS under pressure>` |

### Purge Implementation

- Provide a user-accessible **"Delete all data"** action in Settings that removes:
  - All database files
  - All log files
  - All cache files
  - All secure storage entries
  - All temporary files in the app's internal and cache directories

- Temporary files MUST be created in `context.cacheDir` and deleted within the same session.

---

## 14. Backup, Import, Export, And Recovery

- Backup supported: `<yes/no>`
- Backup format: `<encrypted/plaintext/both>`
- Import supported: `<yes/no>`
- Recovery flow: `<description>`
- Plaintext export policy: `<disallowed / allowed with confirmation>`

---

## 15. Security Testing Strategy

| Area | Test Type | Notes |
|------|-----------|-------|
| Crypto format | Unit | Deterministic test vectors; encrypt then decrypt round-trip |
| Secret storage | Unit or integration | Verify no write to plain SharedPreferences or plaintext file |
| Lock / auth flow | UI or integration | Lock triggers on pause; re-auth required on resume |
| Backup and recovery | Integration | Full round-trip: export → purge → import → verify data intact |
| Data purge | Integration | After purge: verify DB, logs, cache, and secure storage are empty |
| R8 / ProGuard | Release build verification | Confirm `isMinifyEnabled = true` in release build type |
| Debuggable | Release build verification | `android:debuggable=false` confirmed in merged manifest |
| Permission audit | Release build verification | Merged manifest contains only declared, needed permissions |

---

## 16. Incident Response Notes

- Triage owner: `<owner>`
- Severity model: `<brief model>`
- Immediate containment actions:
  - `<action 1: e.g. halt distribution of affected version>`
  - `<action 2: e.g. notify affected users>`
- Patch release process reference: `release_process.md`

---

## 17. Open Risks And Future Hardening

- Risk: `<risk>`
  Hardening option: `<option>`
- Risk: `<risk>`
  Hardening option: `<option>`

---

## 18. Security Review Checklist

Complete before every production release.

- [ ] Threat model reviewed and current.
- [ ] Sensitive data inventory updated.
- [ ] Logging policy reviewed — no new log statements introduce PII exposure.
- [ ] Storage and backup behavior reviewed.
- [ ] Permission usage reviewed — no unnecessary permissions.
- [ ] R8/ProGuard `isMinifyEnabled = true` confirmed in release build type.
- [ ] Mapping file (`mapping.txt`) archived for this release version.
- [ ] `android:debuggable=false` verified in merged release manifest.
- [ ] OWASP Mobile Top 10 checklist (section 12) completed and signed off.
- [ ] Data retention policy reviewed — purge paths tested.
- [ ] Recovery, import, export, and migration paths tested.
- [ ] Platform-specific controls verified (FLAG_SECURE, EncryptedSharedPreferences, BiometricPrompt).
- [ ] Tests cover the highest-risk failure modes.
