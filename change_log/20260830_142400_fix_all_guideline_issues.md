# Change Log: Fix All Identified Guideline Issues

**Date:** 2026-08-30 14:24:00 (Local Time)  
**Plan:** `../plans/20260830_142400_fix_all_guideline_issues.md`  
**Scope:** Resolution of build artifact output paths, release installation command accuracy, related document cross-references, model fallback behavior, and Gradle signing DSL standardization.

---

## 1. Summary of Changes

Executed the approved plan to fix all identified issues across the Kotlin Guidelines repository:
1. Standardized all build output paths to include the `app/` module prefix (`app/build/outputs/...`).
2. Updated the release APK installation command in `release_process.md` to `adb install -r app/build/outputs/apk/release/app-release.apk` for accurate execution on non-debuggable variants.
3. Added `guidelines/guideline.md (submodule)` to the related documents section (§22) in `architecture.md`.
4. Refined `parseStringMap` in `guideline.md` so that missing or non-object `details` JSON fields safely fall back to `fallback.details`.
5. Standardized `storeFile` resolution in `kotlin_build_configuration_guide.md` and `kotlin_project_engineering_standard.md` to use idiomatic Gradle Kotlin DSL (`rootProject.file(...)`).

---

## 2. Files Modified

- **`architecture.md`**:
  - Updated merged manifest permission verification command in §7 to point to `app/build/outputs/apk/release/<apk>`.
  - Added `guidelines/guideline.md (submodule)` to §22 (Related Documents).
- **`security.md`**:
  - Updated mapping directory path in §8.1 to `app/build/outputs/mapping/release/`.
  - Updated bash and PowerShell `aapt2` APK verification commands in §8.2 to `app/build/outputs/apk/release/app-release.apk`.
- **`release_process.md`**:
  - Updated mapping file path in §6.1 and §9 to `app/build/outputs/mapping/release/mapping.txt`.
  - Updated release installation command in §9 from `./gradlew installRelease` to `adb install -r app/build/outputs/apk/release/app-release.apk`.
- **`guideline.md`**:
  - Updated `parseStringMap` inside `AppConfig.fromJson` to return `fallback.details` when `raw` is null.
- **`kotlin_build_configuration_guide.md`**:
  - Standardized `storeFile` resolution in §Signing Configuration to `rootProject.file(keystoreProps.getProperty("storeFile", "release.jks"))`.
- **`kotlin_project_engineering_standard.md`**:
  - Standardized `storeFile` resolution in §5.5 to `rootProject.file(keystoreProps.getProperty("storeFile", "release.jks"))`.
- **`plans/20260830_142400_fix_all_guideline_issues.md`**:
  - Updated plan status to Approved.

---

## 3. Privacy & Standards Verification

- All paths use relative repository paths only.
- Zero local system details (no user names, computer names, local drive paths, or network details).
- Zero secrets or credentials included.
