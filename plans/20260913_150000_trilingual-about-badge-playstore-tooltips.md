# Plan: Trilingual Apps (EN/ML/SA), "Made with ❤️ from India" About Badge, Play Store Readiness, Tooltips

**Status:** completed (revision 3 implemented — see
`change_log/20260913_152000_trilingual-about-badge-playstore-tooltips.md`)

**Date:** 2026-09-13

**Reference:** the sister Flutter guidelines repository already carries these rules (its plans
`20260913_125208_trilingual-about-badge-playstore-tooltips.md` and
`20260913_134700_section-8-5-sanskrit-malayalam-quality.md`). This plan ports the same rules to
Kotlin + Jetpack Compose, and changes the Flutter-only parts into the Android way of doing things.

## 0. Revision history

### 0.1 Revision 2 fixes (still in force)

| # | Problem found | Fix |
|---|---|---|
| R1 | Play installs only the phone's language resources, so an in-app switch to another language shows English. | `bundle { language { enableSplit = false } }` MUST (3.3, 3.5) |
| R2 | Library strings in ~80 languages leak (e.g. Hindi date picker under "System default"). | Locale filter to `en`, `ml`, `sa` MUST (3.3) |
| R3 | Below API 33 the per-app language reaches Activities only; notifications, workers, widgets stay in the system language. | Localized-context helper; no string lookups in ViewModels (3.3) |
| R4 | Notification channel names keep the old language. | Re-register channels (3.3) |
| R5 | Hindi-marker regex misses words between `>` and `<` in XML. | Add `>` `<` as word edges (3.3) |
| R6 | Wrong label budget. | EN 20 / ML 22 / SA 22 visible characters (3.3) |
| R7 | Glossary uses one word for two meanings, and a noun for a button. | One term per meaning (3.3) |
| R8 | Colour-emoji heart ignores text colour. | Superseded by N1 |
| R9 | Wrong `translatable="false"` example. | Endonyms, brand names, symbols (3.3) |
| R10 | "Play ready" checked only at upload. | Build-time MUSTs from day one + console items (3.5) |
| R11 | Outdated Play facts. | Updated, marked "re-check" (3.5) |
| R12 | Weak tooltip check. | Wrapper + CI gate (3.4) |
| R13 | Missing Flutter tests (copy-of-English, assets, three-locale UI tests, plurals). | Added (3.3) |
| R14 | Missing files. | Added (section 2) |
| R15 | Language change recreates the Activity. | State in ViewModel / `rememberSaveable` (3.3) |

### 0.2 Revision 3 — critical review findings

