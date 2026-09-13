# Architecture

Use this document to describe the current system design of the Kotlin Android app.

> **How to fill this in.** This is a template. Fill in only the sections that apply to your app
> and mark the rest `N/A` (with a one-line reason) rather than inventing content. A small
> single-screen tool will legitimately mark many sections `N/A`; a shipped sensitive-data app
> will fill in most of them. The goal is an accurate picture, not a fully-populated form.

## 1. Scope

- Product: `<app name>`
- Repository type: `application` or `library`
- Engineering standard profiles in force:
  - `Core Baseline`
  - `Production App Extension` if applicable
  - `Sensitive Data Extension` if applicable
- Platform: `Android` (minSdk `<NN>`, targetSdk `<NN>`, compileSdk `<NN>`)

---

## 2. Goals And Non-Goals

### Goals

- `<goal 1>`
- `<goal 2>`
- `<goal 3>`

### Non-Goals

- `<non-goal 1>`
- `<non-goal 2>`

---

## 3. Architecture Summary

Describe the current architecture in one short paragraph.

Example:

> The app uses a single-module MVVM architecture with Jetpack Compose for the UI layer.
> A single ViewModel exposes StateFlows to the Composable screens. Data persistence is
> isolated behind a Repository abstraction over a Room database. App-wide configuration
> is loaded from a JSON asset at startup. The app is fully offline; no network access is
> used or permitted.

---

## 4. Repository Structure

### Current Structure Tier

- `Tier 1` or `Tier 2`
- Why this tier is appropriate now:
  - `<reason 1>`
  - `<reason 2>`

### Top-Level Source Layout

