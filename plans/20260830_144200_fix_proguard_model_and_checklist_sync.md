# Implementation Plan: Fix ProGuard Rules, Room Entity Column Info, Long Type Safety, and Guideline Section Sync

**Date:** 2026-08-30 14:42:00 (Local Time)  
**Status:** Approved  
**Scope:** Update ProGuard DAO retention rule, align `Todo` entity column annotations in engineering standard, ensure explicit Long literal for `PackageInfoFlags`, add `secrets` plugin entry to version catalog snippet, and synchronize section checklist with template in `CLAUDE_MD_GUIDELINE.md` and `AGENTS_MD_GUIDELINE.md`.

---

## 1. Issues & Rationale

1. **Room DAO ProGuard Rule Precision:**
   - *Issue:* `kotlin_build_configuration_guide.md` uses `-keep @androidx.room.Dao interface * { *; }`. If a project uses abstract class DAOs (`abstract class BaseDao<T>`), they are excluded from this rule and could be stripped or obfuscated by R8.
   - *Fix:* Change to `-keep @androidx.room.Dao class * { *; }`, which matches both classes and interfaces in ProGuard syntax.

2. **Room Entity Column Mapping Consistency:**
   - *Issue:* In `kotlin_project_engineering_standard.md` §4.6, the `Todo` data class omits `@ColumnInfo(name = "...")`, whereas §12.3 and §13.1 assume `snake_case` columns (`created_at`, `is_completed`).
   - *Fix:* Add `@ColumnInfo(name = "is_completed")`, `@ColumnInfo(name = "created_at")`, and `@ColumnInfo(name = "completed_at")` in §4.6.

3. **Type Safety in `PackageManager.PackageInfoFlags.of()`:**
   - *Issue:* In `guideline.md` §1.1 `ConfigService.kt`, `PackageManager.PackageInfoFlags.of(0)` passes an integer literal to a method expecting `long`.
   - *Fix:* Update to `PackageManager.PackageInfoFlags.of(0L)`.

4. **Version Catalog Plugin Declaration for Secrets Plugin:**
   - *Issue:* `kotlin_build_configuration_guide.md` shows `alias(libs.plugins.secrets)` in the Secrets plugin section, but the version catalog sample omitted the corresponding `secrets` plugin declaration.
   - *Fix:* Add `secrets` version and plugin declaration to the sample version catalog in `kotlin_build_configuration_guide.md`.

5. **Section Checklist Synchronization with Template:**
   - *Issue:* `CLAUDE_MD_GUIDELINE.md` and `AGENTS_MD_GUIDELINE.md` §2 and §3 checklist list 16 sections, but the §4 fill-in-the-blanks template includes dedicated sections for *Package naming* and *String resources*.
   - *Fix:* Update §2 and §3 in both guideline files to explicitly enumerate *Package naming* and *String resources* so that §2, §3, and §4 have exact 1-to-1 correspondence.

---

## 2. Proposed File Modifications

1. `kotlin_build_configuration_guide.md`:
   - Update `-keep @androidx.room.Dao interface *` to `-keep @androidx.room.Dao class *`.
   - Add `secrets` plugin and version to the sample version catalog.
2. `kotlin_project_engineering_standard.md`:
   - Update `Todo` entity in §4.6 to include `@ColumnInfo` annotations.
3. `guideline.md`:
   - Update `PackageInfoFlags.of(0)` to `PackageInfoFlags.of(0L)` in `ConfigService.kt`.
4. `CLAUDE_MD_GUIDELINE.md`:
   - Update §2 canonical section order and §3 checklist table to include `Package naming` and `String resources`.
5. `AGENTS_MD_GUIDELINE.md`:
   - Update §2 canonical section order and §3 checklist table to include `Package naming` and `String resources`.

---

## 3. Verification Plan

- Inspect all modified files to ensure strict adherence to simple English, Markdown link correctness, and code snippet accuracy.
- Confirm zero local machine details or credentials in all modified files.
