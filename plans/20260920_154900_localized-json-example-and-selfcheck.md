# Plan: Localized JSON Example and Self-Check Update

**Status:** Completed

## Issue

The JSON example in `guideline.md` §1.1 currently shows `author`, `aiUsed`, and `ideUsed` as plain strings (e.g., `"author": "Your Name"`). The Flutter Guidelines already updated these to use full `{"en", "ml", "sa"}` locale maps with transliterations. The Kotlin Guidelines need the same update for parity.

Also, the self-check lists in `AGENTS_MD_GUIDELINE.md` §9 and `CLAUDE_MD_GUIDELINE.md` §8 do not have an explicit check item about specifying all three languages in localization rules. The Flutter version already has this item.

## Proposed Changes

### A. `guideline.md` — §1.1 JSON example and localization rules (lines 46–88)

#### A1. Update the JSON code block (lines 46–72)

Replace the current JSON example with a concrete example using full locale maps for `appName`, `description`, `author`, `license`, `aiUsed`, and `ideUsed`. Only `email`, `version`, and `build` stay as plain strings.

New JSON:

```json
{
  "appName": {
    "en": "SreerajP PDF App",
    "ml": "ശ്രീരാജ് പി പിഡിഎഫ് ആപ്പ്",
    "sa": "श्रीराजः पी पीडीएफ् अनुप्रयोगः"
  },
  "description": {
    "en": "One-line description of what the app does.",
    "ml": "ആപ്പ് എന്തു ചെയ്യുന്നു എന്നതിന്റെ ഒറ്റവരി വിവരണം.",
    "sa": "एतत् अनुप्रयोगः किं करोति इति एकपङ्क्तिवर्णनम्।"
  },
  "version": "1.0.0",
  "build": "1",
  "details": {
    "author": {
      "en": "Sreeraj P",
      "ml": "ശ്രീരാജ് പി",
      "sa": "श्रीराजः पी"
    },
    "email": "sreerajp@zohomail.in",
    "license": {
      "en": "All libraries used are open source.",
      "ml": "ഉപയോഗിച്ച എല്ലാ ലൈബ്രറികളും ഓപ്പൺ സോഴ്സ് ആണ്.",
      "sa": "सर्वाणि प्रयुक्तानि पुस्तकालयानि मुक्तस्रोतानि सन्ति।"
    },
    "aiUsed": {
      "en": "Anthropic Claude / Google Gemini",
      "ml": "ആന്ത്രോപിക് ക്ലോഡ് / ഗൂഗിൾ ജെമിനി",
      "sa": "आन्त्रोपिक् क्लोड् / गूगल् जेमिनि"
    },
    "ideUsed": {
      "en": "Visual Studio Code / Antigravity",
      "ml": "വിഷ്വൽ സ്റ്റുഡിയോ കോഡ് / ആന്റിഗ്രാവിറ്റി",
      "sa": "विश्वल् स्टुडियो कोड् / आन्टिग्राविटि"
    }
  }
}
```

> Note: The `sa` transliterations for `aiUsed` and `ideUsed` use Devanagari script (not Malayalam script), matching the Flutter Guidelines' corrected version.

#### A2. Update the bullet-point rules below the JSON (lines 74–88)

Replace the current localization rules with clearer rules that match the Flutter Guidelines:

- `appName`, `description`, `version`, `build` are required top-level fields.
- `details` is a free map. Add or remove rows as needed; the About screen renders each entry as a labelled row.
- **Only technical, non-display values** that are never shown as user-facing text — such as email addresses, URLs, version strings, and build numbers — MAY be plain strings.
- **`appName`, `author`, `aiUsed`, and `ideUsed` MUST use the full locale map** `{"en": …, "ml": …, "sa": …}` with transliterations in Malayalam and Sanskrit. These are display values that appear on screen and must be readable in each script.
- Sentences, prose, and anything a user reads as descriptive text MUST use the locale map with all three languages filled in (§3, engineering standard §8).
- **Detail keys are identifiers, not labels.** Use `lowerCamelCase` keys (`author`, `aiUsed`); the visible label comes from `about_detail_<id in snake_case>` in `strings.xml` (`about_detail_author`, `about_detail_ai_used`) so the label itself is translated. See §1.3.
- Keep `version` and `build` in sync with `versionName` / `versionCode` in `build.gradle.kts`. `ConfigService.loadAndVerify` logs a non-fatal debug note if they drift (see the `ConfigService` rules below).

---

### B. `AGENTS_MD_GUIDELINE.md` — §9 self-check (line 374)

Add a new checklist item after the existing line about the three mandatory languages:

```
- [ ] Localization rules explicitly specify all three: English (`en`), Malayalam (`ml`), and Sanskrit (`sa`). Never drop Sanskrit.
```

This goes after the line that currently reads:
> `- [ ] The three mandatory languages are named: English, Malayalam, Sanskrit — with string parity across values/, values-ml/, values-sa/.`

---

### C. `CLAUDE_MD_GUIDELINE.md` — §8 self-check (line 375)

Add the same new checklist item after the same existing line:

```
- [ ] Localization rules explicitly specify all three: English (`en`), Malayalam (`ml`), and Sanskrit (`sa`). Never drop Sanskrit.
```

---

## Files changed

| File | Change |
|---|---|
| `guideline.md` | Update JSON example (§1.1) with full locale maps for `appName`, `description`, `author`, `license`, `aiUsed`, `ideUsed`. Update the bullet-point rules. |
| `AGENTS_MD_GUIDELINE.md` | Add localization self-check item in §9. |
| `CLAUDE_MD_GUIDELINE.md` | Add localization self-check item in §8. |

## Verification

- Read all three files after editing to confirm correct markdown formatting.
- Confirm JSON in the example is valid.
- Confirm parity with the Flutter Guidelines version.
