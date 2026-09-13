# Change Log: Sanskrit Gate — Catch Hindi Words In Markdown Help Files

**Date:** 2026-09-13

**Plan:** [plans/20260913_183600_sanskrit-gate-markdown-edges.md](../plans/20260913_183600_sanskrit-gate-markdown-edges.md)

## 1. What changed

| File | Change |
|---|---|
| `kotlin_project_engineering_standard.md` | §8.5.1 `scripts/check_sanskrit.sh`: the word edges now also include `<` before a word, `>` after a word, and the Markdown marks `*`, `_`, backtick, `#`, `|`, `:`, `;`, `~`, `-`. The script comment lists the edges. The single self-test is now a loop over five samples (XML string, JSON/ARB-style string, Markdown bold, code span, italic). The note below the script describes the new edges and gives a Markdown example. |

The pattern is now identical to the one in the Flutter guidelines, so both guideline sets catch the
same Hindi words in the same places.

## 2. Why

Hindi words wrapped in Markdown (`यह **है**`, ``यह `है` ``, `वह *था*`) in a `*_sa.md` help page
passed the old check, because Markdown marks were not word edges.

## 3. Verification

- Before the change, the new pattern was run with GNU grep on 20 sample lines: all 11 Hindi samples
  were caught, and all 9 Sanskrit samples (including `होम` and `थाली`, which only start like Hindi
  words) passed.
- After the change, the script was copied out of the standard exactly as written and run on a small
  sample app with `values-sa/strings.xml`, `app_config.json` and `help_sa.md` (section 4).
- The `PATTERN` line was compared by script with the Flutter guidelines: identical.

## 4. Test run of the script as written

| Case | Expected | Result |
|---|---|---|
| Sanskrit-only files (`रक्ष्यताम्`, the badge line, `यथा तथा`, `**पुनःस्थाप्यताम्**`, `` `स्थानम्` ``, `_अस्ति_`) | exit 0 | exit 0 ✅ |
| `यह **है**` in `help_sa.md` | exit 1 | exit 1, line reported ✅ |
| `<string name="x">है</string>` in `values-sa/strings.xml` | exit 1 | exit 1, line reported ✅ |