| # | Problem in revision 2 | Why it matters | Fix |
|---|---|---|---|
| N1 | **The heart glyph is not reliable.** Even without U+FE0F, many Android builds draw U+2764 from the colour emoji font, which ignores text colour. So the heart could be the wrong red, or not tintable at all. | Rule 1 needs the same look on every device. | Draw the heart as an inline `Icons.Filled.Favorite` icon (`InlineTextContent`, 1 em, tinted `#E53935`). No glyph depends on the font (3.1). |
| N2 | **`AppCompatActivity` crashes on the default Compose theme.** New Compose projects use a `android:Theme.Material.*` parent. `AppCompatActivity` throws "You need to use a Theme.AppCompat theme". | Every app following rev 2 would crash at launch. | The XML theme parent MUST be `Theme.AppCompat.DayNight.NoActionBar` (or a Material Components descendant) (3.3 §8.1). |
| N3 | **Formatting helper was not deterministic.** "Use `sa` if the device has data" is wrong: ICU often *has* a thin `sa` entry, which can give Devanagari digits or placeholder month names (`M01`). | Dates could look broken under Sanskrit on some phones. | Fixed mapping, no device test: `en` → `Locale.ENGLISH` with the system region if it is English, `ml` → `ml-IN`, `sa` → `Locale.ENGLISH`. Western digits everywhere. Use `Locale.forLanguageTag`, not the deprecated constructor (3.3 §8.3). |
| N4 | **Material 3 pickers use the configuration locale, not our helper.** `DatePicker` / `TimePicker` under `sa` can show the same broken month names or digits. | Rule 6 visible bug. | MUST verify the pickers under `sa`. If broken, wrap the picker so it formats with the helper's locale (reference snippet), or pass the locale where the Material 3 version allows it (3.3 §8.3). |
| N5 | **Grapheme counting differs between tools.** JDK 17 `java.text.BreakIterator` and Flutter's `characters` package do not split Malayalam/Devanagari conjuncts the same way. The "22" would mean different things on the two stacks. | Budget test would pass on one stack and fail on the other. | Count with ICU4J `BreakIterator.getCharacterInstance()` (test-only dependency). While implementing, run the counter over the whole glossary and fix or mark anything over budget (3.3 §8.6). |
| N6 | **Label test would fail the glossary's own About row.** Flutter allows `प्रयुक्ता कृत्रिमबुद्धिः` (24) as an About row label, but a `label_` prefix makes it "short". | CI would fail on a documented, allowed case. | About row labels use the `about_detail_` prefix, which is outside the short list (3.3 §8.6). |
| N7 | **About content in JSON skips every language check.** `app_config.json` locale maps and `help_<lang>.md` are not `strings.xml`, so lint, the parity test and the Hindi gate never see them. | Rules 4, 5, 6 have a hole. | The parity test also checks that every locale map in `app_config.json` has non-empty `en`, `ml`, `sa`, and that every `help_en.*` has `ml`/`sa` twins. The Hindi gate also scans `sa` JSON values and `*_sa.*` assets (3.3 §8.7). |
| N8 | **Multi-module apps.** Gates pointed at `app/src/main/res/values-sa/strings.xml` only. Feature modules have their own `res/`. | Strings in feature modules would escape all checks. | All gates and tests scan `**/src/main/res/values*/strings.xml` in every module (3.3, 3.4). |
| N9 | **Nav glossary is inconsistent.** Back, Next and Previous are nouns but are usually buttons, and Close is already an imperative in the same table. Rev 2 fixed only Exit. | Same R7 problem, half fixed. | Navigation terms that act as buttons (Back, Next, Previous, Close, Exit, More) get two forms: a label/title form and a button form. Button forms follow §8.5.1 (3.3 §8.5). |
| N10 | **No review of the Sanskrit/Malayalam wording itself.** The plan changes glossary words, but no fluent reader checks them. | Rule 4 asks for *proper* Sanskrit; an unchecked edit could make it worse. | Every new or changed ML/SA term is listed in the change log under "needs native-reader review". §8.5 adds a rule that glossary changes need that review before they are used in an app (3.3 §8.5, 5). |
| N11 | **"Every text" had no stated limits.** Permission dialogs, share sheet, Play in-app review, Google sign-in, system notification chrome and the launcher label use the system language. No app can change that. | Without a list, reviewers would chase impossible fixes, or the rule would be quietly ignored. | §8.7 lists these system-owned surfaces as accepted exceptions. Everything the app draws is covered (3.3 §8.7). |
| N12 | **Channel re-registration timing was vague.** On API 33+ the language can change from system settings while the app is not in front. | R4 fix would miss that path. | Register channels with localized names in `Application.onCreate` and again in `onConfigurationChanged` when the locale changes (3.3 §8.7). |
| N13 | **`generateLocaleConfig` output was not checked.** Which languages end up in the generated file depends on the AGP version and the resource sources. | The system "App languages" screen might list wrong languages. | Release checklist step: confirm the generated locale config lists exactly `en`, `ml`, `sa`. If not, write `res/xml/locales_config.xml` by hand and turn generation off (3.3 §8.1, 3.5). |
| N14 | **Lint could be silenced.** `tools:ignore="MissingTranslation"` or a lint baseline would hide missing strings. | Rule 5 bypass. | Forbidden. CI grep fails on `ignore="MissingTranslation"`, and baselines must not contain translation issues (3.3 §8.1). |
| N15 | **`%1$s` must match in all three files.** A translator who drops the heart placeholder would crash the badge or hide the heart. | Rule 1 bug. | Lint `StringFormatMatches` / `StringFormatCount` as errors. The badge test checks that the heart is rendered in all three locales (3.1). |
| N16 | **Play gaps.** The Advertising ID declaration was missing (libraries such as Firebase add `AD_ID`). The screenshot rule was copied wrong (the rule is 320–3840 px per side, with the long side at most 2× the short side). 16 KB only applies when the app or a dependency ships native `.so` files. | Rejected upload or wasted work. | Fixed in 3.5. |
| N17 | **Template gaps.** `architecture.md` has nowhere to record the language setup, digit decision, min/target SDK policy or `app_name` translation choice, which the rules now ask for. The existing `cd_` string prefix in §7.3 conflicts with the new prefixes. | Rules point to a field that does not exist; mixed naming. | Add a "Localization" block to `architecture.md` §16; change the §7.3 example to `tooltip_` (section 2, 3.4). |
| N18 | **Existing broken reference.** `guideline.md` line 61 says "see §1.5", but there is no §1.5. | Wrong link, found while checking section numbers. | Point it to the `ConfigService` rules in §1.1 (section 2). |
| N19 | **Weak points in the platform "saves the choice" claim.** Below API 33 AppCompat reads the saved choice from disk on the main thread, which StrictMode reports. On API 33+ the platform stores it. Neither needs an app preference. | Someone might add a second DataStore key that disagrees with the platform. | §8.4: never store the language in app preferences as well. The StrictMode disk read at startup is expected (3.3 §8.4). |

