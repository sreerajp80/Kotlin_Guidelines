# Plan: Fix Documentation Cross-References and Guideline Alignment

**Status:** Approved

## 1. Issue Description

A critical analysis identified a few minor gaps in documentation cross-referencing and ecosystem alignment:

1. **`kotlin_project_engineering_standard.md` §21.1:** The "Required Documents For App Repositories" table lists `CLAUDE.md`, `AGENTS.md`, `README.md`, `docs/GUIDELINES_MANIFEST.md`, `docs/architecture.md`, `docs/release_process.md`, `plans/`, and `change_log/`. However, it does not mention `DOCS_FOLDER_GUIDELINE.md` for governing documentation files created in `docs/`.
2. **`architecture.md` §22:** The "Related Documents" list in the template blueprint omits `guidelines/DOCS_FOLDER_GUIDELINE.md` under submodule references.
3. **`docs/architecture_README.md`:** The "How the documents work together" summary table mentions only 5 technical documents, omitting a reference to the companion guidelines (`guideline.md`, `CLAUDE_MD_GUIDELINE.md`, `AGENTS_MD_GUIDELINE.md`, `DOCS_FOLDER_GUIDELINE.md`).

## 2. Proposed Changes

### File 1: `kotlin_project_engineering_standard.md`
- In §21.1, add `docs/` files following `DOCS_FOLDER_GUIDELINE.md` to the required documents table.

### File 2: `architecture.md`
- In §22 ("Related Documents"), add `guidelines/DOCS_FOLDER_GUIDELINE.md` (submodule) to the list.

### File 3: `docs/architecture_README.md`
- Update section "How the documents work together" to clarify how the full suite of documents operates as a unified system.

## 3. Verification Plan
- Validate that all markdown cross-links remain valid.
- Verify privacy rules and relative paths in all modified files.
