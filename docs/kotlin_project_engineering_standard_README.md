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
   - Mandatory string externalization into `res/values/strings.xml` (even single-language apps)
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
| Strings / Localization | `l10n.yaml` + ARB files | `res/values/strings.xml` |
| Testing | `flutter test` | JUnit + Robolectric + Compose UI test rules |
| Code Shrinking | `--obfuscate` | R8 / ProGuard rules (`proguard-rules.pro`) |

---

## How to Use This in a Project

1. Copy `GUIDELINES_MANIFEST.md` into your project's `docs/` directory.
2. Link the shared `Kotlin_Guidelines` as a Git submodule under `docs/guidelines/`.
3. In your project's `CLAUDE.md` and `AGENTS.md`, declare the profile in force (`Core Baseline`, `Production App Extension`, etc.).
4. When writing code, tests, or migrations, adhere strictly to the corresponding sections of the standard.