## 1. What the issue is

The Kotlin guidelines make `strings.xml` mandatory, but they still talk about a "single-language
app" and leave out eight rules the owner wants enforced in every app:

1. **About badge.** Every About screen ends with the same "Made with ❤️ from India" line.
2. **Three languages.** Every app ships English, Malayalam and Sanskrit. The default is the system
   language. The user can change it inside the app.
3. **Play Store readiness.** Every app is always ready to publish on Google Play.
4. **Sanskrit must be Sanskrit,** not Hindi written in Devanagari.
5. **Every feature in all three languages.**
6. **Every text the app draws follows the language choice.**
7. **Short labels** in all three languages; only descriptive text may be long.
8. **Tooltips** on every icon-only button, with localized text.

## 2. Files to be changed

| File | Change |
|---|---|
| `guideline.md` | §1.1 localized JSON schema + `LocalizedText` + `AppConfig`, fix the "§1.5" reference (N18); §1.2 Pattern B rule; §1.3 localized row labels; new §1.4 badge; §3 layout (`values-ml/`, `values-sa/`, `res/resources.properties`) and rules; §4 checklist |
| `kotlin_project_engineering_standard.md` | §7.3 tooltip MUST and `tooltip_` example (N17) + new §7.7; §8 rewritten (3.3); §17.4 bundled-font licensing; §18 tests; §19 CI gates; §22 AI rules (remove the "single-language app" line); §23.1 Definition of Done |
| `kotlin_build_configuration_guide.md` | New "Languages" section: locale filter, `generateLocaleConfig` + check, `resources.properties`, language split off, AppCompat + theme parent, lint translation checks as errors; target SDK row points to the Play gate |
| `release_process.md` | New §9A Play gate; §8 "Localization" and "Google Play Store Readiness" checklist blocks; §9 steps; §1 note that build-time items apply to every app |
| `architecture.md` | §16 gains a "Localization" block: languages, locale filter, formatting locales and digits, `app_name` translation choice, font strategy; §19 notes the Play target-SDK policy (N17) |
| `CLAUDE_MD_GUIDELINE.md` | Section list item 11 and table row renamed to "Localization"; template block rewritten; self-check boxes |
| `AGENTS_MD_GUIDELINE.md` | Same edits, word-for-word aligned |
| `docs/kotlin_project_engineering_standard_README.md` | Plain-English items; remove "even single-language apps" |
| `docs/release_process_README.md` | Plain-English item for the Play gate |
| `docs/architecture_README.md` | `guideline.md` row mentions the badge and three languages |
| `README.md`, `GUIDELINES_MANIFEST.md`, `CLAUDE.md`, `AGENTS.md` | `guideline.md` description line updated |

