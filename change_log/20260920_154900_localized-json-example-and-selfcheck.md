# Change Log: Localized JSON Example and Self-Check Update

**Date:** 2026-09-20

**Plan:** [../plans/20260920_154900_localized-json-example-and-selfcheck.md](../plans/20260920_154900_localized-json-example-and-selfcheck.md)

## Summary

Updated the Kotlin Guidelines for parity with the Flutter Guidelines. The JSON example in `guideline.md` now shows full `{"en", "ml", "sa"}` locale maps for all display values. The self-check lists in both `AGENTS_MD_GUIDELINE.md` and `CLAUDE_MD_GUIDELINE.md` now include an explicit localization check item.

## Files changed

| File | What changed |
|---|---|
| `guideline.md` | §1.1 JSON example updated: `appName`, `description`, `author`, `license`, `aiUsed`, and `ideUsed` now show full locale maps with concrete Malayalam and Sanskrit transliterations. Only `email`, `version`, and `build` remain as plain strings. Bullet-point rules below the JSON clarified: only technical non-display values may be plain strings; `appName`, `author`, `aiUsed`, and `ideUsed` MUST use the full locale map. Detail keys rule updated to reference `strings.xml` label convention. |
| `AGENTS_MD_GUIDELINE.md` | §9 self-check: added item — "Localization rules explicitly specify all three: English (`en`), Malayalam (`ml`), and Sanskrit (`sa`). Never drop Sanskrit." |
| `CLAUDE_MD_GUIDELINE.md` | §8 self-check: added the same localization check item. |

## Notes

- The `AppConfig` data class already used `appName: LocalizedText` — no model change was needed.
- The `sa` transliterations for `aiUsed` and `ideUsed` use Devanagari script (matching the Flutter Guidelines).
- Only relative paths used. No local system details or sensitive data present.
