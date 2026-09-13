# Change Log: Trilingual Apps (EN/ML/SA), "Made with ❤️ from India" About Badge, Play Store Readiness, Tooltips

**Date:** 2026-09-13

**Plan:** [plans/20260913_150000_trilingual-about-badge-playstore-tooltips.md](../plans/20260913_150000_trilingual-about-badge-playstore-tooltips.md) (revision 3)

## 1. Summary

The Kotlin guidelines now enforce eight rules for every app, ported from the Flutter guidelines
and adapted to Kotlin + Jetpack Compose:

1. Every About screen ends with the same "Made with ❤️ from India" badge.
2. Every app ships English, Malayalam and Sanskrit. It starts in the system language, and the user
   can change the language in Settings.
3. Every app is always ready for Google Play.
4. Sanskrit text is real Sanskrit, not Hindi.
5. Every feature works in all three languages.
6. Every text the app draws follows the chosen language.
7. Menu, button, label, tab and tooltip text is short in all three languages.
8. Every icon-only button has a localized tooltip.

## 2. Files changed

| File | What changed |
|---|---|
| `guideline.md` | §1.1 About JSON now supports `{"en","ml","sa"}` language maps, with a new `LocalizedText` class and an updated `AppConfig`; the broken "see §1.5" reference now points to the `ConfigService` rules. §1.2 Pattern B may hold only language-independent values. §1.3 About rows use localized `about_detail_<id>` labels, with a reference Composable. New §1.4 badge: fixed strings in three languages, red inline heart icon, reference `MadeWithLove` Composable, checks. §3 layout adds `l10n/`, the three `values*` folders, `resources.properties`, `scripts/`, plus rules for languages, Sanskrit, the picker, short labels, tooltips and Play readiness. §4 checklist extended. |
| `kotlin_project_engineering_standard.md` | §7.3 requires tooltips (example uses `tooltip_` instead of `cd_`). New §7.7: tooltip rules, `TooltipIconButton` wrapper, version note, two checks. §8 rewritten (8.1–8.9): build/manifest/activity setup table, string rules, formatting locales, Material 3 pickers under `sa`, plurals, fonts, TalkBack and input, in-app language picker with reference code, a background-text helper for Android 12 and older, the Sanskrit & Malayalam quality rules, CI gate and glossary, label budget with ICU4J test, per-feature completeness with parity tests, notification channels, accepted exceptions, RTL, formatting. §17.4 Noto font licence note. New §18.7 localization tests. New §19.4 CI gates (Sanskrit, icon buttons, lint suppressions). §22 and §23.1 updated; the "single-language app" wording is removed. |
| `kotlin_build_configuration_guide.md` | New "Languages" section: locale filter, `generateLocaleConfig`, `bundle.language.enableSplit = false`, lint errors, test system property, dependencies and version catalog entries, `resources.properties`, AppCompat manifest service, `AppCompatActivity` + AppCompat theme, how to verify the generated locale config. targetSdk row points to the Play gate. |
| `release_process.md` | §1 note that the Play build-time items apply to every app. §8 new "Localization" and "Google Play Store Readiness" checklist blocks, plus a badge check. §9 steps for the language gates and the Play gate. New §9A Google Play readiness gate (9A.1–9A.8). |
| `architecture.md` | §16 new "Localization" record table and tooltip line; §19 store constraint line. |
| `CLAUDE_MD_GUIDELINE.md`, `AGENTS_MD_GUIDELINE.md` | Section 11 renamed to "Localization rules"; template block rewritten; four self-check lines. The two files are identical in these parts (checked with `diff`). |
| `docs/kotlin_project_engineering_standard_README.md` | Core Baseline list updated; Flutter/Kotlin table rows for languages, the language switch, tooltips; new plain-English "The Three Languages" section. |
| `docs/release_process_README.md` | New items for Play readiness and language checks. |
| `docs/architecture_README.md`, `README.md`, `GUIDELINES_MANIFEST.md`, `CLAUDE.md`, `AGENTS.md` | `guideline.md` description mentions the badge and the three languages. |

## 3. Differences from the plan