## 3. The plan for the fix

### 3.1 About badge (rule 1) — `guideline.md` new §1.4

- Last element of the About screen, centered, ≥ 24 dp above it, `navigationBarsPadding()` below.
- Words: `MaterialTheme.colorScheme.onSurfaceVariant`, `bodySmall`.
- **Heart (N1):** an inline `Icons.Filled.Favorite` icon, 1 em square, tinted `Color(0xFFE53935)`,
  added with `appendInlineContent` + `InlineTextContent`. It does not depend on any emoji font.
- Strings use a `%1$s` marker where the heart goes. Fixed wording, identical to Flutter:

  | File | `about_made_with_love` | `about_made_with_love_a11y` |
  |---|---|---|
  | `values/strings.xml` | `Made with %1$s from India` | `Made with love from India` |
  | `values-ml/strings.xml` | `സ്നേഹത്തോടെ %1$s ഇന്ത്യയിൽ നിന്ന്` | `സ്നേഹത്തോടെ ഇന്ത്യയിൽ നിന്ന്` |
  | `values-sa/strings.xml` | `सस्नेहं निर्मितम् %1$s भारततः` | `सस्नेहं निर्मितम् भारततः` |

- Reference `ui/components/MadeWithLove.kt`: formats with a sentinel, splits, inserts the inline
  icon, and uses `Modifier.clearAndSetSemantics { contentDescription = a11y }`. Not clickable, not
  configurable, never removed or reworded per app.
- **Checks (N15):** lint `StringFormatMatches` / `StringFormatCount` as errors; a UI test renders the
  badge in `en`, `ml`, `sa` and confirms the inline heart is present.

### 3.2 Localized About content (rules 1, 6)

- **Pattern A:** `appName`, `description` and each `details` value may be a plain string (same in
  every language) or a `{"en": …, "ml": …, "sa": …}` map. New `LocalizedText` with
  `resolve(languageCode)` falling back to `en`. `fromJson` accepts both shapes.
- `details` keys become stable ids (`author`, `email`, `license`, `aiUsed`, `ideUsed`). Row labels
  come from `strings.xml` as `about_detail_<id>` (N6), with the raw key as fallback. Wording comes
  from the glossary's About table.
- **Pattern B:** `BuildConfig` holds only language-independent values. Labels and prose come from
  `strings.xml`.
- The screen reads the language from `LocalConfiguration.current.locales[0]`.

### 3.3 Three languages + in-app switcher (rules 2, 4, 5, 6, 7) — standard §8 rewritten

**§8 intro.** Every app ships `en`, `ml`, `sa`. There is no single-language app.

**§8.1 Minimum setup (all MUST):**

- `res/values/strings.xml` (English, default), `res/values-ml/strings.xml`,
  `res/values-sa/strings.xml` — in every module that has user-visible text (N8).
- `res/resources.properties` with `unqualifiedResLocale=en`; `androidResources { generateLocaleConfig = true }`.
  The generated file MUST list exactly `en`, `ml`, `sa`. If it doesn't, write
  `res/xml/locales_config.xml` by hand and turn generation off (N13).
- Locale filter to `en`, `ml`, `sa` (`androidResources.localeFilters` on AGP versions that have it,
  otherwise `defaultConfig.resourceConfigurations`) (R2).
