# Plan: Record the Owner's Review of the Four Glossary Items

**Status:** completed (see `change_log/20260913_192000_record-owner-review-of-terms.md`)

**Date:** 2026-09-13

## 1. What the issue is

The change log `change_log/20260913_152000_trilingual-about-badge-playstore-tooltips.md` lists four
Malayalam and Sanskrit items under "needs native-reader review":

| # | Meaning | Malayalam | Sanskrit |
|---|---|---|---|
| 1 | About (title) | ആപ്പിനെക്കുറിച്ച് | विषयपरिचयः |
| 2 | Exit (button) | പുറത്തുകടക്കുക | निष्क्रम्यताम् |
| 3 | Exit (title, label) | പുറത്തുകടക്കൽ | निष्क्रमणम् |
| 4 | Rule: direction words on buttons stay nominal | — | प्रत्यागमनम्, अग्रिमम्, पूर्वम्, अधिकम् |

On 2026-09-13 the repository owner, a fluent reader, checked all four and confirmed they are correct.
The records in this repository still say the review is pending.

The Flutter guidelines repository records the same review, but its record has three problems:
it names no reviewer, it wrongly calls the Sanskrit `विवरणम्` "Description" (the glossary uses
`विवरणम्` for Details and `वर्णनम्` for Description), and two lines still say "still waiting for review".

## 2. Files to be changed

### This repository (Kotlin)

| File | Change |
|---|---|
| `change_log/20260913_152000_trilingual-about-badge-playstore-tooltips.md` | Section 4: add a note that all four items were reviewed and confirmed by the repository owner (a fluent reader) on 2026-09-13; keep the table as the record of what was reviewed |
| `change_log/` | New change log for this change |

The guideline documents themselves do not change: the words are already in the glossary, and the
review rule stays for future terms.

### Flutter guidelines repository (same owner decision, record fixes only)

| File | Change |
|---|---|
| `change_log/20260913_191000_flutter-parity-test-and-fluent-reader-review.md` | Name the reviewer as "the repository owner (fluent reader)"; fix `विवरणम्` (Description) → `वर्णनम्` (Description), noting `विवरणम्` is Details; change the `निष्क्रम्यताम्` grammar label from "कर्मणि लोट्" to "भावे लोट्" (the verb takes no object) |
| `plans/20260913_190000_flutter-parity-test-and-fluent-reader-review.md` | Same two corrections |
| `change_log/20260913_163000_flutter-l10n-parity-and-fixes.md` | Remove the stale "which are also still waiting for review" sentence; name the reviewer |

## 3. Out of scope

- No glossary words change.
- No guideline rules change.
- Committing either repository (only on request).

## 4. After implementation

Write `change_log/20260913_hhMMss_record-owner-review-of-terms.md` in this repository referencing
this plan, and set this plan's `Status:` to `completed`.
