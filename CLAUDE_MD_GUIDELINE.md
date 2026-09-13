# Guideline: How to Write a `CLAUDE.md` for a Kotlin Android Project

Every Kotlin Android project **MUST** have a `CLAUDE.md` file at its root, alongside a mandatory `AGENTS.md` file (see [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md) for non-Claude LLMs). This guideline defines the mandatory standards and structure for creating and maintaining `CLAUDE.md` across all Kotlin projects (both new and migrated existing apps).

`CLAUDE.md` is read by Claude Code (and AI coding assistants) automatically at the start of every session in that project. It is the first document the AI inspects, so it MUST be clear, concise where possible, and complete where required. Every Kotlin repository MUST maintain both `CLAUDE.md` and `AGENTS.md` at project root as mandatory Core Baseline requirements (**MUST**).

---

## 1. First choose a profile: Thin or Thick

Pick one of two styles before you write anything.

### Thin pointer profile
Use this when the project already has (or will have) a full `docs/` folder — `architecture.md`,
`security.md`, `release_process.md`, and so on.

- `CLAUDE.md` stays short. It gives identity, commands, and a short rule summary.
- It **points** to the `docs/` files for the detail. It does not repeat them.
- Rule: if a detail lives in a `docs/` file, do not copy it into `CLAUDE.md` — link to it.

### Thick self-contained profile
Use this when the project is small or has **no** `docs/` folder, so `CLAUDE.md` must hold
everything itself.

- `CLAUDE.md` inlines the detail: schema tables, ViewModel/repository tables, full dos-and-don'ts.

### How to choose
- Has a `docs/` set already, or the project is medium/large → **Thin**.
- No `docs/`, small project, one developer → **Thick**.
- When unsure, start **Thin** and grow. It is easier to add detail than to trim a wall of text.

---

## 2. Canonical section order

Write sections in this order. Skip the ones that do not apply (see the checklist in §3).

1. Title + read-first banner
2. Project identity / tech stack
3. Doc references (Thin profile) — the "read these before working" table
4. Package naming (backtick `in` keyword if applicable)
5. Hard / non-negotiable rules
6. Architecture rules (layers, boundaries, state, navigation, database)
7. Build & run commands
8. Build types / product flavors
9. Signing / keystore
10. Security rules
11. Localization rules — English, Malayalam, Sanskrit (mandatory for all apps)
12. Code style / naming conventions
13. Testing rules
14. Dependency constraints
15. Where things live (project tree)
16. Workflow rules (plan → approve → log) — from global rules
17. Communication rules (simple English) — from global rules
18. Dos & Don'ts ("What Claude must always / never do")

---

## 3. Section checklist (required vs optional)

| # | Section | Required? | Thin | Thick |
|---|---------|-----------|------|-------|
| 1 | Title + read-first banner | **Always** | short line | short line |
| 2 | Project identity / tech stack | **Always** | short list/table | full table |
| 3 | Doc references table | Thin only | **yes** | n/a (no docs) |
| 4 | Package naming | If applicable | inline | inline |
| 5 | Hard / non-negotiable rules | If any exist | summarize + link | inline full |
| 6 | Architecture rules | **Always** | 1 paragraph + link | inline full |
| 7 | Build & run commands | **Always** | inline | inline |
| 8 | Build types / flavors | If flavors used | inline or link | inline |
| 9 | Signing / keystore | If it ships releases | link | inline |
| 10 | Security rules | **Always** | summarize + link | inline |
| 11 | Localization rules | **Always** | inline | inline |
| 12 | Code style / naming | **Always** | short | full table |
| 13 | Testing rules | **Always** | short + link | full |
| 14 | Dependency constraints | If constrained | link | inline allow/block lists |
| 15 | Where things live (tree) | Recommended | short | full |
| 16 | Workflow rules (plan/log) | **Always** | inline (see §5) | inline (see §5) |
| 17 | Communication rules | **Always** | inline (see §5) | inline (see §5) |
| 18 | Dos & Don'ts | Recommended | optional | **yes** |

Every Kotlin app **MUST** have a root `CLAUDE.md` file. All "Always" sections in the table above must appear in every project's `CLAUDE.md`, regardless of profile.

---

## 4. Fill-in-the-blanks template