- `bundle { language { enableSplit = false } }` (R1).
- `androidx.appcompat` 1.6+; `MainActivity` extends `AppCompatActivity`; the XML theme parent is
  `Theme.AppCompat.DayNight.NoActionBar` or a Material Components descendant (N2).
- Manifest: `androidx.appcompat.app.AppLocalesMetadataHolderService` (`enabled=false`,
  `exported=false`, meta-data `autoStoreLocales=true`) — needed because the baseline `minSdk` is 24.
- Lint `MissingTranslation`, `ExtraTranslation`, `StringFormatMatches`, `StringFormatCount` are
  errors. `tools:ignore` for them and lint-baseline entries for them are forbidden, and CI checks
  this (N14).

**§8.2 String externalization:** current rules stay. All three files are required, with real
translations. `translatable="false"` only for text that must look the same in every language:
endonyms, brand names (including `app_name` if the app records that choice in `architecture.md`),
symbols. Translator comments go in `values/strings.xml` only.

**§8.3 The three languages:**

- Table of locale, script and role. English is the fallback; libraries show English under `sa`,
  never Hindi.
- **Formatting locales (N3):** fixed mapping in `formattingLocale(appLocale)`:
  `en` → English (keeping the system region when the system language is English), `ml` → `ml-IN`,
  `sa` → `Locale.ENGLISH`. Western digits in all three. Build locales with `Locale.forLanguageTag`.
  Never `hi`.
- **Material 3 pickers (N4):** `DatePicker`, `DateRangePicker` and `TimePicker` MUST be checked
  under `sa`. If month names or digits are wrong, format them with the helper's locale (reference
  snippet), or pass the locale where the Material 3 version supports it.
- **Plurals:** every `<plurals>` has an `other` form that reads correctly for any number (Android
  may have no plural rules for `sa`).
- **Fonts:** check every screen in `ml` and `sa` on a clean device. Bundle Noto Sans Malayalam /
  Noto Sans Devanagari in `res/font/` if glyphs are missing (§17.4 licences).
- **TalkBack:** endonyms in another script SHOULD carry a `LocaleList` span.
- **Text input:** fields that expect Malayalam or Sanskrit text SHOULD set
  `KeyboardOptions(hintLocales = …)`.

**§8.4 In-app language selection:**

- Resolution: saved choice → first system language that is `en`/`ml`/`sa` → English.
- Set with `AppCompatDelegate.setApplicationLocales(LocaleListCompat.forLanguageTags("ml"))` on the
  main thread. An empty list means "System default". Read with `getApplicationLocales()`.
- The platform (API 33+) or AppCompat (below 33) saves the choice. **Never keep a second copy** in
  app preferences. The AppCompat disk read at startup is expected in StrictMode (N19).
- Applies at once, app-wide. The Activity is recreated, so screen state lives in a ViewModel or
  `rememberSaveable`, and the user stays on the same screen (R15).
- The picker is in Settings: "System default" first, then English / മലയാളം / संस्कृतम्, with the
  current choice marked and read by TalkBack.
- A `LanguageRepository` wraps the AppCompat calls (reference code + Settings Composable).

**§8.5 Sanskrit & Malayalam quality:** ported from the final Flutter §8.5, with these Kotlin changes:

- The CI Hindi-marker gate scans every module's `values-sa/strings.xml`, plus `sa` JSON values and
  `*_sa.*` assets (N7, N8). `>` and `<` are added as word edges (R5). The doc includes a test string
  showing the gate fails on `<string name="x">है</string>`.
- One term per meaning (R7): About = `विषयपरिचयः` / `ആപ്പിനെക്കുറിച്ച്` everywhere, Profile =
  `परिचयः`, Description = `വിവരണം`.
- Navigation terms used as buttons (Back, Next, Previous, Close, Exit, More) get a label form and a
  button form, following §8.5.1 (N9).