| Plan item | What was done instead | Why |
|---|---|---|
| N6 said one About row label (`प्रयुक्ता कृत्रिमबुद्धिः`, 24 characters) is over budget | The glossary was measured with the stricter count described in §8.6 (vowel signs and viramas join the letter before; every consonant counts). The longest Malayalam or Sanskrit term is 13, so nothing is over budget. The Flutter note was replaced. The `about_detail_` exemption is kept, so real About labels may still wrap. | The "24" in the Flutter note counted code points, not visible characters. |
| N9: a label form and a button form for Back, Next, Previous, Close, Exit, More | Close and Exit get separate button (imperative) and label (noun) forms. Back, Next, Previous and More are direction words and may keep their existing form on buttons, like Yes / No. | Making new Sanskrit button forms for direction words would add unreviewed words. The new rule is written in §8.5.1. |
| N19: never keep a second copy of the language | Rule kept, with one controlled exception: a read-only background mirror (§8.4.2). The mirror is written by `LanguageRepository.set` and overwritten from AppCompat in `MainActivity.onCreate`. | Below Android 13, `getApplicationLocales()` is empty until the first Activity is created, so a notification from a cold-started worker could not know the language (R3). |
| N15: a UI test confirms the heart is present | A JVM test checks `%1$s` appears exactly once in all three files, and the About screenshot test runs in all three locales. | The inline heart icon is hidden from semantics (TalkBack reads the words), so a UI test cannot find it. |
| §19 order | The gates became §19.4 and Pre-Commit stays §19.3. | Keeps existing numbering. |
| Not in plan | §8.9 notes that `java.time` needs API 26 (desugaring with `minSdk 24`). §8.3.5 marks `KeyboardOptions(hintLocales)` as Compose 1.8+. §8.3.2 says to resolve app strings outside the picker wrapper. | Found while writing the reference code. |

## 4. Needs native-reader review

> **Reviewed on 2026-09-13.** The repository owner, a fluent reader, checked all four items below
> and confirmed they are correct. Recorded in
> [20260913_192000_record-owner-review-of-terms.md](20260913_192000_record-owner-review-of-terms.md).

These Malayalam and Sanskrit terms were **added or changed** by this work. Under the new review
rule (§8.5.4), a fluent reader must confirm them before apps use them:

| Meaning | Malayalam | Sanskrit | Change |
|---|---|---|---|
| About (title, everywhere) | ആപ്പിനെക്കുറിച്ച് | विषयपरिचयः | Bad → Good table now matches the About glossary (was `വിവരണം` / `परिचयः`) |
| Exit (button) | പുറത്തുകടക്കുക | निष्क्रम्यताम् | New Sanskrit imperative form |
| Exit (title, label) | പുറത്തുകടക്കൽ | निष्क्रमणम् | New Malayalam noun form |
| Rule: direction words on buttons | — | `प्रत्यागमनम्`, `अग्रिमम्`, `पूर्वम्`, `अधिकम्` stay nominal | New form rule |

All other glossary terms, the Bad → Good table and the badge strings were copied unchanged from the
Flutter guidelines' final §8.5 and About badge.

## 5. Verification

- The Sanskrit gate pattern was run with GNU grep on 10 sample lines. It caught `है`, `हैं`, `है।`,
  `सेटिंग्स` and a nukta letter inside `<string>` tags. It passed `स्थाप्यताम्`,
  `पुनःस्थाप्यताम्`, `यथा तथा कथा`, `न कोऽपि दत्तांशः प्राप्तः` and the Sanskrit badge line.
- The ported §8.5 text was checked by script for leftover Flutter terms (`ARB`, `.arb`,
  `AppLocalizations`, `l10n.yaml`, `Flutter`): none.
- A repository search found no leftover "single-language" rule, `String resources` section, `cd_`
  example, or `§1.5` reference.
- Every section number referenced in `guideline.md`, `release_process.md`,
  `kotlin_build_configuration_guide.md` and `architecture.md` has a matching heading.
- The `CLAUDE_MD_GUIDELINE.md` and `AGENTS_MD_GUIDELINE.md` localization blocks and self-checks are
  identical.
- No machine paths or local system details in any changed file; no stray control characters.
- **Not verified:** the Kotlin, Gradle and shell snippets are documentation and were not compiled
  or run in an app. Material 3 tooltip function names, the AGP locale-filter property, and all Play
  policy values marked "at the time of writing" must be re-checked against current versions.

## 6. Follow-up (not done here)

The Flutter guidelines repository has the same gaps and should get its own plan: glossary conflicts
(About / Profile / Description, Exit form), the Play language split setting, heart rendering as an
emoji, visible-character counting, language checks for JSON and asset content, and the About-row
label exemption.
