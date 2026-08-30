# Change Log: Fix ProGuard DAO Rule, Room Entity Annotations, Type Safety, and Guideline Checklists

**Date:** 2026-08-30 14:42:00 (Local Time)  
**Plan:** `../plans/20260830_144200_fix_proguard_model_and_checklist_sync.md`  
**Scope:** Update Room DAO ProGuard keep rule, align `Todo` entity column annotations, pass explicit Long literal in `ConfigService`, add `secrets` plugin entry to the version catalog snippet, and synchronize section checklists in root instruction guidelines.

---

## 1. Summary of Changes

Executed the approved plan to fix ProGuard rules, model annotations, type safety, and checklist alignment:
1. Updated Room DAO ProGuard rule in `kotlin_build_configuration_guide.md` from `interface *` to `class *` so that both interface and abstract class DAOs are retained.
2. Added `secrets` plugin version and catalog entry to `libs.versions.toml` snippet in `kotlin_build_configuration_guide.md`.
3. Added `@ColumnInfo(name = "...")` annotations to the `Todo` data class snippet in `kotlin_project_engineering_standard.md` (§4.6) to ensure consistent `snake_case` column naming matching §12.3 and §13.1.
4. Updated `PackageManager.PackageInfoFlags.of(0)` to `PackageManager.PackageInfoFlags.of(0L)` in `guideline.md` (§1.1) for type accuracy.
5. Synchronized §2 canonical section order and §3 checklist tables in `CLAUDE_MD_GUIDELINE.md` and `AGENTS_MD_GUIDELINE.md` to explicitly enumerate *Package naming* and *String resources*, establishing 1-to-1 parity with the §4 fill-in-the-blanks templates.

---

## 2. Files Modified

- **`kotlin_build_configuration_guide.md`**:
  - Updated `-keep @androidx.room.Dao class * { *; }` in §R8 / ProGuard Configuration.
  - Added `secrets` plugin entry in §Version Catalog.
- **`kotlin_project_engineering_standard.md`**:
  - Added `@ColumnInfo` annotations to `Todo` entity fields in §4.6.
- **`guideline.md`**:
  - Updated `PackageInfoFlags.of(0L)` in `ConfigService.kt` in §1.1.
- **`CLAUDE_MD_GUIDELINE.md`**:
  - Updated §2 and §3 checklist to include *Package naming* (§4) and *String resources* (§11).
- **`AGENTS_MD_GUIDELINE.md`**:
  - Updated §2 and §3 checklist to include *Package naming* (§4) and *String resources* (§11).
- **`plans/20260830_144200_fix_proguard_model_and_checklist_sync.md`**:
  - Marked plan as Approved.

---

## 3. Privacy & Standards Verification

- All paths use relative repository paths only.
- Zero local system details (no user names, computer names, local drive paths, or network details).
- Zero secrets or credentials included.
