# Plan: Fix Kotlin Project Guideline Issues

**Status:** Completed  
**Date:** 2026-08-30 13:50:00 (Local Time)  
**Scope:** Resolution of all functional bugs, link resolution errors, profile discrepancies, and missing files across the Kotlin Guidelines suite.

---

## 1. Issue Description

A critical analysis of the `Kotlin_Guidelines` repository identified several categories of issues:
1. **Functional API & Toolchain Incompatibilities:**
   - `guideline.md` accesses `info.longVersionCode` directly, causing runtime crashes on Android 7.0–8.1 (`minSdk 24` to API 27).
   - `guideline.md` uses Kotlin 2.2 unnamed catch syntax `catch (_: Exception)` while the toolchain is pinned to Kotlin 2.1.x.
   - `guideline.md` and `kotlin_build_configuration_guide.md` omit mandatory `buildFeatures { buildConfig = true }` required in AGP 8.x for `buildConfigField`.
   - `SimpleDateFormat` instantiation without `Locale.US` causing locale-dependent date formatting.
2. **Profile Alignment & Document Ecosystem Discrepancies:**
   - `DOCS_FOLDER_GUIDELINE.md §6` unconditionally requires `security.md` and `release_process.md` for all new projects, directly contradicting the 3-profile system (`Core Baseline`, `Production App Extension`, `Sensitive Data Extension`).
   - `DOCS_FOLDER_GUIDELINE.md` references phantom documents (`dependencies.md`, `project_structure.md`, `workflow_rules.md`) without templates, and places `implementation_plan.md` in `docs/` conflicting with `plans/`.
3. **Relative Path & Markdown Link Errors:**
   - `GUIDELINES_MANIFEST.md` lists submodule paths as `docs/guidelines/...`, which resolves to `docs/docs/guidelines/...` when opened from `docs/GUIDELINES_MANIFEST.md`.
   - `DOCS_FOLDER_GUIDELINE.md` incorrectly instructs linking to `docs/guidelines/...` instead of `guidelines/...`.
   - Sibling links in `architecture.md`, `security.md`, and `release_process.md` use redundant `docs/` prefixes.
4. **Structural & Repository Alignment:**
   - `AGENTS_MD_GUIDELINE.md §3` table points to `§5` instead of `§6` for workflow and communication rules.
   - `kotlin_project_engineering_standard.md §3.3` places `libs.versions.toml` at the project root instead of under `gradle/`.
   - Missing project-root `CLAUDE.md` and `AGENTS.md` for the `Kotlin_Guidelines` repository.

---

## 2. Proposed Changes

### 2.1 Functional Code & Gradle Fixes
- **`guideline.md`**:
  - Replace `info.longVersionCode` with `androidx.core.content.pm.PackageInfoCompat.getLongVersionCode(info)`.
  - Replace `catch (_: Exception)` with `catch (e: Exception)`.
  - Add `buildFeatures { buildConfig = true }` and imports (`java.util.Properties`, `java.text.SimpleDateFormat`, `java.util.Date`, `java.util.Locale`) to Pattern B snippet.
  - Add `Locale.US` to `SimpleDateFormat("yyyy-MM-dd HH:mm", Locale.US)`.
- **`kotlin_build_configuration_guide.md`**:
  - Add `buildFeatures { buildConfig = true }` where `buildConfigField` is used.
  - Update `SimpleDateFormat` to include `Locale.US`.
  - Standardize `storeFile` resolution in alternative signing configuration with `rootProject.file(...)`.

### 2.2 Profile & Documentation Standard Alignment
- **`DOCS_FOLDER_GUIDELINE.md`**:
  - Update §6 to strictly align with Applicability Profiles (Core Baseline vs Extensions).
  - Clarify where living topics belong (sections in `architecture.md` / `security.md` vs standalone docs).
  - Fix submodule relative link guidance to `guidelines/...`.
- **`GUIDELINES_MANIFEST.md`**:
  - Update relative paths in tables to `guidelines/...` with clarification on root vs `docs/` resolution.
- **`AGENTS_MD_GUIDELINE.md`**:
  - Update table references from `(see §5)` to `(see §6)`.
- **`kotlin_project_engineering_standard.md`**:
  - Fix tree diagram in §3.3 to show `gradle/libs.versions.toml`.

### 2.3 Blueprint Cross-References
- **`architecture.md`**, **`security.md`**, **`release_process.md`**:
  - Fix sibling links (`security.md`, `release_process.md`, `../README.md`).

### 2.4 Repository Root Instructions
- **`CLAUDE.md`** & **`AGENTS.md`**:
  - Create root guideline and instruction files for this repository (`Kotlin_Guidelines`).

---

## 3. Verification Plan
- Review all modified markdown files for formatting, syntax, link consistency, and profile alignment.
- Verify zero absolute paths or local machine secrets exist across all modified documents.