Copy this into a new project's `CLAUDE.md` and replace every `<...>` placeholder. Delete any
section that the checklist marks optional and the project does not need. Notes in `> ` blocks are
guidance — remove them from the final file.

````markdown
# CLAUDE.md — <App Name>

This file is read by Claude Code at the start of every session in this repository.
Read it before making any change. <If Thin: See the docs table below for full detail.>

---

## Project identity

| Field | Value |
|-------|-------|
| App name | <App Name> |
| Type | <one-line description of what the app does> |
| Platform | Android (minSdk <NN>, targetSdk <NN>, compileSdk <NN>) |
| Package / namespace | <in.sreerajp.app_name> |
| Kotlin | <2.1.x or higher> |
| Compose BOM | <2025.x or higher> |
| AGP | <8.x> |
| JDK | <17> |
| State management | <ViewModel + StateFlow / ViewModel + LiveData> |
| Navigation | <single-Activity tab switching / Compose Navigation / none> |
| Database | <Room / SharedPreferences / DataStore / none> |
| Orientation | <portrait only / both> |
| Connectivity | <fully offline — no INTERNET / online optional / online> |

> Keep this table honest and current. It is the fastest way for the AI to orient.

---

## Read these docs before working   <!-- Thin profile only -->

| Document | Read when |
|----------|-----------|
| docs/architecture.md | Changing structure, screens, state, services, models, repositories |
| docs/security.md | Touching permissions, logging, storage, crypto, manifest |
| docs/release_process.md | Building a release, versioning, release checklist |
| docs/kotlin_build_configuration_guide.md | Build config, signing, flavors, Gradle, ProGuard |
| docs/kotlin_project_engineering_standard.md | Any code change — layers, naming, testing |
| docs/GUIDELINES_MANIFEST.md | The shared Kotlin guidelines index |

> If a doc is copied into this project's own `docs/`, the local copy wins over the master.

---

## Package naming

One identifier everywhere: the source package, the build `namespace`, and the `applicationId`
are all `<in.sreerajp.app_name>`.

<If applicable:> Note that `in` is a Kotlin keyword, so it must be backticked in source — in
every `package` line and in every import of our own code:

```kotlin
package `in`.sreerajp.app_name.ui
import `in`.sreerajp.app_name.data.repository.AppRepository
```

---

## Hard rules (must follow — these override convenience)

1. <e.g. Open source only. No commercial/source-available SDKs. Check a package license first.>
2. <e.g. Offline-first. The app works fully offline; online parts are optional.>
3. <e.g. Never crash on bad input. Every parser has a failure path with a friendly message.>

> Only list rules that are truly non-negotiable for this app. Drop this section if there are none.

---

## Architecture rules

- Layout: <single-module MVVM under `app/src/main/java/<package>/` — config/ data/ ui/
  services/ utils/ MainActivity.kt>. Do not restructure without instruction.
- Layer boundaries: Composables must not know <SQL / SharedPreferences keys / file paths>.
  ViewModels must not know <Compose imports / navigation routes>.
- Dependency direction: <Composables → ViewModel → Repository → Database/API → Models>.
- Models are immutable (`data class` with `copy()`). Never mutate in place.
- <No direct DB access from Composables — go through the ViewModel/repository layer.>

---

## Build & run commands

```bash
./gradlew assembleDebug          # build debug APK
./gradlew installDebug           # build + install on device/emulator
./gradlew testDebugUnitTest      # run JVM/Robolectric unit tests
./gradlew connectedDebugAndroidTest  # run instrumented tests
./gradlew lint                   # Android lint (must be clean)
```

---

## Build types / flavors   <!-- if flavors are used -->

| Build type / flavor | App ID | Display name | Signing |
|---------------------|--------|--------------|---------|
| debug | <id> | <App Name> Debug | Debug keystore (automatic) |
| release | <id> | <App Name> | Release keystore (keystore.properties) |

---

## Signing / keystore   <!-- if the app ships releases -->

- Keystore file: `<name>.jks` at project root. Alias: `<alias>`. Keep at least one offline backup.
- Create `keystore.properties` (gitignored — never commit).
- `.gitignore` must include: `keystore.properties`, `*.jks`, `*.keystore`.

---

## Security rules

