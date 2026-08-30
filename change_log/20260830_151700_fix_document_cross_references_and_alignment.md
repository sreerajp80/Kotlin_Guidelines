# Change Log: Fix Document Cross-References, Dual Instruction Parity, and Explainer Sections

**Date:** 2026-08-30
**Plan Reference:** `plans/20260830_151600_fix_document_cross_references_and_alignment.md`
**Scope:** Updated section cross-reference in `release_process.md`, paired root instruction references in `README.md`, added root instruction files to `architecture.md` related documents, and added missing section entries to `docs/architecture_README.md`.

---

## Changes Made

1. **`release_process.md`:**
   - Corrected the `release` row pointer in §5 (*Environment And Build Matrix*) from `See section 8 for full commands` to `See section 9 for full commands`.

2. **`README.md`:**
   - Updated the template explanation paragraph to state `referenced from its CLAUDE.md and AGENTS.md;`, ensuring consistency with root instruction requirements across the repository.

3. **`architecture.md`:**
   - Added `- `../CLAUDE.md`` and `- `../AGENTS.md`` to §22 (*Related Documents*).

4. **`docs/architecture_README.md`:**
   - Updated the section priority table under *What should you fill out BEFORE starting the project?* to include §12 (*Dependency Injection*), §16 (*UI System*), §19 (*Operational Constraints*), and §22 (*Related Documents*), ensuring all 22 sections are mapped.

---

## Verification

- Inspected all edited files to confirm valid relative paths, clear wording, and zero local system details.