- **New rule (N10):** any new or changed glossary term in Malayalam or Sanskrit needs review by a
  fluent reader before apps use it. The change log for this work lists every such term under
  "needs native-reader review".

**§8.6 Label conciseness:**

- Short text: English 1–2 words and ≤ 20; Malayalam 1–2 words and ≤ 22; Sanskrit 1 word and ≤ 22.
- Count with ICU4J `BreakIterator.getCharacterInstance()` (test-only dependency) (N5). When writing
  the doc, run that count over the glossary and fix or mark anything over budget.
- No shrinking fonts, truncating or "…" to make text fit. Sentence case in English.
- Prefixes: `action_`, `label_`, `title_`, `tab_`, `nav_`, `tooltip_` are short;
  `desc_`, `help_`, `empty_`, `error_`, `body_`, `about_detail_` are exempt (N6). A JVM test
  enforces the budget.

**§8.7 Per-feature completeness:**

- A feature is not done until it works in all three languages.
- JVM parity test over every module: `<string>`, `<plurals>`, `<string-array>` names match across
  the three files; every locale map in `app_config.json` has non-empty `en`/`ml`/`sa`; every
  `help_en.*` asset has `ml` and `sa` twins (N7, N8).
- A test fails when an `ml`/`sa` value is an exact copy of English (allow-list for brand names,
  symbols, and strings that are only placeholders).
- **Context rule (R3):** ViewModels and repositories return `@StringRes` / `UiText`, never resolved
  strings. Code without an Activity (notifications, WorkManager, widgets, services) uses
  `createConfigurationContext(...)` built from `AppCompatDelegate.getApplicationLocales()`.
- **Notification channels (N12):** registered with localized names in `Application.onCreate` and
  again in `onConfigurationChanged` when the locale changes.
- **Accepted exceptions (N11):** system-owned UI (permission dialogs, share sheet, system
  notification chrome, launcher label, Play in-app review/update dialogs, Google sign-in) follows the
  system language. Everything the app draws is covered.
- UI and screenshot tests run in `en`, `ml`, `sa` (Robolectric `@Config(qualifiers = …)`,
  Roborazzi per locale).

**§8.8 RTL** and **§8.9 Locale-sensitive formatting:** current text kept and renumbered. §8.8 notes
that none of the three languages is RTL. §8.9 uses `formattingLocale`, and sorts user-visible text
with `java.text.Collator` for that locale.

### 3.4 Tooltips (rule 8) — standard §7.3 + new §7.7

- Covers `IconButton` and its filled/tonal/outlined variants, `IconToggleButton`, all FAB sizes
  without text, `TopAppBar` actions, the overflow button, `NavigationBarItem` / `NavigationRailItem`
  with `alwaysShowLabel = false`, search-bar icons, and custom icon-only `clickable` elements.
- One shared `ui/components/TooltipIconButton.kt` (and a FAB twin): Material 3 `TooltipBox` +
  `PlainTooltip`. One text parameter feeds both the tooltip and the `contentDescription`. The doc
  notes the `ExperimentalMaterial3Api` opt-in and that the position-provider function name depends
  on the Material 3 version.
- Text from `strings.xml` with `tooltip_` prefix (the §7.3 `cd_` example is changed, N17). It names
  the action and stays within §8.6.
- A control with visible text next to its icon needs no tooltip. A tooltip is never the only source
  of needed information and never replaces a confirmation.
- **Checks:** (1) CI grep over all modules fails when a raw icon-button or icon-FAB call appears
  outside the wrapper files; (2) a Compose UI test asserts every clickable node without text has a
  non-empty `contentDescription`, in all three locales.

### 3.5 Play Store readiness (rule 3) — `release_process.md` new §9A

- **Build-time — MUST from day one** (9A.1–9A.4). **Console-time — MUST before the first upload**
  (9A.5–9A.8).
