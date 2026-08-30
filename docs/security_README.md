# Explainer: Security Blueprint

## What is this document?

`security.md` is a **security design and compliance blueprint** for Kotlin Android applications that handle sensitive user data, encryption, authentication, or private files.

---

## What does it cover?

1. **Threat Modeling & Data Inventory**:
   - Explicitly catalogs what threats are in scope (lost device, log leaks, casual snooping) and out of scope (device root compromise, nation-state physical attacks).
   - Sensitive data inventory detailing what requires protection and where it lives.
2. **Platform Controls**:
   - `FLAG_SECURE` for preventing screenshots and screen-recording on sensitive screens.
   - `EncryptedSharedPreferences` / Android Keystore for secure key storage.
   - Biometric authentication (`BiometricPrompt`) with strong crypto integration.
   - Disabling `android:allowBackup` or creating custom backup rules.
3. **OWASP Mobile Top 10 Checklist**:
   - M1 to M10 review checklist to verify before shipping production builds.
4. **Data Retention & Purge**:
   - Implementation of "Delete all data" functionality to wipe databases, caches, and secure storage cleanly.
5. **Security Verification**:
   - Verifying `android:debuggable=false`, testing crypto roundtrips, and ensuring zero leaks in logs.

---

## When to Use It

- Any project handling passwords, tokens, financial data, health records, or private encrypted files.
- Filling out the `security.md` blueprint when adopting the `Sensitive Data Extension` profile.
