# Change Log: Fix Kotlin Project Guideline Issues

**Date:** 2026-08-30 13:50:00 (Local Time)  
**Plan:** `../plans/20260830_135000_fix_kotlin_guideline_issues.md`  
**Scope:** Resolution of functional bugs, link resolution errors, profile discrepancies, and missing repository root instruction files.

---

## 1. Summary of Changes

Executed the approved plan to fix all identified issues across the Kotlin Guidelines suite, ensuring API safety on `minSdk 24`, full Kotlin 2.1 syntax compliance, AGP 8.x Gradle compatibility, profile alignment, and accurate Markdown link resolution.

## 2. Files Modified and Created

### Code & Guideline Refinements
- **`guideline.md`**:
  - Replaced direct `info.longVersionCode` call with `androidx.core.content.pm.PackageInfoCompat.getLongVersionCode(info)` to prevent runtime crashes on Android 7.0–8.1 (`minSdk 24` to API 27).
  - Replaced unnamed catch parameter `catch (_: Exception)` with `catch (e: Exception)` for full Kotlin 2.1 compiler compatibility.
  - Added `buildFeatures { buildConfig = true }` and missing imports (`java.util.Properties`, `java.text.SimpleDateFormat`, `java.util.Date`, `java.util.Locale`) to Pattern B Gradle configuration.
  - Added `Locale.US` to `SimpleDateFormat`.
- **`kotlin_build_configuration_guide.md`**:
  - Added `buildFeatures { buildConfig = true }` to the About-screen build date snippet.
  - Standardized `storeFile` resolution with `rootProject.file(...)`.
  - Added `Locale.US` to `SimpleDateFormat`.

### Standard & Profile Alignment
- **`DOCS_FOLDER_GUIDELINE.md`**:
  - Updated §6 to align strictly with the 3 Applicability Profiles (`Core Baseline`, `Production App Extension`, `Sensitive Data Extension`).
  - Clarified that living architecture topics reside in `architecture.md`, workflow rules in root instructions, and individual change plans in `plans/`.
  - Fixed submodule relative linking example to `guidelines/security.md`.
- **`GUIDELINES_MANIFEST.md`**:
  - Updated table paths to `guidelines/...` so relative links resolve correctly when opened from `docs/GUIDELINES_MANIFEST.md`.
- **`AGENTS_MD_GUIDELINE.md`**:
  - Fixed checklist table references to point to §6 for workflow and communication rules.
- **`kotlin_project_engineering_standard.md`**:
  - Updated project tree layout in §3.3 to show `gradle/libs.versions.toml`.

### Blueprint Cross-Reference Corrections
- **`architecture.md`**: Fixed §22 related document links (`../README.md`, `GUIDELINES_MANIFEST.md`, submodule links).
- **`security.md`**: Fixed patch release process reference to sibling `release_process.md`.
- **`release_process.md`**: Fixed OWASP checklist reference to sibling `security.md`.

### Repository Root Instructions
- **`CLAUDE.md`**: Created project-root guidelines for Claude Code sessions in `Kotlin_Guidelines`.
- **`AGENTS.md`**: Created project-root guidelines for AI coding assistants and open LLM agents in `Kotlin_Guidelines`.

## 3. Privacy & Standards Verification
- All file paths use relative repository paths only.
- Zero local system details (no user names, computer names, local drive paths, or network details).
- Zero secrets or credentials included.