```text
app/src/main/java/<package>/
|-- <package>
|-- <package>
`-- MainActivity.kt
```

### Ownership Rules

| Path | Responsibility |
|------|----------------|
| `<package>/...` | `<responsibility>` |
| `<package>/...` | `<responsibility>` |

---

## 5. App Initialization Sequence

Document the exact order of initialization steps that run in `Application.onCreate()` and
`MainActivity.onCreate()`. Getting this order wrong causes crashes that may only appear in
release builds.

| Step | Code / Call | Notes |
|------|-------------|-------|
| 1 | `Application.onCreate()` | Application-level init |
| 2 | DI / service locator init | e.g. Hilt, or manual singleton setup |
| 3 | Database open + migrate | Room schema version N applied |
| 4 | Config load | `ConfigService.load(context)` |
| 5 | Logging init | Logger configured before any log calls |
| 6 | `setContent { }` in Activity | Compose UI entry point |

Document any initialization that is deferred (lazy) and explain why.

---

## 6. App Lifecycle Behavior

Document what the app does in response to each lifecycle state change.

| Lifecycle Event | App Behavior |
|----------------|--------------|
| `ON_RESUME` | `<e.g. resume timers, re-validate lock state>` |
| `ON_PAUSE` | `<e.g. flush pending DB writes, trigger app lock>` |
| `ON_STOP` | `<e.g. cancel background work>` |
| `ON_DESTROY` | `<e.g. close file handles>` |
| Low memory (`onTrimMemory`) | `<e.g. clear image cache>` |

---

## 7. Offline Behavior

State whether the app is online, offline-first, or cache-assisted, and document the implications.

- **Connectivity requirement**: `fully offline` / `offline-first` / `online required`
- **Network permission**: `INTERNET permission absent` / `present but optional`
- **Offline data source**: `<Room / files / SharedPreferences>`

If the app is **fully offline**:
- The AndroidManifest MUST NOT contain `<uses-permission android:name="android.permission.INTERNET" />`.
  Verify this by inspecting the merged manifest:
  - `aapt2 dump permissions app/build/outputs/apk/release/<apk>` — lists every permission baked into the APK.
  - Or open the merged manifest in Android Studio: Build → Analyze APK → select the APK → open `AndroidManifest.xml`.
- All dependencies have been audited for transitive network activity.

---

## 8. State Management

- Primary pattern: `ViewModel + StateFlow` / `ViewModel + LiveData` / `<other>`
- Why this pattern was chosen:
  - `<reason 1>`
  - `<reason 2>`
- State boundaries:
  - Composables own: `<ui-only concerns>`
  - ViewModel owns: `<screen/app state concerns>`
  - Repository owns: `<data access concerns>`

---

## 9. Data Flow

Describe the expected request and update path.

```text
Composable -> ViewModel -> Repository -> Database (Room) / Remote (Retrofit)
```

If the app intentionally omits a layer, document that here.

### Rules

- Composables must not know: `<SQL/HTTP/crypto/etc.>`
- ViewModels must not know: `<Context for UI operations/navigation/etc.>`
- Repositories abstract: `<api/db/cache/etc.>`

---

## 10. Error Handling Architecture

Document how errors are classified and propagated from the data layer to the UI.

- **Global error handler**: `Thread.setDefaultUncaughtExceptionHandler` configured in
  `Application.onCreate()`. See `config/` or `utils/` for implementation.
- **Domain exception hierarchy**: sealed class/interface in the data or domain layer.

| Exception Class | Thrown By | Meaning |
|----------------|-----------|---------|
| `StorageException` | Repository | DB read/write failure |
| `ValidationException` | ViewModel | Input failed business rules |
| `<add more>` | `<layer>` | `<meaning>` |

- **Error escalation policy**: `<document when a recoverable error becomes a session error, etc.>`

---

## 11. Domain Model

### Current Schema Version

Room database version: `<N>` (increment this whenever a migration is added)

Migration history:

| Version | Change Summary |
|---------|---------------|
| 1 | Initial schema: `<table names>` |
| 2 | `<change>` |

### Core Models Or Entities

| Type | Purpose | Mutable? | Notes |
|------|---------|----------|-------|
| `<ModelName>` | `<purpose>` | `No` | `<notes>` |
| `<ModelName>` | `<purpose>` | `No` | `<notes>` |

### Serialization Strategy

- JSON models (Moshi/Gson/kotlinx.serialization): `<yes/no>`
- Room entities: `<yes/no>`
- Separate domain entities from transport models: `<yes/no and why>`

### Database Indexes

| Table | Indexed Columns | Reason |
|-------|----------------|--------|
| `<table>` | `<column>` | `<query that needs it>` |

---

## 12. Dependency Management And Injection

- DI approach: `<manual wiring / Hilt / Koin / manual singletons>`
- App-root dependencies:
  - `<dependency>`
  - `<dependency>`
- Test replacement strategy:
  - `<mock/fake/override approach>`

---

## 13. Navigation

- Navigation approach: `<Compose Navigation / single-Activity tab switching / custom>`
- Route definition location: `<path>`
- Protected-route strategy: `<auth/app-lock gating pattern>`
- Deep-link support: `<yes/no>`

---

## 14. Persistence And External Systems

### Local Storage

- Database: `<Room>`
- Key-value storage: `<SharedPreferences / DataStore>`
- Secure storage: `<EncryptedSharedPreferences / AndroidKeystore>`

### Network

- Network client: `<Retrofit+OkHttp / Ktor / none>`
- Offline behavior: `<online-only/offline-first/cache-assisted/fully offline>`

### Platform Integrations

- `<integration>`: `<purpose>`
- `<integration>`: `<purpose>`

---

## 15. Environment And Build Model

- Build types: `<debug/release>`
- Product flavors: `<dev/prod/staging/none>`
- Runtime config mechanism: `<BuildConfig fields / assets / none>`
- Build outputs supported:
  - `<debug APK>`
  - `<release APK split-per-abi>`
  - `<app bundle>`
- Code shrinking: `<R8 enabled for release builds / disabled>`

---

## 16. UI System

- Theme source of truth: `<ui/theme/Theme.kt>`
- Design tokens location: `<ui/theme/Color.kt, Type.kt, Shape.kt>`
- Shared Composable strategy: `<where shared UI lives>`
- Accessibility expectations:
  - Minimum touch target: 48 × 48 dp on mobile
  - Color contrast: WCAG AA minimum (4.5:1 normal text, 3:1 large text)
  - Screen reader: TalkBack tested before each release
  - Text scale: layouts verified at 1.0×, 1.5×, 2.0× text scale
  - Tooltips: every icon-only control uses `TooltipIconButton` / `TooltipFab` (engineering standard §7.7)

### Localization

Every app ships English, Malayalam and Sanskrit (engineering standard section 8). Record this
app's choices:

| Item | This app |
|---|---|
| Languages | `en` (default `values/`), `ml` (`values-ml/`), `sa` (`values-sa/`) — fixed |
| Modules with UI strings | `<app, feature modules…>` |
| Locale filter | `<resourceConfigurations / androidResources.localeFilters>` = `en`, `ml`, `sa` |
| Locale config | `<generated (verified on <date>) / hand-written res/xml/locales_config.xml>` |
| Language picker location | `<Settings screen route>` |
| `app_name` | `<translated in all three / translatable="false" brand name>` |
| Formatting locales | `en` → English, `ml` → `ml-IN`, `sa` → English; Western digits `<or recorded exception>` |
| Fonts for Malayalam / Devanagari | `<system fonts verified on <device list> / bundled Noto fonts in res/font/>` |
| Translated asset content | `<none / assets/content/help_<lang>.md …>` |
| Native-reader reviewer | `<role, not a personal name or email>` |

---

## 17. Logging

- Logger implementation: `<Logcat with tags / Timber / custom>`
- Log file location (if applicable): `<path e.g. app cache dir>`
- Verbose logging gate: `<BuildConfig.DEBUG flag>`
- Sensitive data policy: `<never logged / explicitly listed exceptions>`

---

## 18. Testing Strategy

| Test Type | Scope | Notes |
|-----------|-------|-------|
| Unit (JUnit + Robolectric) | `<scope>` | `<notes>` |
| Compose UI | `<scope>` | `<notes>` |
| Screenshot (Roborazzi) | `<scope>` | `<notes>` |
| Instrumented | `<scope>` | `<notes>` |

### Test Layout

```text
app/src/test/java/<package>/
|-- <mirrored packages>
|-- helpers/
`-- fixtures/

app/src/androidTest/java/<package>/
|-- <instrumented tests>
```

