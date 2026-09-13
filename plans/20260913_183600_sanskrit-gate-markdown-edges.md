# Plan: Sanskrit Gate — Catch Hindi Words In Markdown Help Files

**Status:** completed (see `change_log/20260913_190000_sanskrit-gate-markdown-edges.md`)

**Date:** 2026-09-13

## 1. What the issue is

`scripts/check_sanskrit.sh` (engineering standard §8.5.1) fails the build when a Hindi word appears
in Sanskrit text. It scans `values-sa/*.xml`, `app_config.json`, and `*_sa.*` asset files, which
includes Markdown help pages such as `help_sa.md`.

Short Hindi words (`है`, `था`, `और` …) only match when they stand between **word edges**. The
current edges are whitespace, quotes, brackets, punctuation, the daṇḍa `।`, and the XML tag edges
`>` / `<`. Markdown marks are not edges, so Hindi words wrapped in Markdown slip through. Tested
with GNU grep:

| Sample in a help file | Current result |
|---|---|
| `यह **है**` (bold) | not caught |
| ``यह `है` `` (code span) | not caught |
| `वह *था*` (italic) | not caught |

A review of the Flutter guidelines found the same gap, and it has been fixed there with the pattern
below, so both guideline sets will behave the same.

## 2. Files to be changed

| File | Change |
|---|---|
| `kotlin_project_engineering_standard.md` | §8.5.1: new `PATTERN` in `scripts/check_sanskrit.sh`; the self-test checks several samples instead of one; update the comment and the note below the script |
| `change_log/` | New change log for this change |

## 3. The fix

1. Widen the word edges:
   - Before a word: `[\s"'([{<>।,*_`#|:;~-]` or start of line.
   - After a word: `[\s"')\]}<>।,.?!*_`#|:;~-]` or end of line.
   - Adds `<` before and `>` after (HTML inside Markdown), and the Markdown marks `*`, `_`,
     backtick, `#`, `|`, `:`, `;`, `~`, `-`.
2. Replace the single self-test with a loop over five samples, one per text format:
   `"greeting": "है"`, `<string name="x">है</string>`, `यह **है**`, ``यह `है` ``, `वह *था*`.
3. Update the script comment and the note so they list the new edges.

**Tested before writing this plan** (GNU grep, 20 sample lines):

- Caught, as required (11): `"greeting": "है"`, `<string name="x">है</string>`, `यह **है**`,
  `# है`, `| है |`, ``यह `है` ``, `वह *था*`, `- है`, `है:`, `यह ठीक है।`, `सेटिंग्स`.
- Not caught, as required (9): `स्थाप्यताम्` inside XML, `यथा तथा कथा`, `**पुनःस्थाप्यताम्**`,
  `` `स्थानम्` ``, `_अस्ति_`, `न कोऽपि दत्तांशः प्राप्तः`, the Sanskrit badge line, and words that
  only start like a Hindi word (`होम`, `थाली`).

## 4. Out of scope

- The list of Hindi words itself is unchanged.
- Reviewing Malayalam and Sanskrit wording (still needs a fluent reader).

## 5. After implementation

Write `change_log/20260913_hhMMss_sanskrit-gate-markdown-edges.md` referencing this plan, and set
this plan's `Status:` to `completed`.
