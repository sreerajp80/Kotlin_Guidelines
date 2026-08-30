# Implementation Plan: Fix Code Types, Test Signatures, and Stream Handling

**Status:** Approved  
**Date:** 2026-08-30 14:34:00 (Local Time)  
**Scope:** Fix type mismatch in `TodoUiState` ViewModel and Composable examples, synchronize test invocation signatures with `TodoScreenContent`, add missing `@Entity` annotation to data model in architecture section, close `FileInputStream` safely in Gradle build guide, and modernize `PackageManager` package info retrieval in `ConfigService`.

---

## 1. Issues Identified

1. **`TodoUiState` Sealed Interface vs Constructor Mismatch in `kotlin_project_engineering_standard.md` (§4.3, §6.2, §6.3):**
   - In §4.3, `MutableStateFlow(TodoUiState())` tries to call a constructor on `TodoUiState`, which is defined as a `sealed interface` in §6.3.
   - In §6.2, `TodoScreen` accesses `uiState.todos` directly on the sealed interface instead of unwrapping the `TodoUiState.Content` state.

2. **Test Function Signatures in `kotlin_project_engineering_standard.md` (§18.5, §18.6):**
   - In §18.5 and §18.6, test snippets call `TodoScreen(todos = emptyList())`, but in §6.2 `TodoScreen` takes `(viewModel: TodoViewModel = viewModel())` while the stateless Composable accepting `todos: List<Todo>` is named `TodoScreenContent`.

3. **Missing `@Entity` Annotation in Model Example in `kotlin_project_engineering_standard.md` (§4.6):**
   - The `Todo` model example includes `@PrimaryKey(autoGenerate = true)` on `id`, but lacks the class-level `@Entity(tableName = "todos")` annotation.

4. **Unclosed `FileInputStream` in `kotlin_build_configuration_guide.md` (§Signing Configuration):**
   - The `local.properties` snippet uses `load(f.inputStream())` without closing the stream via `.use { load(it) }`.

5. **`PackageManager.getPackageInfo` Deprecation in `guideline.md` (§1.1):**
   - `getPackageInfo(packageName, 0)` is deprecated in API 33+ (Android 13+). Adding SDK version gating avoids compiler warnings.

---

## 2. Proposed Changes

### File: `kotlin_project_engineering_standard.md`
- **§4.3 (State Exposure):** Update `TodoViewModel` to initialize state with `MutableStateFlow<TodoUiState>(TodoUiState.Loading)`.
- **§4.6 (Models And Entities):** Add `@Entity(tableName = "todos")` to the `Todo` data class example.
- **§6.2 (Composable Structure):** Update `TodoScreen` to handle sealed `TodoUiState` via `when (val state = uiState)` and call `TodoScreenContent(todos = state.todos, onToggle = viewModel::toggleTodo)` when state is `TodoUiState.Content`.
- **§18.5 (Compose UI Testing):** Change test invocation from `TodoScreen(todos = emptyList())` to `TodoScreenContent(todos = emptyList(), onToggle = {})`.
- **§18.6 (Roborazzi Screenshot Tests):** Change screenshot test invocation from `TodoScreen(todos = emptyList())` to `TodoScreenContent(todos = emptyList(), onToggle = {})`.

### File: `kotlin_build_configuration_guide.md`
- **§Signing Configuration (Alternative: local.properties Pattern):** Change `if (f.exists()) load(f.inputStream())` to `if (f.exists()) f.inputStream().use { load(it) }`.

### File: `guideline.md`
- **§1.1 (ConfigService loader):** Update `loadAndVerify` to check `Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU` for `PackageInfoFlags.of(0)` with fallback to legacy `getPackageInfo(packageName, 0)`.

---

## 3. Privacy & Standards Verification

- All paths use relative repository paths only.
- Zero local system details (no user names, computer names, local drive paths, or network details).
- Zero secrets or credentials included.

---

## 4. Verification Plan

1. Verify all markdown files contain valid relative links.
2. Verify all code snippets are syntactically valid and type-safe.
3. Check that plan and change log adhere strictly to the privacy rules.
