# Explainer: Release Process

## What is this document?

`release_process.md` is a **step-by-step runbook** for taking a Kotlin Android app from development to a signed, hardened, verified production release.

---

## Key Parts of the Runbook

1. **Pre-Flight Hardening**:
   - Verify R8 code shrinking is enabled (`isMinifyEnabled = true`).
   - Archive the generated `mapping.txt` (critical for de-obfuscating crash traces).
   - Verify `android:debuggable=false` via `aapt2`.
   - Validate APK / AAB size against the size budget.
2. **Release Checklist**:
   - Comprehensive checklist covering code quality (lint, tests), performance (jank profiling, startup time), security (permissions, OWASP), and metadata.
3. **Execution Steps**:
   - Clean git branch, bump `versionCode` and `versionName` in `app/build.gradle.kts`.
   - Run lint (`./gradlew lint`) and unit tests (`./gradlew testDebugUnitTest`).
   - Assemble APK (`./gradlew assembleRelease`) or bundle (`./gradlew bundleRelease`).
   - Verify signature and test artifact on a real device/emulator.
   - Tag the release in git (`git tag v1.0.0`).
4. **Google Play Readiness (§9A)**:
   - Every app must always be ready to publish on Google Play, even before its first release.
   - Build-side items (app id, target API level, App Bundle with the language split turned off, permissions) apply from day one.
   - Play Console items (privacy policy, Data safety form, store images, English and Malayalam listings, testing track) are done before the first upload.
   - Sanskrit cannot be a store listing language, but it still ships inside the app.
5. **Language Checks**:
   - Before release, open the app in English, Malayalam and Sanskrit, run the language scripts, and confirm the About screen badge.
6. **Post-Release & Rollback**:
   - Documenting release evidence and emergency hotfix / rollback procedures.

---

## When to Use It

- Preparing any build that will be sent to external testers, QA, or submitted to the Google Play Store / GitHub Releases.