- **9A.1 Identity:** permanent `applicationId`, strictly increasing `versionCode`, `versionName` in
  sync with About, `app_name` ≤ 30 characters, `<queries>` when the app looks up other apps.
- **9A.2 API level:** `targetSdk` meets Play's current rule (at the time of writing, API 36 for new
  apps and updates from 31 August 2026 — re-check every release); `compileSdk` ≥ `targetSdk`;
  `minSdk` recorded in `architecture.md`; 64-bit; 16 KB page size **when the app or any dependency
  ships native `.so` files** (N16); edge-to-edge.
- **9A.3 Build and signing:** `.aab` from `./gradlew bundleRelease`; R8 on; language split off and
  the generated locale config checked (R1, N13); Play App Signing; upload key backed up offline;
  `keystore.properties` never committed; `mapping.txt` uploaded; native debug symbols only when
  there is native code.
- **9A.4 Manifest and policy:** permissions justified; sensitive ones avoided or declared (all-files
  access, exact alarms, accessibility service, SMS/call log, background location,
  `QUERY_ALL_PACKAGES`, broad photo/video access → photo picker); `AD_ID` removed with
  `tools:node="remove"` when the app has no ads or analytics that need it (N16);
  `foregroundServiceType`; `debuggable=false`; `allowBackup` and `dataExtractionRules` chosen on
  purpose; no cleartext; no accidental `exported="true"`; prominent in-app disclosure before
  collecting sensitive data.
- **9A.5 Console declarations:** privacy policy URL; Data safety form matching real behaviour
  (including SDKs); Advertising ID declaration (N16); content rating; target audience; ads,
  government, financial and health declarations where they apply; account deletion path if accounts
  exist; developer contact; closed test for new personal accounts (at the time of writing, 12 testers
  for 14 days).
- **9A.6 Listing assets:** icon 512×512 PNG; feature graphic 1024×500 (JPEG or 24-bit PNG, no
  alpha); 2–8 phone screenshots, each side 320–3840 px and the long side at most 2× the short side
  (N16); tablet sets if offered on tablets; title ≤ 30, short description ≤ 80, full description
  ≤ 4000.
- **9A.7 Listing languages:** English and Malayalam (`ml-IN`) listings with screenshots in that
  language. Sanskrit is not a Play listing language (at the time of writing); it ships in-app only.
- **9A.8 Pre-launch:** internal testing track first; clean pre-launch report; a Play-served build
  checked in all three languages, including a language different from the phone's (catches R1);
  staged rollout with Android vitals checks.
- Matching §8 checklist blocks, a §9 step, and a §1 note.

### 3.6 Consistency pass

- Mirror the MUST rules into `CLAUDE_MD_GUIDELINE.md` / `AGENTS_MD_GUIDELINE.md` (section list,
  table row, template block, self-check), word-for-word aligned.
- Update the explainers, `README.md`, `GUIDELINES_MANIFEST.md`, `CLAUDE.md`, `AGENTS.md`,
  `docs/architecture_README.md`.
- Grep for "single-language", "one language", "ships only", "String resources", "cd_" and fix every
  match.
- Check that every section number referenced across documents resolves, and that every new link is
  relative.
- Check every Kotlin/Gradle snippet for API names against the toolchain baseline (Kotlin 2.1, AGP
  8.x, Compose BOM 2025.x), and mark version-dependent names as such.

## 4. Out of scope

- No app code is created or changed; documentation only.
- Translating any real app's string files.
- Other stores.
- **Follow-up for the Flutter repository (separate plan):** glossary conflicts (R7, N9), Play
  language split (R1), heart rendering (N1), grapheme counting (N5), JSON/asset language checks (N7),
  and the About-row label exemption (N6).

## 5. After implementation

Write the change log to `change_log/20260913_hhMMss_trilingual-about-badge-playstore-tooltips.md`:
reference this plan with a relative path, list every corrected or added glossary term under
"needs native-reader review", and set this plan's `Status:` to `completed`.