- Never log secrets, keys, tokens, or decrypted data — even in debug builds.
- <Store sensitive data in EncryptedSharedPreferences; never in plain SharedPreferences.>
- Request only the permissions the app needs; <never add INTERNET if the app is offline>.
- <android:allowBackup="false" must remain in the manifest.> (if applicable)

---

## Localization rules   <!-- mandatory for every app: English, Malayalam, Sanskrit -->

- This app ships three languages: **English (`values/`), Malayalam (`values-ml/`), Sanskrit
  (`values-sa/`)**. Every feature and every screen works in all three.
- All user-visible text comes from `strings.xml` via `stringResource()` or `context.getString()` —
  never a raw string literal in a Composable. ViewModels return `@StringRes` / `UiText`, not
  resolved strings.
- Every new string goes into **all three** files with a real translation. Lint
  `MissingTranslation` is an error and is never suppressed or baselined.
- **Sanskrit means Sanskrit, not Hindi in Devanagari.** Follow the engineering standard §8.5 rules
  and glossary; `scripts/check_sanskrit.sh` must pass.
- The language defaults to the system language (English when it is none of the three). The
  Settings language picker (System default / English / മലയാളം / संस्कृतम्) uses
  `AppCompatDelegate.setApplicationLocales` (§8.4). Never remove the locale filter,
  `generateLocaleConfig`, `bundle.language.enableSplit = false`, or the `AppCompatActivity` base.
- Menu, button, label, tab and tooltip strings stay short in all three languages (§8.6); only
  descriptive text may be long.
- Every icon-only control uses `TooltipIconButton` / `TooltipFab` with a localized tooltip (§7.7).
- The About screen is data-driven, localized, and ends with the "Made with ❤️ from India" badge
  (`guideline.md` §1.4).
- Literals are allowed only for logs, non-UI exception messages, asset paths, route names, and
  map/JSON keys.

---

## Code style / naming

- Files: `PascalCase.kt` for classes, `camelCase.kt` acceptable for utility files.
- Classes: `PascalCase`; variables/methods: `camelCase`; constants: `SCREAMING_SNAKE_CASE`.
- Package names: `lowercase` (backtick `in` when it is part of the package path).
- Prefer `val` over `var`, `data class` for models, `sealed class/interface` for state.
- Run ktlint or detekt and keep Android Lint at zero errors before every commit.

---

## Testing rules

- Mirror source package structure in `test/` (e.g. `test/<package>/data/`, `test/<package>/ui/`).
- <Coverage target: e.g. 80% on data and domain layers.>
- Critical areas that must be covered before release: <list them>.
- Add or update a test whenever you add or change a ViewModel/Repository/DAO.

---

## Dependency constraints   <!-- if constrained -->

- Blocked (never add): <e.g. network libraries for an offline app>.
- Before adding any new dependency: check its transitive deps, state why it is
  needed, and confirm it fits the hard rules.

---

## Where things live

```
CLAUDE.md            # this file — project rules
AGENTS.md            # project rules for AI agents / LLMs
docs/                # design docs (Thin profile)
plans/               # one plan per change (see workflow rules)
change_log/          # one log per implemented change
app/src/main/        # app source
app/src/test/        # unit tests (JVM/Robolectric)
app/src/androidTest/ # instrumented tests
```

---

## Workflow rules (mandatory — from global rules)

Every change follows plan-before-changing and log-after-changing:

1. **Plan before changing.** Write a full plan to `plans/` named
   `yyyymmdd_hhMMss_<short-slug>.md` with a `**Status:**` line, the files to change, the issue,
   and the fix. Then **STOP and get explicit approval** before editing/creating/deleting any
   project file (other than the plan). A question or ambiguous reply is not approval.
2. **Log after changing.** After implementing, write a change log to `change_log/` named
   `yyyymmdd_hhMMss_<short-slug>.md` describing what changed and referencing its plan.
3. **Relative paths & privacy only.** `plans/` and `change_log/` files are committed and may become
   public on the internet. They MUST use relative repository paths only (never absolute system
   paths like `C:\...`, `l:\...`, or `file:///...`). They MUST NOT contain any **local system
   details** — OS user name, computer/host name, home or drive-letter paths, network share names,
   LAN/internal IP addresses, local server URLs with ports, device serial numbers, personal email
   addresses — or any secret (API keys, tokens, passwords, keystore passphrases, credentials, PII).
   Write them as if a stranger will read them; nothing should reveal the machine they came from.