### Critical Test Areas

- `<critical logic area>`
- `<critical flow>`
- `<migration or parsing path>`
- Database upgrade path from version 1 to current
- App lock trigger and re-authentication flow (if applicable)

---

## 19. Operational Constraints

Document constraints that shape implementation choices.

- Minimum supported Android version: `minSdk <NN>` (Android `<version>`)
- Target Android version: `targetSdk <NN>` (Android `<version>`)
- Performance constraints:
  - Cold startup target: under 2 seconds to first meaningful frame (release build)
  - APK size budget: `<see engineering standard section 10.7>`
- Store constraints: Google Play readiness gate (`release_process.md` §9A); `targetSdk` re-checked
  against Play's current target API level policy before every release
- Regulatory constraints: `<if any>`
- Team constraints: `<single developer / multi-developer / release cadence>`
- Offline constraints: `<no INTERNET permission / no network dependencies>`

---

## 20. Decisions And Tradeoffs

Record the decisions that are likely to be questioned later.

| Decision | Chosen Option | Why | Tradeoff |
|----------|---------------|-----|----------|
| `<topic>` | `<choice>` | `<reason>` | `<tradeoff>` |
| `<topic>` | `<choice>` | `<reason>` | `<tradeoff>` |

---

## 21. Known Risks And Follow-Ups

- Risk: `<risk>`
  Mitigation: `<mitigation>`
- Risk: `<risk>`
  Mitigation: `<mitigation>`

---

## 22. Related Documents

- `../README.md`
- `../CLAUDE.md`
- `../AGENTS.md`
- `guidelines/guideline.md` (submodule)
- `guidelines/kotlin_project_engineering_standard.md` (submodule)
- `guidelines/kotlin_build_configuration_guide.md` (submodule)
- `guidelines/DOCS_FOLDER_GUIDELINE.md` (submodule)
- `release_process.md` (required for shipped apps)
- `security.md` (required for sensitive-data apps)
- `GUIDELINES_MANIFEST.md`
