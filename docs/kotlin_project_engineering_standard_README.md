# Explainer: Kotlin Project Engineering Standard

## What is this document?

`kotlin_project_engineering_standard.md` is the **master rulebook** for building native Android apps with Kotlin and Jetpack Compose. It contains the universal engineering standards that apply across all projects — regardless of app size, domain, or team.

It is organized into 24 numbered sections, from project structure and MVVM architecture to accessibility, performance, database migrations, security, testing, CI, git hygiene, and AI assistant guidelines.

---

## The Three Profiles System

Not every rule in the 24 sections applies to every app. To keep small tools lean without compromising larger projects, the standard uses **3 applicability profiles**:

1. **`Core Baseline`** (Applies to **Every** app):
   - Mandatory root `CLAUDE.md` and `AGENTS.md`
   - Plan-before-changing and log-after-changing workflow rules
   - Privacy rule for `plans/` and `change_log/` (relative paths only, no local system details, no secrets)
   - Three mandatory app languages — English, Malayalam and Sanskrit — with an in-app language picker (section 8)
   - Proper Sanskrit, never Hindi, checked by a script and a shared word list (section 8.5)
   - Short menu, button, label and tooltip text in all three languages (section 8.6)
   - A localized tooltip on every icon-only button (section 7.7)
   - MVVM architecture with Compose and ViewModel
   - Accessibility baseline (48dp touch targets, WCAG AA contrast, TalkBack labels)
   - Room migration and database integrity rules
   - Clean lint and test execution

2. **`Production App Extension`** (Applies to apps shipped to users / stores):
   - R8/ProGuard code shrinking (`isMinifyEnabled = true`)
   - Mapping file archiving (`mapping.txt`)
   - Performance frame budget (16ms / 60Hz) and cold startup targets (< 2s)
   - APK / AAB size budgets
   - CI automated verification pipelines

3. **`Sensitive Data Extension`** (Applies to apps handling secrets, finance, health, private files):
   - `EncryptedSharedPreferences` / AndroidKeystore for secret material
   - Authenticated cryptography (AES-GCM)
   - App lock and background lock on `onPause`
   - `FLAG_SECURE` window screenshot protection
   - Full OWASP Mobile Top 10 compliance checklist

---

## Key Differences from Flutter Standards

| Concern | Flutter Equivalent | Kotlin Android Equivalent |
|---|---|---|
| Language & Framework | Dart / Flutter widgets | Kotlin / Jetpack Compose |
| Build System | `pubspec.yaml` | Gradle Kotlin DSL (`build.gradle.kts`) + `libs.versions.toml` |
| State Management | Riverpod / Provider / Bloc | ViewModel + `StateFlow` / `SharedFlow` |
| Local Storage | `sqflite` | Room database (`@Database`, `@Entity`, `@Dao`) |
| Code Generation | `build_runner` | KSP (Kotlin Symbol Processing) |
| Strings / Localization | `l10n.yaml` + `app_en.arb`, `app_ml.arb`, `app_sa.arb` | `values/`, `values-ml/`, `values-sa/` `strings.xml` |
| In-app language switch | `LocaleController` + `MaterialApp.locale` | `AppCompatDelegate.setApplicationLocales` (needs `AppCompatActivity` + AppCompat theme) |
| Icon-button tooltips | `tooltip:` parameter | `TooltipIconButton` wrapper around Material 3 `TooltipBox` |
| Testing | `flutter test` | JUnit + Robolectric + Compose UI test rules |
| Code Shrinking | `--obfuscate` | R8 / ProGuard rules (`proguard-rules.pro`) |

---

## The Three Languages In Plain English

Every app works in **English, Malayalam and Sanskrit**. Here is what that means in practice:

- **Three string files.** Every piece of text lives in `values/strings.xml` (English),
  `values-ml/strings.xml` (Malayalam) and `values-sa/strings.xml` (Sanskrit). A new feature is not
  finished until its text is in all three.
- **The user chooses.** The app starts in the phone's language (or English if the phone uses a
  different language). A picker in Settings lets the user change it, and the change applies at once.
- **Some build settings are required.** Without them the picker silently fails, for example after
  a Play Store install. The build configuration guide lists each setting and what it prevents.
- **Sanskrit must be real Sanskrit.** Hindi uses the same letters, so it can look like Sanskrit.
  A script fails the build when common Hindi words appear, and a shared word list keeps the same
  terms in every app. A fluent reader still checks new wording.
- **Short labels.** Buttons, menus, tabs and tooltips stay short in every language; only
  explanations may be long.
- **Tooltips.** Every button that shows only an icon explains itself when pressed and held.

---

## How to Use This in a Project

1. Copy `GUIDELINES_MANIFEST.md` into your project's `docs/` directory.
2. Link the shared `Kotlin_Guidelines` as a Git submodule under `docs/guidelines/`.
3. In your project's `CLAUDE.md` and `AGENTS.md`, declare the profile in force (`Core Baseline`, `Production App Extension`, etc.).
4. When writing code, tests, or migrations, adhere strictly to the corresponding sections of the standard.
