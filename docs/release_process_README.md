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
4. **Post-Release & Rollback**:
   - Documenting release evidence and emergency hotfix / rollback procedures.

---

## When to Use It

- Preparing any build that will be sent to external testers, QA, or submitted to the Google Play Store / GitHub Releases.
