# Implementation Plan: Fix Document Cross-References, Dual Instruction Parity, and Explainer Sections

**Status:** Implemented
**Date:** 2026-08-30
**Scope:** Correct the release command section pointer in `release_process.md`, align root instruction references in `README.md`, add root instruction links in `architecture.md`, and complete the section mappings in `docs/architecture_README.md`.

---

## Issues

1. **`release_process.md` (§5):**
   - Line 58 references "See section 8 for full commands", but Section 8 is the Release Checklist. Android build commands are located under Section 9.

2. **`README.md` (Section *These are templates*):**
   - Line 11 states "...referenced from its `CLAUDE.md`" omitting `AGENTS.md`, which is inconsistent with all other guideline documents that require both.

3. **`architecture.md` (§22):**
   - Section 22 lists related docs including `../README.md` and submodule paths, but omits `../CLAUDE.md` and `../AGENTS.md`.

4. **`docs/architecture_README.md` (Section *What should you fill out BEFORE starting the project?*):**
   - The priority table omits §12 (*Dependency Management And Injection*), §16 (*UI System*), §19 (*Operational Constraints*), and §22 (*Related Documents*).

---

## Proposed Fixes

### 1. `release_process.md`
- In §5 (line 58), update `See section 8 for full commands` to `See section 9 for full commands`.

### 2. `README.md`
- In Section *These are templates* (line 11), update `referenced from its `CLAUDE.md`;` to `referenced from its `CLAUDE.md` and `AGENTS.md`;`.

### 3. `architecture.md`
- In §22 *Related Documents* (lines 354–361), add `- `../CLAUDE.md`` and `- `../AGENTS.md``.

### 4. `docs/architecture_README.md`
- In the table under *What should you fill out BEFORE starting the project?*, add rows for:
  - `🟡 Soon` | **§12 Dependency Injection** | DI approach (Hilt vs manual) before writing services
  - `🟡 Soon` | **§16 UI System** | Theme tokens, color palettes, and accessibility constraints
  - `🟡 Soon` | **§19 Operational Constraints** | Performance budget, memory targets, and team conventions
  - `🟢 Later` | **§22 Related Documents** | Sibling and submodule references

---

## Verification Plan

- Inspect `release_process.md`, `README.md`, `architecture.md`, and `docs/architecture_README.md`.
- Verify all relative paths and section numbers match accurately.
- Verify no absolute paths or machine details are introduced.