Create `plans/` and `change_log/` if they do not exist.

---

## Communication rules

- **Always use simple English.** Write all responses, plans, change logs, and explanations in
  plain, simple English. Short sentences, common words. Explain any jargon you must use.

---

## What Claude must always / never do   <!-- recommended, esp. Thick profile -->

**Always:** <read this file first; state the target layer before adding a class; run lint +
test after changes; keep MainActivity thin.>

**Never:** <put business logic in a Composable; call a DAO from a Composable; edit generated
Room `*_Impl` files; add a blocked dependency; log secrets.>
````

---

## 5. Sections you must always keep verbatim in spirit

Two sections come from the user's global rules and must appear in **every** `CLAUDE.md`, both
profiles, worded the same in meaning:

- **Workflow rules** — plan → approve → log, with relative repository paths only, no local system
  details and no sensitive internet-inappropriate data, `plans/` and `change_log/` naming, and the
  hard approval gate.
- **Communication rules** — always simple English.

Do not shorten these into a single link. Keep the short inline version shown in the template so the
AI always sees them, even in a Thin file.

---

## 6. Per-section writing tips

- **Read-first banner**: one or two lines. State that the file is auto-loaded and must be read
  before any change. Do not pad it.
- **Identity table**: a table beats prose. Include SDK versions, minSdk, package/namespace, and the
  connectivity stance (offline vs online) — the AI uses these constantly.
- **Package naming**: explicitly mention the backtick requirement for the `in` keyword if the
  package starts with `in.` — this catches AI-generated code errors.
- **Hard rules**: number them and phrase each as a testable "must". Put the *why* only when it is
  not obvious. Keep the list to real non-negotiables — a long list of soft rules gets ignored.
- **Architecture**: name the exact packages and the dependency direction. State the one or two
  boundaries that get broken most (Composables ↔ DB is the classic one).
- **Commands**: give copy-paste-ready lines. Use `./gradlew` tasks.
- **Security**: lead with "never log secrets" and the permission stance. Link to `docs/security.md`
  in Thin files instead of inlining the threat model.
- **Testing**: name the *critical* areas by feature (migrations, crash recovery, parsers, crypto).
  Generic "write tests" advice is not useful.
- **Dependencies**: for an offline app, an explicit block list is worth its length — it stops a
  networking package sneaking in as a transitive dep.
- **Dos & Don'ts**: most valuable in Thick files. Keep each line one concrete action, not a theme.

---

## 7. Anti-patterns to avoid

- Repeating a `docs/` file inside a Thin `CLAUDE.md`. Link instead — two copies drift.
- A wall of soft "should" rules. Rules that are not enforced train the reader to skip the section.
- Stale versions and tables. If the identity table says the wrong Kotlin version, trust drops.
- Dropping the workflow or simple-English rules to "save space". They are mandatory.
- Forgetting the backtick `in` keyword note when the package starts with `in.`.

---

## 8. Final self-check before saving a new `CLAUDE.md`

- [ ] Profile chosen (Thin or Thick) and the file matches it.
- [ ] All "Always" sections from the §3 checklist are present.
- [ ] Identity table filled with real versions, minSdk, namespace, connectivity stance.
- [ ] Build commands are copy-paste ready and use `./gradlew` tasks.
- [ ] Workflow rules (plan/approve/log) and simple-English rule are present, inline.
- [ ] `plans/` and `change_log/` entries use relative paths only and contain zero local system details and zero sensitive data — safe to publish on the internet.
- [ ] The three mandatory languages are named: English, Malayalam, Sanskrit — with string parity across `values/`, `values-ml/`, `values-sa/`.
- [ ] The Sanskrit-not-Hindi rule and the in-app language picker rule are present.
- [ ] The tooltip rule (every icon-only control) and the short-label rule are present.
- [ ] The About-screen rule is present, including the "Made with ❤️ from India" badge.
- [ ] Every `<...>` placeholder from the template is replaced or its section deleted.
- [ ] No `docs/` content is duplicated (Thin) / nothing critical is missing (Thick).
- [ ] Links point to files that exist (or are planned) in this project.
- [ ] Section order matches §2.
