# Plan: Fix All Identified Guideline Issues

**Status:** Approved  
**Date:** 2026-08-30 14:24:00 (Local Time)  
**Scope:** Resolution of artifact path discrepancies, release installation command accuracy, related document cross-referencing, model fallback consistency, and Gradle signing DSL standardization.

---

## 1. Issues and Proposed Fixes

### 1. Build Artifact Path Consistency (`app/build/...`)
- **Issue:** Several documents reference `build/outputs/...` instead of `app/build/outputs/...`, which is inaccurate for standard single-module Android applications.
- **Affected Files:**
  - `architecture.md` (Line 122: APK path in offline verification)
  - `security.md` (Line 135: mapping directory; Lines 153, 157: APK paths)
  - `release_process.md` (Line 85: mapping file path; Line 214: mapping directory)
- **Fix:** Prefix all build artifact output paths with `app/`.

### 2. Release Installation Command Accuracy
- **Issue:** `release_process.md` lists `./gradlew installRelease`, but AGP does not generate an `installRelease` task for non-debuggable release builds.
- **Affected File:** `release_process.md` (Line 232)
- **Fix:** Replace with `adb install -r app/build/outputs/apk/release/app-release.apk`.

### 3. Missing Cross-Reference in Architecture Blueprint
- **Issue:** `architecture.md §22` lists related guideline documents but misses `guidelines/guideline.md`.
- **Affected File:** `architecture.md` (Section 22)
- **Fix:** Add `- guidelines/guideline.md (submodule)` to the related documents list.

### 4. Field-by-Field Fallback in `AppConfig.fromJson`
- **Issue:** In `AppConfig.fromJson`, if `details` is missing or not a JSON object, `parseStringMap` returns `emptyMap()` rather than falling back to `fallback.details`.
- **Affected File:** `guideline.md` (Lines 104–112)
- **Fix:** Update `parseStringMap` to return `fallback.details` when `raw == null`.

### 5. Gradle DSL Keystore File Resolution Consistency
- **Issue:** `kotlin_build_configuration_guide.md` and `kotlin_project_engineering_standard.md` use `file("${rootDir}/${keystoreProps.getProperty(...)")`, whereas `rootProject.file(...)` is cleaner and standard in Gradle Kotlin DSL.
- **Affected Files:**
  - `kotlin_build_configuration_guide.md` (Line 77)
  - `kotlin_project_engineering_standard.md` (Line 310)
- **Fix:** Standardize `storeFile` assignment to `rootProject.file(keystoreProps.getProperty("storeFile", "release.jks"))`.

---

## 2. Files to Modify

| File | Target Modifications |
|---|---|
| `architecture.md` | Fix APK path in §7; add `guidelines/guideline.md` to §22 |
| `security.md` | Fix mapping path in §8.1; fix APK paths in §8.2 |
| `release_process.md` | Fix mapping paths in §6.1 and §9; update release install command to `adb install` in §9 |
| `guideline.md` | Update `parseStringMap` fallback in `AppConfig.fromJson` to `fallback.details` |
| `kotlin_build_configuration_guide.md` | Standardize `storeFile` to `rootProject.file(...)` in §Signing Configuration |
| `kotlin_project_engineering_standard.md` | Standardize `storeFile` to `rootProject.file(...)` in §5.5 |

---

## 3. Privacy & Standards Verification
- Relative repository paths only.
- Zero local system details (no user names, computer names, local drive paths, or network details).
- Zero secrets or credentials included.
