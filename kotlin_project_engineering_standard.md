# Kotlin Android Project Engineering Standard

This document is a reusable engineering standard for native Android Kotlin projects using
Jetpack Compose.

It is intentionally layered. Small apps should inherit the core baseline without being forced into
release-process or high-security requirements that do not fit the product.

---

## 1. How To Use This Standard

### 1.1 Conformance Language

Use these terms consistently:

- `MUST`: mandatory for the stated scope.
- `SHOULD`: expected default; deviations require a reason.
- `MAY`: optional.

### 1.2 Applicability Profiles

Every Kotlin Android repository MUST declare which profile applies.

| Profile | Applies To | Purpose |
|---------|------------|---------|
| `Core Baseline` | All Kotlin Android application repositories | Universal maintainability and code-quality rules |
| `Production App Extension` | Apps shipped to real users, external QA, or store review | Release, CI, UX, and environment discipline |
| `Sensitive Data Extension` | Apps handling auth secrets, financial data, health data, PII, or locally encrypted content | Stronger security, storage, logging, and backup rules |

A simple internal tool may use only `Core Baseline`.
A public consumer app will usually use `Core Baseline` plus `Production App Extension`.
An authenticator, password manager, finance, or health app will usually use all three.

### 1.3 Repository Types

This document primarily targets Android application repositories (single-module, single `app/`
project). If the repository is an Android library module, `release/`, `signing/`, and
integration-test requirements apply only if the library ships a runnable sample app.

---

## 2. Core Principles

1. Structure by responsibility first, then by implementation detail.
2. Keep business logic outside Composables.
3. Prefer explicit code over clever abstractions.
4. Enforce standards through tooling where practical.
5. One repository SHOULD have one clear way to do state, navigation, theming, errors, and testing.
6. Optimize for current complexity, not hypothetical future complexity.
7. Security and logging policy are product requirements, not cleanup work.
8. Performance is a feature; it is designed in, not bolted on.
9. Accessibility is a correctness requirement, not a nice-to-have.

---

## 3. Project Structure

### 3.1 Choose The Simplest Layout That Fits

#### Tier 1: Layer-First

Use for smaller apps, single-domain apps, or early products.

```text
app/src/main/java/<package>/
|-- config/          # AppConfig + ConfigService (About). Fixed path — see guideline.md §1.
|-- data/
|   |-- model/       # Data classes / Room entities
|   |-- db/          # Room database, DAOs, migrations
|   `-- repository/  # Repository abstraction
|-- ui/
|   |-- screens/     # Full-page Composable screens
|   |-- components/  # Reusable Composables
|   `-- theme/       # Color, Type, Shape, Theme definitions
|-- services/        # Platform + business services (optional)
|-- utils/           # Small helpers, extensions (optional)
`-- MainActivity.kt
```

The `config/` path is fixed across every tier: the About-screen `AppConfig` model and
`ConfigService` loader always live at `<package>/config/`, as required by `guideline.md §1`.
A small Tier 1 app keeps the rest of the layout flat.

#### Tier 2: Feature-First

Use when multiple product areas evolve independently or when several developers routinely touch
unrelated features.

```text
app/src/main/java/<package>/
|-- config/              # AppConfig + ConfigService (About). Fixed path.
|-- core/
|   |-- database/        # Room database, shared DAOs, migration runner
|   |-- logging/         # Logger service
|   |-- lifecycle/       # ProcessLifecycleOwner callbacks
|   `-- di/              # DI graph root (if using Hilt/Koin)
|-- features/
|   |-- home/
|   |   |-- data/        # Feature-local data (repo, DAO, models)
|   |   `-- ui/          # Feature-local Composables
|   |-- settings/
|   |   |-- data/
|   |   `-- ui/
|   `-- about/
|       `-- ui/          # About screen reads from config/
|-- ui/
|   |-- theme/           # Shared design tokens
|   `-- components/      # Shared Composable components
`-- MainActivity.kt
```

### 3.2 Tier Selection Guidance

| Signal | Tier 1 | Tier 2 |
|--------|--------|--------|
| Number of distinct product areas | 1–3 | 4+ |
| Team size | 1–2 developers | 3+ |
| Feature coupling | Tightly coupled | Loosely coupled |
| Navigation complexity | Simple (tabs or flat) | Multi-graph or deep linking |

Start with Tier 1 unless at least two Tier 2 signals are present. Moving from Tier 1 to Tier 2
is a low-risk refactor; starting with Tier 2 unnecessarily adds friction to small projects.

### 3.3 Root Layout

```text
project-root/
|-- CLAUDE.md                # Mandatory — see CLAUDE_MD_GUIDELINE.md
|-- AGENTS.md                # Mandatory — see AGENTS_MD_GUIDELINE.md
|-- README.md
|-- settings.gradle.kts
|-- build.gradle.kts         # Root project build script
|-- gradle.properties
|-- gradle/
|   |-- libs.versions.toml   # Version catalog
|   `-- wrapper/
|-- keystore.properties      # Signing secrets (git-ignored)
|-- <name>.jks               # Release keystore (git-ignored)
|-- app/
|   |-- build.gradle.kts     # App module build script
|   |-- proguard-rules.pro
|   |-- about.properties     # About-screen config (Pattern B, if used)
|   `-- src/
|       |-- main/
|       |   |-- AndroidManifest.xml
|       |   |-- assets/
|       |   |   `-- config/
|       |   |       `-- app_config.json   # About-screen config (Pattern A, if used)
|       |   |-- java/<package>/           # Kotlin source
|       |   `-- res/                       # Android resources
|       |-- test/                          # JVM / Robolectric tests
|       `-- androidTest/                   # Instrumented tests
|-- docs/
|   |-- GUIDELINES_MANIFEST.md
|   |-- architecture.md
|   `-- guidelines/           # Git submodule pointing to Kotlin_Guidelines
|-- plans/
`-- change_log/
```

---

## 4. Architecture Baseline

### 4.1 Pattern: MVVM with ViewModel

The default architecture pattern is MVVM (Model-View-ViewModel) using Android Architecture
Components:

- **View (Composable)**: UI rendering, input handling, UI-only state
- **ViewModel**: Screen-level state, business logic orchestration, coroutine scope
- **Model**: Data classes, Room entities, repository abstraction

```text
@Composable Screen ── collects ──▶ ViewModel (StateFlow/SharedFlow) ── calls ──▶ Repository ── reads/writes ──▶ Room / API
```

### 4.2 Layer Boundaries

| Layer | May Depend On | Must Not Know About |
|-------|---------------|---------------------|
| Composable (UI) | ViewModel | Room, DAOs, SQL, Retrofit, OkHttp, file I/O |
| ViewModel | Repository, Use Cases, Services | Compose imports, `@Composable`, navigation routes |
| Repository | Room DAOs, Retrofit services, file I/O | ViewModel, Composables |
| Room DAOs / Entities | Android framework (minimal) | ViewModel, Composables |

### 4.3 State Exposure

- ViewModels MUST expose state as `StateFlow<T>` (or `LiveData<T>` for legacy codebases).
  `StateFlow` is preferred for Compose projects because `collectAsStateWithLifecycle()` respects
  the Activity lifecycle automatically.
- One-off events (navigation, toasts, snackbars) SHOULD use `SharedFlow<T>` or a `Channel<T>`
  collected in the Composable.
- Composables MUST NOT hold business state. UI-only state (expanded/collapsed, scroll position,
  text field value before submission) MAY be held in `remember {}` or `rememberSaveable {}`.

```kotlin
class TodoViewModel(private val repository: TodoRepository) : ViewModel() {
    private val _uiState = MutableStateFlow<TodoUiState>(TodoUiState.Loading)
    val uiState: StateFlow<TodoUiState> = _uiState.asStateFlow()

    private val _messages = MutableSharedFlow<String>()
    val messages: SharedFlow<String> = _messages.asSharedFlow()

    fun addTodo(title: String) {
        viewModelScope.launch {
            repository.insert(Todo(title = title))
            _messages.emit("Todo added")
            refreshState()
        }
    }
}
```

### 4.4 Dependency Injection

Manual DI (constructor injection, factory methods, or ViewModel factories) is acceptable for
small projects. Hilt is recommended for medium-to-large projects where the dependency graph
becomes complex.

Rules:
- Do not use more than one DI system in a single project.
- Never pass Android `Context` to a Repository or DAO. If a service needs a `Context`, inject
  `Application` context via the DI graph, not the Activity context.
- ViewModel dependencies MUST be injected, not looked up via singletons.

### 4.5 App Initialization Sequence

Document the initialization order in `docs/architecture.md`. The recommended order:

1. `Application.onCreate()` — register lifecycle observers, init DI (if using Hilt)
2. Database open + migration (Room auto-migration on first query or explicit open)
3. Config load: `ConfigService.loadAndVerify(context)`
4. Logging init (if custom)
5. `MainActivity.onCreate()` → `setContent { }` (Compose entry point)

### 4.6 Models And Entities

- Data models MUST be immutable Kotlin `data class` types. Use `copy()` for mutations — never
  mutate a field in place.
- Room entities are `data class` types annotated with `@Entity`. They MAY live in the `data/model/`
  or `data/db/` package.
- If the app has both a Room entity and a domain/UI model for the same concept, the
  repository is responsible for mapping between them. Composables MUST NOT import Room entity types.
- Sealed classes or sealed interfaces MUST be used for state types (loading/success/error), not
  nullable fields or ad-hoc enums.

```kotlin
@Entity(tableName = "todos")
data class Todo(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    @ColumnInfo(name = "is_completed") val isCompleted: Boolean = false,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis(),
    @ColumnInfo(name = "completed_at") val completedAt: Long? = null,
)
```

---

## 5. Environment And Build Configuration

### 5.1 Gradle Kotlin DSL

All build scripts MUST use the Kotlin DSL (`build.gradle.kts`), not Groovy (`build.gradle`).

### 5.2 Version Catalog

Dependencies MUST be declared in a Gradle version catalog (`gradle/libs.versions.toml`). Direct
`implementation("group:artifact:version")` declarations in `build.gradle.kts` are not allowed
for production dependencies.

### 5.3 Toolchain Requirements

| Concern | Minimum | Notes |
|---------|---------|-------|
| Kotlin | 2.1.x or higher | Pin to a specific minor in the version catalog |
| Compose BOM | 2025.x or higher | Align all Compose library versions |
| AGP (Android Gradle Plugin) | 8.x | AGP 9.x: audit before adopting |
| JDK | Java 17 | AGP 8.x + Gradle 8.14+ requires JDK 17 |
| compileSdk | 36 (API 36) | Or the latest stable SDK |
| targetSdk | 36 (API 36) | Must match Play Store requirements |
| minSdk | 24 (Android 7.0) | Or higher based on app requirements |

### 5.4 Build Features

Enable only what the app uses:

```kotlin
buildFeatures {
    compose = true
    buildConfig = true   // Enable only if BuildConfig fields are used
    // viewBinding = true  // Only if needed for legacy views
}
```

### 5.5 Signing Configuration

See `guideline.md §2` for the keystore source-of-truth. In `app/build.gradle.kts`:

```kotlin
val keystorePropsFile = rootProject.file("keystore.properties")
val keystoreProps = Properties().apply {
    if (keystorePropsFile.exists()) keystorePropsFile.inputStream().use { load(it) }
}

android {
    signingConfigs {
        create("release") {
            storeFile = rootProject.file(keystoreProps.getProperty("storeFile", "release.jks"))
            storePassword = keystoreProps.getProperty("storePassword")
            keyAlias = keystoreProps.getProperty("keyAlias")
            keyPassword = keystoreProps.getProperty("keyPassword")
        }
    }
    buildTypes {
        release {
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

### 5.6 Build Types

| Build Type | `isMinifyEnabled` | `isShrinkResources` | `isDebuggable` | Signing |
|------------|-------------------|---------------------|----------------|---------|
| `debug` | false | false | true (default) | Debug keystore |
| `release` | true (SHOULD) | true (SHOULD) | false (default) | Release keystore |

### 5.7 Product Flavors

Product flavors are optional. Use them when the app needs distinct environments (dev/staging/prod)
or distinct configurations (free/premium).

See `kotlin_build_configuration_guide.md` for the full flavor reference.

### 5.8 16 KB Page Size Compliance

Starting with Android 15 (API 35) and required for Google Play submissions:
- Set `android:extractNativeLibs="true"` or ensure all native libraries in the APK are
  16 KB page-aligned.
- Most pure-Kotlin/Compose apps without custom native code are unaffected. Apps that bundle
  `.so` files (via NDK, native dependencies like ZXing, SQLite custom builds) MUST verify
  alignment.

---

## 6. UI And UX Baseline

### 6.1 Compose Theming

Use Material 3 (`MaterialTheme`) as the foundation. Define design tokens in a central location:

```text
ui/theme/
|-- Color.kt        # Color definitions
|-- Type.kt         # Typography definitions
|-- Shape.kt        # Shape definitions (optional)
`-- Theme.kt        # MaterialTheme wiring (light + dark)
```

Rules:
- Never use hardcoded color values (`Color(0xFF...)`) in Composables. Reference `MaterialTheme.colorScheme`.
- Dark mode MUST be supported or the decision to omit it documented.
- All design tokens (colors, typography, shapes) MUST be defined in the theme package and
  consumed by Composables via `MaterialTheme`.

### 6.2 Composable Structure

- Keep Composable functions focused. Extract sub-Composables when a function exceeds ~120 lines.
- State hoisting: stateless Composables receive state as parameters and emit events via lambdas.
  Stateful wrappers call the ViewModel and pass state down.
- Use `@Preview` annotations for visual iteration during development. Cover both light and dark
  themes in previews.

```kotlin
// Stateless — receives state, emits events
@Composable
fun TodoItem(
    todo: Todo,
    onToggle: (Todo) -> Unit,
    modifier: Modifier = Modifier,
) {
    // ...
}

// Stateful wrapper — connects to ViewModel
@Composable
fun TodoScreen(viewModel: TodoViewModel = viewModel()) {
    val uiState by viewModel.uiState.collectAsStateWithLifecycle()
    when (val state = uiState) {
        is TodoUiState.Loading -> LoadingIndicator()
        is TodoUiState.Empty -> EmptyScreen()
        is TodoUiState.Error -> ErrorScreen(message = state.message)
        is TodoUiState.Content -> TodoScreenContent(
            todos = state.todos,
            onToggle = viewModel::toggleTodo,
        )
    }
}
```

### 6.3 Screen State Pattern

Every screen that loads asynchronous data MUST handle at least these states:

| State | When | What To Show |
|-------|------|--------------|
| Loading | Data is being fetched | Shimmer, placeholder, or centered progress indicator |
| Content | Data loaded successfully | The actual UI |
| Empty | Data loaded but the result set is empty | Empty-state illustration + call to action |
| Error | Data fetch failed | Error message + retry action |

Use a sealed class or sealed interface:

```kotlin
sealed interface TodoUiState {
    data object Loading : TodoUiState
    data class Content(val todos: List<Todo>) : TodoUiState
    data object Empty : TodoUiState
    data class Error(val message: String) : TodoUiState
}
```

### 6.4 User Feedback

- Use `SnackbarHostState` and `Snackbar` for transient confirmations and non-critical errors.
- Use `AlertDialog` for destructive or irreversible actions (delete, reset, purge).
- Never use `Toast` in Compose apps — `Snackbar` integrates with the Compose layout.
- Error messages shown to the user MUST be human-readable. No stack traces, no internal class
  names, no SQL error codes.

### 6.5 Animations And Transitions

- Use `AnimatedVisibility`, `AnimatedContent`, and `Crossfade` for enter/exit transitions.
- Use `animateDpAsState`, `animateColorAsState`, and `animateFloatAsState` for property animations.
- Keep animations short: 150–300 ms for UI transitions, 200–400 ms for page transitions.
- Animations MUST be testable: disable in tests using `testTags` or by setting duration to 0.

### 6.6 Keyboard And Input Handling

- Use `imePadding()` and `WindowInsets.ime` to avoid keyboard overlap.
- Use `LaunchedEffect` to request focus after navigating to a screen with an input field.
- Dismiss the keyboard before submitting a form.

### 6.7 Edge-to-Edge

Android 15 (API 35) enables edge-to-edge display by default. All apps MUST handle system bars
correctly:

- Use `enableEdgeToEdge()` in `MainActivity.onCreate()`.
- Apply `WindowInsets` padding in Composables for safe content areas:
  `Modifier.padding(WindowInsets.systemBars.asPaddingValues())` or use `Scaffold` which
  handles it automatically via `contentWindowInsets`.

---

## 7. Accessibility

This section is `Core Baseline`.

### 7.1 Touch Targets

All interactive elements MUST have a minimum touch target of 48 × 48 dp on mobile. Use
`Modifier.sizeIn(minWidth = 48.dp, minHeight = 48.dp)` or `Modifier.minimumInteractiveComponentSize()`.

### 7.2 Color Contrast

All text and interactive elements MUST meet WCAG AA minimum contrast ratios:

| Element | Minimum Ratio |
|---------|---------------|
| Normal text (< 18 sp) | 4.5:1 |
| Large text (≥ 18 sp) | 3:1 |
| Interactive icons and controls | 3:1 |

Both light and dark themes MUST be checked independently.

### 7.3 Semantics And Content Descriptions

- All interactive Composables (buttons, checkboxes, switches, clickable areas) MUST have a
  `contentDescription` or `semantics { }` label.
- Every icon-only control MUST also show a **tooltip** with the same localized text. Use the
  shared `TooltipIconButton` wrapper from 7.7; do not call `IconButton` directly.
- Decorative images, and icons that sit next to a visible text label, MUST use
  `contentDescription = null` to exclude them from TalkBack.
- Use `Modifier.semantics { stateDescription = ... }` for custom stateful components.
- Labels come from `strings.xml` in all three languages (section 8), so TalkBack speaks the
  user's chosen language.

```kotlin
TooltipIconButton(
    icon = Icons.Default.Delete,
    label = stringResource(R.string.tooltip_delete_todo), // tooltip + contentDescription
    onClick = onDelete,
)
```

### 7.4 Font Scaling

Verify layouts at text scale factors 1.0×, 1.5×, and 2.0×. No text should be clipped or
overflow its container at any of these values. Use `sp` for text sizes (not `dp`).

### 7.5 Focus And Keyboard Navigation

Tab order MUST be logical for keyboard and TalkBack users. Test with TalkBack enabled and with
an external keyboard.

### 7.6 Compliance Verification

Before shipping, verify accessibility:

**Touch target verification:**
Use Android Studio's Layout Inspector to verify interactive elements meet the 48 × 48 dp
minimum.

**Contrast verification:**
Use the [WebAIM Contrast Checker](https://webaim.org/resources/contrastchecker/) or
Android Studio's Visual Linting to verify foreground/background pairs.

**Screen reader verification:**
Enable TalkBack on a device and navigate the entire app. Confirm every interactive element has
a readable label and that decorative elements are excluded.

**Font scaling verification:**
In device Settings → Accessibility → Font size, set to the maximum and verify layouts.

### 7.7 Tooltips On Icon-Only Controls (Mandatory)

**Every control whose only visible content is an icon MUST have a tooltip.** An icon without a
label is a guess for a sighted user. The tooltip explains it on long-press (touch) and on hover
(mouse, ChromeOS), and the same text is the `contentDescription` for TalkBack.

This applies to:

| Control | How the tooltip is supplied |
|---|---|
| `IconButton`, `FilledIconButton`, `FilledTonalIconButton`, `OutlinedIconButton`, `IconToggleButton` | `TooltipIconButton` wrapper |
| `FloatingActionButton`, `SmallFloatingActionButton`, `LargeFloatingActionButton` (icon only) | `TooltipFab` wrapper |
| `TopAppBar` navigation icon, actions, and the overflow (⋮) button | `TooltipIconButton` for each |
| `NavigationBarItem` / `NavigationRailItem` with `alwaysShowLabel = false` or no label | wrap the item's icon in `TooltipBox` with the destination's label |
| Search-bar leading/trailing icons, chip and list-row trailing icon buttons | `TooltipIconButton` |
| Custom icon-only `Modifier.clickable` elements | wrap in `TooltipBox` **and** set `Modifier.semantics { contentDescription = label; role = Role.Button }` |

Rules:

- The tooltip text MUST come from `strings.xml` (prefix `tooltip_` or `action_`, see 8.6), so it
  renders in the user's chosen language like every other string.
- The tooltip names **the action, not the icon**: "Delete note", not "Trash icon".
- Tooltip text follows the short-label budget in 8.6.
- A control that already shows a visible text label next to its icon (e.g. an
  `ExtendedFloatingActionButton` with text, a `NavigationBarItem` with its label shown) does not
  need a tooltip. Adding one is allowed but MUST NOT repeat the label word for word.
- Never use a tooltip as the only way to get information needed to use the app — it is a hint,
  not content.
- Destructive actions still need a confirmation; a tooltip is not a confirmation.

Reference wrapper — `ui/components/TooltipIconButton.kt`:

```kotlin
/**
 * The only way to draw an icon-only button. [label] is shown as the tooltip and
 * used as the contentDescription, so both are always present and localized.
 */
@OptIn(ExperimentalMaterial3Api::class)
@Composable
fun TooltipIconButton(
    icon: ImageVector,
    label: String,
    onClick: () -> Unit,
    modifier: Modifier = Modifier,
    enabled: Boolean = true,
) {
    TooltipBox(
        positionProvider = TooltipDefaults.rememberPlainTooltipPositionProvider(),
        tooltip = { PlainTooltip { Text(label) } },
        state = rememberTooltipState(),
    ) {
        IconButton(onClick = onClick, modifier = modifier, enabled = enabled) {
            Icon(imageVector = icon, contentDescription = label)
        }
    }
}
```

Write `TooltipFab` the same way around `FloatingActionButton`. Keep both wrappers in
`ui/components/`; the CI gate in 19.4 allows raw icon-button calls only in those files.

> **Version note.** `TooltipBox` needs `@OptIn(ExperimentalMaterial3Api::class)`. Newer Material 3
> versions replace `TooltipDefaults.rememberPlainTooltipPositionProvider()` with
> `TooltipDefaults.rememberTooltipPositionProvider(...)` that takes an anchor position. Use the
> name your Compose BOM provides; the rule (tooltip + same localized `contentDescription`) does not
> change.

**Verification** — both are required:

1. `scripts/check_icon_buttons.sh` (19.4) fails the build when a raw icon-button or icon-FAB call
   appears outside the wrapper files.
2. A Compose UI test helper asserts every clickable node without text has a non-empty
   `contentDescription`. Call it in each screen test, in all three locales (18.7):

```kotlin
fun ComposeContentTestRule.assertIconOnlyControlsHaveLabels() {
    val iconOnly = hasClickAction() and SemanticsMatcher.keyNotDefined(SemanticsProperties.Text)
    onAllNodes(iconOnly).fetchSemanticsNodes().forEach { node ->
        val labels = node.config.getOrElse(SemanticsProperties.ContentDescription) { emptyList() }
        check(labels.any { it.isNotBlank() }) { "Icon-only control without a label: $node" }
    }
}
```

---

## 8. Localization And Internationalization

This section is `Core Baseline` and applies to every user-facing app repository.

**Every app ships three languages: English (`en`), Malayalam (`ml`) and Sanskrit (`sa`).** There is
no single-language app. The app starts in the system language when that is one of the three and in
English otherwise, and the user can change the language inside the app at any time. Every feature,
every screen, and every string the app draws — labels, menus, buttons, tooltips, dialogs,
notifications, widgets, errors, empty states, About content — renders in the language the user
selected.

| Sub-section | Rule |
|---|---|
| 8.1 | Minimum build, manifest and activity setup |
| 8.2 | String externalization — no user-visible literals |
| 8.3 | The three languages: formatting, pickers, plurals, fonts, TalkBack |
| 8.4 | In-app language selection, and text shown without an Activity |
| 8.5 | Sanskrit & Malayalam quality — rules, CI gate, standard glossary |
| 8.6 | Short UI labels vs. descriptive text |
| 8.7 | Per-feature language completeness, background text, tests |
| 8.8 | RTL layout support |
| 8.9 | Locale-sensitive formatting |

### 8.1 Minimum Setup (All Apps)

Each item below prevents a specific failure, so none of them is optional. Full Gradle, manifest,
theme and version-catalog snippets are in `kotlin_build_configuration_guide.md`, section
"Languages".

| MUST | What goes wrong without it |
|---|---|
| `res/values/strings.xml` (English, default), `res/values-ml/strings.xml`, `res/values-sa/strings.xml` in **every module** that has user-visible text | The language is missing |
| `res/resources.properties` with `unqualifiedResLocale=en`, and `androidResources { generateLocaleConfig = true }` | Android 13+ does not list the app under system "App languages" |
| The generated locale config lists exactly `en`, `ml`, `sa` — checked once per app and after every AGP upgrade; otherwise a hand-written `res/xml/locales_config.xml` | The system screen offers wrong languages |
| Locale filter `en`, `ml`, `sa` (`androidResources.localeFilters`, or `defaultConfig.resourceConfigurations` on AGP versions without it) | Library strings in ~80 other languages stay in the app; on a Hindi phone the date picker and dialogs show Hindi while the app shows English |
| `bundle { language { enableSplit = false } }` | Play installs only the phone's language; picking another language in the app shows English |
| `androidx.appcompat` 1.6+, `MainActivity : AppCompatActivity`, XML theme parent `Theme.AppCompat.DayNight.NoActionBar` (or a Material Components descendant) | Below Android 13 the language switch does nothing; with the default Compose theme, `AppCompatActivity` crashes at launch |
| Manifest `androidx.appcompat.app.AppLocalesMetadataHolderService` with `autoStoreLocales=true` | Below Android 13 the choice is forgotten after a restart (baseline `minSdk` is 24) |
| Lint `MissingTranslation`, `ExtraTranslation`, `StringFormatMatches`, `StringFormatCount` set to **error**, and never silenced by `tools:ignore`, `@SuppressLint`, `disable`, or a lint baseline (`scripts/check_lint_suppressions.sh`, 19.4) | Missing or broken translations ship |

### 8.2 String Externalization (Mandatory, All Apps)

Every app MUST externalize its user-visible strings into `strings.xml`. This is not optional and
does not wait for a translation request.

Required for every app:

- All three `strings.xml` files exist (8.1).
- Every user-visible string is defined in `strings.xml` and read through
  `stringResource(R.string.key)` in Compose or `context.getString(R.string.key)` in Kotlin.
  A raw string literal in a Composable is not allowed.
- Every `<string>`, `<plurals>` and `<string-array>` exists in **all three** files with a real
  translation. An English value copied into `values-ml` or `values-sa` as a placeholder is an
  unfinished feature, not a translation (8.7).
- `translatable="false"` is allowed only for text that must look the same in every language: the
  language names in the picker (8.4), brand names (including `app_name`, when the app records that
  choice in `docs/architecture.md` §16), and symbols. Such strings live only in `values/`.
- Translator comments (`<!-- ... -->` above a string) go in `values/strings.xml`. Say when a string
  is short UI text, for example `<!-- Toolbar button. Keep to one or two words. -->`.
- String names use the prefixes in 8.6, so the length budget can be checked automatically.

**Narrow exceptions** — these MAY stay as plain Kotlin literals, because a user never reads them:

| Allowed as a literal | Example |
|---|---|
| Log and debug messages | `Log.d(TAG, "cache miss for $id")` |
| Exception messages not shown in the UI | `throw IllegalStateException("db not initialized")` |
| Technical identifiers | Asset paths, route names, map/JSON keys, test tags |
| Developer-only screens | A debug menu that never ships to users |

Anything a real user reads — screen titles, buttons, labels, hints, error text shown on screen,
empty states, snackbars, dialogs, notification text, widget text — goes in `strings.xml`.

Directory structure (repeat in every module with user-visible text):

```text
app/src/main/res/
|-- values/strings.xml      # REQUIRED — English, the default
|-- values-ml/strings.xml   # REQUIRED — Malayalam
|-- values-sa/strings.xml   # REQUIRED — Sanskrit (Devanagari)
`-- resources.properties    # REQUIRED — unqualifiedResLocale=en (app module)
```

**Adding a fourth language later** is a small, mechanical job, because the strings are already
externalized:

1. Add `values-<code>/strings.xml` in every module, with every string translated.
2. Add `<code>` to the locale filter, the language picker (8.4), `formattingLocale` (8.3.1), and
   the parity and label-length tests (8.6, 8.7).
3. Verify the generated locale config.

No screen code changes.

### 8.3 The Three Mandatory Languages

| Locale | Language | Script | Folder | Role |
|---|---|---|---|---|
| `en` | English | Latin | `values/` | Default and fallback |
| `ml` | Malayalam | Malayalam | `values-ml/` | Full UI translation |
| `sa` | Sanskrit | Devanagari | `values-sa/` | Full UI translation (see 8.5) |

AndroidX and Material 3 ship no Sanskrit strings. Under `sa`, their built-in text falls back to the
default English resources. Because of the locale filter (8.1), it can never fall back to Hindi.

#### 8.3.1 Formatting locales

Android's date and number data for `sa` is thin. Depending on the device it can produce Devanagari
digits or placeholder month names such as `M01`. Do not test at runtime whether the device "has
data" — use this fixed mapping everywhere a formatter needs a locale (8.9):

```kotlin
// l10n/FormattingLocale.kt
/**
 * Locale handed to date, time and number formatters. The UI text stays in the
 * user's language; only formatting data is chosen here. Never falls back to Hindi.
 */
fun formattingLocale(appLocale: Locale): Locale = when (appLocale.language) {
    "ml" -> Locale.forLanguageTag("ml-IN-u-nu-latn")
    "en" -> Locale.Builder().setLocale(appLocale).setUnicodeLocaleKeyword("nu", "latn").build()
    else -> Locale.forLanguageTag("en-u-nu-latn") // "sa" and anything unexpected
}
```

- `en` keeps the system region when the system language is English (for example `en-IN` date
  order); `ml` uses `ml-IN`; `sa` uses English formatting.
- Digits are Western (0–9) in all three languages (`nu-latn`), unless the app records a different
  decision in `docs/architecture.md` §16.
- Build locales with `Locale.forLanguageTag` or `Locale.Builder`, not the deprecated `Locale(...)`
  constructors.

#### 8.3.2 Material 3 date and time pickers

`DatePicker`, `DateRangePicker` and `TimePicker` format months and digits from the configuration
locale, not from `formattingLocale`. They MUST be checked under `sa` (18.7). If months or digits
look wrong, run the picker with the formatting locale:

```kotlin
// l10n/WithFormattingLocale.kt
/** Runs [content] with the formatting locale, so Material pickers format correctly under `sa`. */
@Composable
fun WithFormattingLocale(content: @Composable () -> Unit) {
    val configuration = LocalConfiguration.current
    val context = LocalContext.current
    val formattingConfig = remember(configuration) {
        Configuration(configuration).apply { setLocale(formattingLocale(configuration.locales[0])) }
    }
    val formattingContext = remember(context, formattingConfig) {
        context.createConfigurationContext(formattingConfig)
    }
    CompositionLocalProvider(
        LocalConfiguration provides formattingConfig,
        LocalContext provides formattingContext,
    ) { content() }
}
```

- Wrap only the picker and its state (call `rememberDatePickerState()` inside the wrapper), not the
  whole dialog.
- Resolve your own strings (title, confirm and dismiss buttons) **outside** the wrapper and pass
  them in. Inside it, `stringResource` returns English.
- If your Material 3 version lets you pass a locale to the picker state directly, that is equally
  acceptable.

#### 8.3.3 Plurals

Android may have no plural rules for `sa`, so it may only ever use the `other` form. Every
`<plurals>` entry MUST have an `other` item that reads correctly for any number, in all three
files. Malayalam uses `one` and `other`.

#### 8.3.4 Fonts and script coverage

Most Android devices draw Malayalam and Devanagari, but not every manufacturer's image draws them
fully, and a missing glyph shows as an empty box — a silent, ship-blocking bug for two of our three
languages.

- Before every release, open every screen in `ml` and in `sa` on a clean device, including the
  oldest supported Android version, and confirm: no boxes, no clipped tall letters (both scripts
  are taller than Latin), no overflow.
- Never fix text container heights (7.4); let text wrap.
- If glyphs are missing or broken, bundle Noto Sans Malayalam and Noto Sans Devanagari in
  `res/font/` and use them in the typography for those languages (17.4 licensing).
- Record the font decision in `docs/architecture.md` §16.

#### 8.3.5 TalkBack, text input, and the launcher label

- Language names in the picker are written in their own script. Their text SHOULD carry a locale
  span (`SpanStyle(localeList = LocaleList("ml"))`) so TalkBack pronounces them correctly.
- Text fields where the user types Malayalam or Sanskrit SHOULD set
  `KeyboardOptions(hintLocales = LocaleList(...))` (Compose 1.8+), so the keyboard offers that
  language.
- The label under the launcher icon follows the **system** language, not the in-app choice.
  Android controls this; it is an accepted exception (8.7).

### 8.4 In-App Language Selection (Mandatory)

The language is the user's choice, not the device's alone.

**Resolution order:**

1. The language the user picked inside the app, if any.
2. Otherwise the first language in the phone's language list that is `en`, `ml` or `sa`.
3. Otherwise English.

Steps 2 and 3 happen automatically because of the locale filter and the English default `values/`
folder (8.1). The app does not code them.

Rules:

- Set the language with `AppCompatDelegate.setApplicationLocales(...)` on the main thread. An empty
  list means **System default**. Read it with `AppCompatDelegate.getApplicationLocales()`.
- Android 13+ saves the choice itself; below Android 13, AppCompat saves it (`autoStoreLocales`,
  8.1). **The app MUST NOT keep its own copy as the source of truth** — two copies drift apart, for
  example when the user changes the language in system settings. The only allowed copy is the
  read-only background mirror in 8.4.2.
- Below Android 13, AppCompat loads the saved choice from disk when the first Activity is created.
  StrictMode reports this disk read; it is expected. Call `getApplicationLocales()` after
  `MainActivity.onCreate`, never in `Application.onCreate`.
- A change applies **at once and app-wide**; the user is never asked to restart. The Activity is
  recreated, so screen state MUST live in a ViewModel or `rememberSaveable`, and the user MUST stay
  on the same screen (Navigation Compose keeps the back stack across recreation).
- The picker MUST live in Settings, MUST offer **System default** as the first option, and MUST
  list each language in its own script (its endonym), not translated:

  | Option | Shown as |
  |---|---|
  | System default | localized: "System default" / "സിസ്റ്റം സ്വതവേ" / "तन्त्रसिद्धम्" |
  | English | `English` |
  | Malayalam | `മലയാളം` |
  | Sanskrit | `संस्कृतम्` |

- The current choice MUST be visibly marked (radio button), and the Settings row MUST tell TalkBack
  the current value.
- All language reads and writes go through one `LanguageRepository`. Screens MUST NOT call
  `AppCompatDelegate` directly.

#### 8.4.1 Reference implementation

```kotlin
// l10n/LanguageRepository.kt
enum class AppLanguage(val tag: String?) {
    SYSTEM(null), ENGLISH("en"), MALAYALAM("ml"), SANSKRIT("sa"),
}

class LanguageRepository(private val appContext: Context) {

    /** Call after MainActivity.onCreate (see 8.4). */
    fun current(): AppLanguage {
        val language = AppCompatDelegate.getApplicationLocales()[0]?.language
        return AppLanguage.entries.firstOrNull { it.tag == language } ?: AppLanguage.SYSTEM
    }

    fun set(language: AppLanguage) {
        val locales = language.tag?.let { LocaleListCompat.forLanguageTags(it) }
            ?: LocaleListCompat.getEmptyLocaleList()
        BackgroundLanguageMirror.write(appContext, language.tag) // 8.4.2
        AppCompatDelegate.setApplicationLocales(locales)
        NotificationChannels.register(appContext)                // 8.7
    }
}
```

```xml
<!-- values/strings.xml — endonyms look the same in every language, so they are not translated -->
<string name="language_name_en" translatable="false">English</string>
<string name="language_name_ml" translatable="false">മലയാളം</string>
<string name="language_name_sa" translatable="false">संस्कृतम्</string>
```

```kotlin
// ui/screens/settings/LanguagePicker.kt
@Composable
fun LanguagePicker(selected: AppLanguage, onSelect: (AppLanguage) -> Unit) {
    val options = listOf(
        AppLanguage.SYSTEM to stringResource(R.string.label_system_default),
        AppLanguage.ENGLISH to stringResource(R.string.language_name_en),
        AppLanguage.MALAYALAM to stringResource(R.string.language_name_ml),
        AppLanguage.SANSKRIT to stringResource(R.string.language_name_sa),
    )
    Column(Modifier.selectableGroup()) {
        options.forEach { (language, name) ->
            Row(
                modifier = Modifier
                    .fillMaxWidth()
                    .selectable(
                        selected = language == selected,
                        onClick = { onSelect(language) },
                        role = Role.RadioButton,
                    )
                    .padding(horizontal = 16.dp, vertical = 12.dp),
                verticalAlignment = Alignment.CenterVertically,
            ) {
                RadioButton(selected = language == selected, onClick = null)
                Spacer(Modifier.width(16.dp))
                Text(name)
            }
        }
    }
}
```

#### 8.4.2 Text shown without an Activity (below Android 13)

Below Android 13, AppCompat applies the chosen language to Activities only, and it loads the saved
choice only when the first Activity is created. Code that runs without an Activity —
notifications, WorkManager jobs, widgets, services, anything using `applicationContext` — would
otherwise use the system language.

Rules:

- `LanguageRepository.set` writes a **read-only mirror** of the choice for background code.
  `MainActivity.onCreate` overwrites the mirror from `getApplicationLocales()`, so the mirror
  always follows AppCompat, never the other way round.
- Background code gets its strings from `context.localizedContext()`, never straight from
  `applicationContext`.
- On Android 13+ the platform applies the language to the whole app, so the helper returns the
  context unchanged.

```kotlin
// l10n/LocalizedContext.kt
object BackgroundLanguageMirror {
    private const val PREFS = "background_language_mirror"
    private const val KEY = "language_tag"

    fun write(context: Context, tag: String?) {
        context.getSharedPreferences(PREFS, Context.MODE_PRIVATE).edit().putString(KEY, tag).apply()
    }

    fun read(context: Context): String? =
        context.getSharedPreferences(PREFS, Context.MODE_PRIVATE).getString(KEY, null)
}

/** A context whose resources use the in-app language, for code without an Activity. */
fun Context.localizedContext(): Context {
    if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) return this
    val tag = BackgroundLanguageMirror.read(this) ?: return this // System default
    val config = Configuration(resources.configuration).apply {
        setLocales(LocaleList.forLanguageTags(tag))
    }
    return createConfigurationContext(config)
}
```

```kotlin
class MainActivity : AppCompatActivity() {
    override fun onCreate(savedInstanceState: Bundle?) {
        super.onCreate(savedInstanceState)
        BackgroundLanguageMirror.write(this, AppCompatDelegate.getApplicationLocales()[0]?.language)
        enableEdgeToEdge()
        setContent { AppTheme { AppNavHost() } }
    }
}
```

### 8.5 Sanskrit & Malayalam Quality — Standard UI Glossary

Every app ships three languages: English, Malayalam, and Sanskrit (8.3). Both Malayalam and Sanskrit
demand deliberate linguistic care to avoid common pitfalls: Hindi leakage in Sanskrit due to the
shared Devanagari script, and awkward English transliterations or calques in Malayalam.

#### 8.5.1 Sanskrit Quality Rules (Pure Sanskrit, Never Hindi)

Sanskrit's derivational system — verbal roots (`धातु`), prefixes (`उपसर्ग`), suffixes
(`कृत्` / `तद्धित प्रत्यय`), and compounds (`समास`) — can derive a term for any UI concept.

Because Sanskrit and Hindi share the Devanagari script, Hindi text *looks* like Sanskrit to anyone
who does not read it. Never use Hindi anywhere as a substitute, crutch, or fallback for Sanskrit.
`values-sa/strings.xml` MUST be authentic, uncompromised Sanskrit.

- **Classical vocabulary and grammar**: Use authentic Sanskrit nominal stems, proper case endings,
  and correct verbal forms (e.g. polite passive imperative `परिवर्त्यताम्`, not Hindi `बदलें`).
- **No transliterated English loans**: Never transliterate English words into Devanagari when a
  standard Sanskrit word exists (`सेटिंग्स` is Hindi/English in Devanagari; use `विन्यासः`).
- **No Hindi function words or syntax**: Do not use Hindi postpositions (`का`, `की`, `के`, `को`,
  `में`, `से`, `पर`), copulas (`है`, `हैं`, `था`, `थे`, `थी`, `हूं`), or verb endings (`करें`,
  `करना`, `रहा`, `गया`, `चाहिए`).
- **No nukta consonants**: The Perso-Arabic consonants with nukta (`क़`, `ख़`, `ग़`, `ज़`, `ड़`, `ढ़`,
  `फ़`) do not occur in Sanskrit.
- **Strict grammatical agreement**: Participles and adjectives must agree with their subject in
  gender and case. In "No data found", `दत्तांशः` is masculine nominative, so the participle must be
  `प्राप्तः` and the indefinite pronoun `कोऽपि`: `न कोऽपि दत्तांशः प्राप्तः` (never neuter `न किमपि दत्तांशं प्राप्तम्`).
- **Valid morphological derivation**:
  - Do not invent verbs by slapping verbal endings onto nouns. "Copy" is `प्रतिलिख्यताम्` (from verb
    root `लिख्` with `प्रति`) or `प्रतिलिपिः क्रियताम्`, never pseudo-verb `प्रतिलिप्यताम्`.
  - The past passive participle for "Copied" is `प्रतिलिखितम्` (or `प्रतिलिपीकृतम्`), never `प्रतिलिपितम्`.
  - Causative passive of `या` (go) is `निर्याप्यते` / `निर्याप्यताम्` (Export), never `निर्यात्यताम्`.
  - "Confirm" is `स्थिरीक्रियताम्` or `दृढीक्रियताम्` (let it be made firm), never `संपुष्यताम्` (which means "let it be nourished").
  - Do not use Hindi loanwords for concepts that have native Sanskrit terms (use `उपयोक्तृविवरणम्` for Account, never Hindi `खाता`; `लेखा` strictly means a line/furrow).
  - Use `ध्वनिः` for audio/sound to avoid confusion with `शब्दः` (Word).
- **Form conventions**:
  - A button or menu item (action commanding the app): polite `-ताम्` imperative (`लोट्`). For a verb
    that takes an object it is passive (`कर्मणि`), e.g. `रक्ष्यताम्` (Save), `अन्विष्यताम्` (Search); for a
    verb that takes no object it is impersonal (`भावे`), e.g. `निष्क्रम्यताम्` (Exit).
  - A title, tab, label, heading, or status: nominal / abstract noun, e.g. `अन्वेषणम्` (Search), `विन्यासः` (Settings).
  - A confirmation or boolean response: indeclinable, e.g. `आम्` (Yes), `न` (No), `अस्तु` (OK).
  - Direction words (Back, Next, Previous, More) are nominal or adverbial labels and MAY keep that
    form on a button, like Yes / No. Close and Exit are actions: on a button they MUST use the
    imperative (`पिधीयताम्`, `निष्क्रम्यताम्`); the nominal form (`निष्क्रमणम्`) is for titles and labels.
- **Punctuation**: Use the **daṇḍa** `।` to end a sentence in descriptive prose; UI labels take no terminator.
- **File location**: Sanskrit strings live in `values-sa/strings.xml` of every module that has
  user-visible text, plus `sa` values in `app_config.json` and `*_sa.*` content assets.
- **Pre-release review**: Machine translation tools commonly output Hindi for Sanskrit requests. All
  Sanskrit strings MUST be reviewed by a fluent reader before release.

**Forbidden markers.** None of these tokens may appear anywhere in Sanskrit text
(`values-sa/*.xml`, `sa` values in `app_config.json`, `*_sa.*` assets). They are reliable
Hindi giveaways and make a good grep-based gate:

```text
है  हैं  था  थे  थी  हूं  हो  करें  करना  करके  रहा  रही  रहे  गया  गयी  चाहिए
नहीं  और  लेकिन  क्या  आपका  आपकी  आपके  हमारा  मेरा  कृपया  सेटिंग्स  ऐप
◌़ (nukta U+093C, and the precomposed nukta letters U+0958–U+095F)
```

`scripts/check_sanskrit.sh` — required in every app, run in CI (19.4):

```bash
#!/usr/bin/env bash
# Fail when a Hindi marker appears in Sanskrit text (standard 8.5).
# Scans every module's values-sa/*.xml, app_config.json (only its "sa" values are
# Devanagari), and *_sa.* asset files (including Markdown help pages).
# Standalone words are matched between word edges: whitespace, quotes, brackets,
# punctuation, daṇḍa, XML/HTML tag edges (< >), and Markdown marks (* _ ` # | : ; ~ -).
set -u
PATTERN='(?<=[\s"'\''([{<>।,*_`#|:;~-]|^)(?:था|थे|थी|हो|है|हैं|हूं|और)(?=[\s"'\''\)\]}<>।,.\?!*_`#|:;~-]|$)|करें|करना|करके|रहा|रही|रहे|गया|गयी|चाहिए|नहीं|लेकिन|क्या|कृपया|सेटिंग्स|ऐप|\x{093C}|[\x{0958}-\x{095F}]'

mapfile -t FILES < <(find . -path '*/build' -prune -o -type f \( \
    -path '*/src/main/res/values-sa/*.xml' -o \
    -path '*/src/main/assets/*_sa.*' -o \
    -path '*/src/main/assets/config/app_config.json' \) -print)

if [ "${#FILES[@]}" -eq 0 ]; then
  echo 'No Sanskrit files found — every app ships values-sa/strings.xml.'; exit 1
fi

# Self-test: the pattern must still catch a Hindi copula in XML, JSON and Markdown text.
for sample in '<string name="x">है</string>' '"sa": "है"' 'यह **है**' 'यह `है`' 'वह *था*'; do
  if ! printf '%s\n' "$sample" | LC_ALL=C.UTF-8 grep -qP "$PATTERN"; then
    echo "check_sanskrit.sh self-test failed on: $sample"; exit 1
  fi
done

if LC_ALL=C.UTF-8 grep -nP "$PATTERN" "${FILES[@]}"; then
  echo 'Hindi markers found in Sanskrit text (standard 8.5).'; exit 1
fi
exit 0
```

> The gate is a smoke test, not a proof of correctness: passing it means no obvious Hindi marker is
> present, not that the Sanskrit is good. A fluent reader still reviews the text.
> Standalone words (था, थे, थी, हो, है, हैं, हूं, और) match only between word edges: whitespace,
> quotes, brackets, punctuation (including the daṇḍa `।`), the tag edges `<` and `>`, and the
> Markdown marks `*`, `_`, `` ` ``, `#`, `|`, `:`, `;`, `~`, `-`. So `<string name="x">है</string>`
> and `यह **है**` in a help file always fail the build, while legitimate Sanskrit such as
> `स्थाप्यताम्`, `स्थानम्`, `पुनःस्थाप्यताम्`, `यथा`, `तथा` and `कथा` never does.

#### 8.5.2 Malayalam Quality Rules (Natural Malayalam, Not English Transliterations)

Malayalam UI strings must sound natural and idiomatic to native Malayalam speakers.

- **Avoid lazy English transliterations; established loanwords allowed**: Do not phonetically
  transliterate English UI jargon into Malayalam script when standard, authentic Malayalam words exist.
  - Save: `സൂക്ഷിക്കുക` (never bare `സേവ്`).
  - Print: `അച്ചടിക്കുക` (never `പ്രിന്റ്`).
  - Vibration: `കമ്പനം` (never `വൈബ്രേഷൻ`).
  - Optional: `ഐച്ഛികം` (never `ഓപ്ഷണൽ`).
  - Number: `സംഖ്യ` (never `നമ്പർ`).
  - Page: `താൾ` (never `പേജ്`).
  - Widely established digital loanwords (such as `ഹോം`, `മെനു`, `പ്രൊഫൈൽ`, `അക്കൗണ്ട്`, `ഡൗൺലോഡ്`,
    `ഓഫ്‌ലൈൻ`, `തീം`, `ഫയൽ`, `ഫോൾഡർ`, `ലിങ്ക്`, `ലൈസൻസ്`) are accepted where no single native term
    carries universal recognition.
- **Action buttons use verb forms**: Action buttons commanding an operation MUST use the verbal
  form ending in `-ക്കുക` / `-ക` (`തിരുത്തുക`, `സൂക്ഷിക്കുക`, `നീക്കുക`, `തുറക്കുക`, `പുറത്തുകടക്കുക`,
  `ലോഗൗട്ട് ചെയ്യുക`), never a bare English noun or uninflected loan.
- **Accurate negation (`ഇല്ല` vs `അല്ല`)**:
  - `ഇല്ല` denotes non-existence, absence, or refusal to perform an action. For confirmation dialog
    action buttons (Yes / No), use **`അതെ` / `ഇല്ല`**.
  - `അല്ല` denotes negation of identity or qualification ("is not", e.g. `ശരിയല്ല`). Do not put
    `അല്ല` on a confirmation prompt's "No" button when the dialog asks if an action should be done.
- **Avoid ungrammatical standalone postpositions**: Postpositions like `കുറിച്ച്` govern an accusative
  noun (e.g. `ആപ്പിനെക്കുറിച്ച്`); standing alone as a screen title or heading, `കുറിച്ച്` is
  ungrammatical. Use `ആപ്പിനെക്കുറിച്ച്` for "About"; `വിവരണം` is reserved for Description.
- **Contextual accuracy over literal calques**:
  - Preferences: `താൽപ്പര്യങ്ങൾ` or `ഇഷ്ടങ്ങൾ` (matches Sanskrit `रुचयः`). `മുൻഗണനകൾ` strictly means
    **Priorities** (precedence/rank) and is a misleading false friend.
  - Apply (theme/filters): `പ്രയോഗിക്കുക` or `നടപ്പിലാക്കുക`. `ബാധകമാക്കുക` means legal liability/enforcement.
  - Sort: `ക്രമീകരിക്കുക` (arrange in order / sort sequence). `അടുക്കുക` means to stack or draw near.
- **Modern Unicode orthography**: Always use standard Unicode Malayalam atomic chillu characters (`ൺ`, `ൻ`, `ർ`, `ൽ`, `ൾ`). Avoid legacy ZWJ sequences or non-standard glyphs.

#### 8.5.3 Bad → Good Translations

| English | Bad (Hindi / English loan / Calque) | Good (Sanskrit) | Good (Malayalam) | Linguistic Rationale |
|---|---|---|---|---|
| Settings | सेटिंग्स / സെറ്റിംഗ്സ് | विन्यासः | ക്രമീകരണങ്ങൾ | Standard native terminology |
| Save | सेव करें / സേവ് | रक्ष्यताम् | സൂക്ഷിക്കുക | Polite imperative in SA; `-ക്കുക` verb in ML |
| Delete | डिलीट करें / ഡിലീറ്റ് | लुप्यताम् / विलुप्यताम् | ഇല്ലാതാക്കുക | Authentic verbal action |
| Cancel | कैंसिल / ക്യാൻസൽ | निरस्यताम् | റദ്ദാക്കുക | Native rejection/dismissal term |
| Copy | कॉपी करें / കോപ്പി | प्रतिलिख्यताम् | പകർത്തുക | `प्रति + लिख्` verb in SA; NOT `प्रतिलिप्यताम्` |
| Export | निर्यात करें / എക്സ്പോർട്ട് | निर्याप्यताम् | കയറ്റുമതി ചെയ്യുക | Correct causative passive of `या` in SA |
| Search | खोजें / സെർച്ച് | अन्वेषणम् (title) / अन्विष्यताम् (action) | തിരയുക | Distinct noun title vs. action button |
| No data found | कोई डेटा नहीं मिला / ഡാറ്റ ഇല്ല | न कोऽपि दत्तांशः प्राप्तः | വിവരങ്ങളൊന്നും കണ്ടെത്തിയില്ല | Gender agreement in SA (`दत्तांशः` masculine nom.) |
| Preferences | प्रेफरेंसेस / മുൻഗണനകൾ | रुचयः | താൽപ്പര്യങ്ങൾ / ഇഷ്ടങ്ങൾ | `മുൻഗണനകൾ` means priorities, not preferences |
| Confirm | संपुष्यताम् / കൺഫേം | स्थिरीक्रियताम् / दृढीक्रियताम् | സ്ഥിരീകരിക്കുക | `पुष्` means nourish; `स्थिरी` means confirm |
| Print | प्रिंट करें / പ്രിന്റ് | मुद्र्यताम् | അച്ചടിക്കുക | Standard Malayalam verb |
| About | ऐप के बारे में / കുറിച്ച് / परिचयः | विषयपरिचयः | ആപ്പിനെക്കുറിച്ച് | `കുറിച്ച്` is a bound postposition, not a title; `विषये` is locative ("regarding"). One term per meaning: `परिचयः` is Profile and `വിവരണം` is Description (8.5.4). |

#### 8.5.4 Standard UI Glossary

Use these exact terms across all apps, in all three languages. When a term you need is missing, add it
**here**, in this standard, rather than inventing inconsistent per-app variants. All short UI terms
fit the 8.6 budget (checked with the stricter count described there).

**Review rule.** A new or changed Malayalam or Sanskrit glossary term MUST be reviewed by a fluent
reader before any app uses it. The change that adds the term lists it in its change log as
"needs native-reader review" until that review is done.

##### Navigation and structure

| English | Malayalam | Sanskrit |
|---|---|---|
| Home | ഹോം | गृहम् |
| Back | പിന്നോട്ട് | प्रत्यागमनम् |
| Next | അടുത്തത് | अग्रिमम् |
| Previous | മുമ്പത്തേത് | पूर्वम् |
| Menu | മെനു | सूची |
| More | കൂടുതൽ | अधिकम् |
| Close | അടയ്ക്കുക | पिधीयताम् |
| Exit (button) | പുറത്തുകടക്കുക | निष्क्रम्यताम् |
| Exit (title, label) | പുറത്തുകടക്കൽ | निष्क्रमणम् |
| Profile | പ്രൊഫൈൽ | परिचयः |
| Notifications | അറിയിപ്പുകൾ | सूचनाः |
| Favorites | പ്രിയപ്പെട്ടവ | प्रियाणि |
| History | നാൾവഴി | इतिवृत्तम् |
| Details | വിശദാംശങ്ങൾ | विवरणम् |
| List | പട്ടിക | आवली |
| Category | വിഭാഗം | वर्गः |
| Page | താൾ | पृष्ठम् |
| Section | ഖണ്ഡം | खण्डः |

##### Actions (buttons, menu items)

| English | Malayalam | Sanskrit |
|---|---|---|
| Save | സൂക്ഷിക്കുക | रक्ष्यताम् |
| Cancel | റദ്ദാക്കുക | निरस्यताम् |
| Delete | ഇല്ലാതാക്കുക | लुप्यताम् |
| Edit | തിരുത്തുക | सम्पाद्यताम् |
| Add | ചേർക്കുക | योज्यताम् |
| Remove | നീക്കുക | अपनीयताम् |
| Create | സൃഷ്ടിക്കുക | सृज्यताम् |
| Update | നവീകരിക്കുക | अद्यतनीक्रियताम् |
| Copy | പകർത്തുക | प्रतिलिख्यताम् |
| Paste | ഒട്ടിക്കുക | स्थाप्यताम् |
| Undo | പഴയപടിയാക്കുക | प्रत्यावर्त्यताम् |
| Redo | വീണ്ടും ചെയ്യുക | पुनःक्रियताम् |
| Search | തിരയുക | अन्विष्यताम् |
| Filter | അരിക്കുക | परिशोध्यताम् |
| Sort | ക്രമീകരിക്കുക | क्रमीक्रियताम् |
| Refresh | പുതുക്കുക | नवीक्रियताम् |
| Share | പങ്കിടുക | वितीर्यताम् |
| Send | അയയ്ക്കുക | प्रेष्यताम् |
| Download | ഡൗൺലോഡ് ചെയ്യുക | अवतार्यताम् |
| Upload | അപ്‌ലോഡ് ചെയ്യുക | आरोप्यताम् |
| Import | ഇറക്കുമതി ചെയ്യുക | आनीयताम् |
| Export | കയറ്റുമതി ചെയ്യുക | निर्याप्यताम् |
| Print | അച്ചടിക്കുക | मुद्र्यताम् |
| Select | തിരഞ്ഞെടുക്കുക | चीयताम् |
| Select all | എല്ലാം തിരഞ്ഞെടുക്കുക | सर्वं चीयताम् |
| Clear | മായ്ക്കുക | रिक्तीक्रियताम् |
| Reset | പുനഃസജ്ജമാക്കുക | पुनःसज्जीक्रियताम् |
| Confirm | സ്ഥിരീകരിക്കുക | स्थिरीक्रियताम् |
| Apply | പ്രയോഗിക്കുക | प्रयुज्यताम् |
| Open | തുറക്കുക | उद्घाट्यताम् |
| Start | ആരംഭിക്കുക | आरभ्यताम् |
| Stop | നിർത്തുക | विरम्यताम् |
| Pause | നിർത്തിവയ്ക്കുക | स्थग्यताम् |
| Resume | പുനരാരംഭിക്കുക | पुनरारभ्यताम् |
| Continue | തുടരുക | अनुवर्त्यताम् |
| Skip | ഒഴിവാക്കുക | त्यज्यताम् |
| Retry | വീണ്ടും ശ്രമിക്കുക | पुनः प्रयत्यताम् |
| Login | പ്രവേശിക്കുക | प्रविश्यताम् |
| Logout | ലോഗൗട്ട് ചെയ്യുക | निर्गम्यताम् |

##### Settings and preferences

| English | Malayalam | Sanskrit |
|---|---|---|
| Settings | ക്രമീകരണങ്ങൾ | विन्यासः |
| Preferences | താൽപ്പര്യങ്ങൾ | रुचयः |
| Language | ഭാഷ | भाषा |
| Theme | തീം | रूपविन्यासः |
| Dark mode | ഇരുണ്ട രൂപം | श्यामरूपम् |
| Light mode | തെളിഞ്ഞ രൂപം | दीप्तरूपम् |
| System default | സിസ്റ്റം സ്വതവേ | तन्त्रसिद्धम् |
| Font size | അക്ഷരവലുപ്പം | अक्षरपरिमाणम् |
| Sound | ശബ്ദം | ध्वनिः |
| Vibration | കമ്പനം | कम्पनम् |
| Backup | കരുതൽശേഖരം | प्रतिलिपिरक्षणम् |
| Restore | പുനഃസ്ഥാപിക്കുക | पुनःस्थाप्यताम् |
| Permissions | അനുമതികൾ | अनुमतयः |
| Account | അക്കൗണ്ട് | उपयोक्तृविवरणम् |
| Privacy | സ്വകാര്യത | गोपनीयता |
| Security | സുരക്ഷ | सुरक्षा |
| Storage | സംഭരണം | सङ्ग्रहः |
| Data | വിവരങ്ങൾ | दत्तांशः |

##### Status, feedback, and empty states

| English | Malayalam | Sanskrit |
|---|---|---|
| Loading | ലോഡുചെയ്യുന്നു | आपूर्यते |
| Please wait | കാത്തിരിക്കുക | प्रतीक्ष्यताम् |
| Success | വിജയം | सफलम् |
| Failed | പരാജയപ്പെട്ടു | असफलम् |
| Error | പിശക് | दोषः |
| Warning | മുന്നറിയിപ്പ് | पूर्वसूचना |
| Information | വിവരം | सूचना |
| Done | പൂർത്തിയായി | समाप्तम् |
| Empty | ശൂന്യം | रिक्तम् |
| No results | ഫലങ്ങളില്ല | न किमपि प्राप्तम् |
| Offline | ഓഫ്‌ലൈൻ | असंयुक्तम् |
| Online | ഓൺലൈൻ | संयुक्तम् |
| Saved | സൂക്ഷിച്ചു | रक्षितम् |
| Deleted | ഇല്ലാതാക്കി | लुप्तम् |
| Copied | പകർത്തി | प्रतिलिखितम् |
| Updated | നവീകരിച്ചു | अद्यतनीकृतम् |
| Required | ആവശ്യം | आवश्यकम् |
| Optional | ഐച്ഛികം | वैकल्पिकम् |
| Invalid | അസാധു | अमान्यम् |

##### Time and date

| English | Malayalam | Sanskrit |
|---|---|---|
| Date | തീയതി | दिनाङ्कः |
| Time | സമയം | समयः |
| Today | ഇന്ന് | अद्य |
| Yesterday | ഇന്നലെ | ह्यः |
| Tomorrow | നാളെ | श्वः |
| Now | ഇപ്പോൾ | इदानीम् |
| Day | ദിവസം | दिनम् |
| Week | ആഴ്ച | सप्ताहः |
| Month | മാസം | मासः |
| Year | വർഷം | वर्षम् |
| Duration | ദൈർഘ്യം | कालावधिः |

##### Content and fields

| English | Malayalam | Sanskrit |
|---|---|---|
| Title | ശീർഷകം | शीर्षकम् |
| Name | പേര് | नाम |
| Description | വിവരണം | वर्णनम् |
| Note | കുറിപ്പ് | टिप्पणी |
| Text | പാഠം | पाठः |
| Image | ചിത്രം | चित्रम् |
| Audio | ഓഡിയോ | श्रव्यम् |
| Video | വീഡിയോ | दृश्यम् |
| File | ഫയൽ | सञ्चिका |
| Folder | ഫോൾഡർ | संपुटम् |
| Document | രേഖ | लेखः |
| Link | ലിങ്ക് | अनुबन्धः |
| Word | വാക്ക് | शब्दः |
| Line | വരി | पङ्क्तिः |
| Number | സംഖ്യ | सङ्ख्या |
| Total | ആകെ | योगः |
| Count | എണ്ണം | गणना |
| Size | വലുപ്പം | परिमाणम् |
| Type | തരം | प्रकारः |
| Status | നില | स्थितिः |

##### Confirmation words

| English | Malayalam | Sanskrit |
|---|---|---|
| Yes | അതെ | आम् |
| No | ഇല്ല | न |
| OK | ശരി | अस्तु |
| Are you sure? | ഉറപ്പാണോ? | निश्चयेन वा? |

##### About screen (matches the `about_detail_<id>` strings in `guideline.md` §1.3)

| English | Malayalam | Sanskrit |
|---|---|---|
| About | ആപ്പിനെക്കുറിച്ച് | विषयपरिचयः |
| Version | പതിപ്പ് | संस्करणम् |
| Build | നിർമ്മിതി | निर्मितिसङ्ख्या |
| Author | രചയിതാവ് | लेखकः |
| Email | ഇമെയിൽ | विद्युत्पत्रम् |
| License | ലൈസൻസ് | अनुज्ञापत्रम् |
| AI used | ഉപയോഗിച്ച AI | प्रयुक्ता कृत्रिमबुद्धिः |
| IDE used | ഉപയോഗിച്ച IDE | प्रयुक्तं विकाससाधनम् |
| Help | സഹായം | साहाय्यम् |
| Feedback | പ്രതികരണം | प्रतिक्रिया |
| Contact | ബന്ധപ്പെടുക | सम्पर्कः |
| Terms | നിബന്ധനകൾ | नियमाः |
| Privacy policy | സ്വകാര്യതാ നയം | गोपनीयतानीतिः |

> About-screen row labels use the `about_detail_` prefix, which is exempt from the 8.6 budget, so a
> long row label may wrap to two lines. Do not copy that liberty into a toolbar or a tab.

### 8.6 Label Conciseness (Short UI Text vs. Descriptive Text)

UI chrome MUST be short in **all three** languages. A long Malayalam or Sanskrit word wrapping onto
two lines in a toolbar, tab, or bottom-navigation item is a layout bug, and Malayalam and Sanskrit
compounds grow fast if written carelessly.

**Budget for short text** — menu items, buttons, tabs, chips, navigation destinations, tooltips,
app-bar titles, list-row labels, form-field labels, switch/checkbox labels, dialog action buttons:

| Language | Target | Hard limit |
|---|---|---|
| English | 1–2 words | 20 characters |
| Malayalam | 1–2 words | 22 characters |
| Sanskrit | 1 word (nominal form preferred) | 22 characters |

**How characters are counted.** A character is a visible character (a grapheme cluster), not a
code point: a vowel sign or virama belongs to the letter before it. Count with ICU4J
`BreakIterator.getCharacterInstance()` (test-only dependency), so the result is the same on every
machine. ICU versions differ on whether a conjunct such as `ക്ക` is one character or two. The
glossary in 8.5.4 was checked with the stricter count (every consonant counts), and every term fits.

Rules:

- Prefer a single word. Drop articles and filler: "Delete" not "Delete this item".
- In Sanskrit follow the form conventions in 8.5.1: a nominal form for titles, tabs and labels
  (`अन्वेषणम्`), a single-word polite imperative for buttons (`अन्विष्यताम्`). Never a multi-word
  verb phrase.
- In Malayalam prefer the common everyday word over a Sanskritized formal one, unless the app's
  subject matter calls for the formal register.
- Do not solve a long translation by shrinking the font, truncating, or adding an ellipsis —
  choose a shorter word.
- Sentence case in English (`Add note`), not Title Case, and never ALL CAPS in Malayalam or
  Sanskrit.

**Descriptive text is exempt** from the budget — and MUST still be complete, natural prose in all
three languages: onboarding copy, empty-state explanations, help text, About `description`, error
explanations, confirmation dialog bodies, notification bodies, tutorial content.

**String name prefixes make the category checkable:**

| Prefix | Category | Budget |
|---|---|---|
| `action_` | buttons, menu items, dialog actions | short |
| `label_` | field labels, row labels, chips, switches | short |
| `title_` | screen, app-bar and dialog titles | short |
| `tab_`, `nav_` | tabs and navigation destinations | short |
| `tooltip_` | tooltips on icon-only controls (7.7) | short |
| `desc_`, `help_`, `empty_`, `error_`, `body_`, `notification_` | descriptive prose | exempt |
| `about_detail_`, `about_made_with_love` | About rows and badge (`guideline.md` §1.3, §1.4) | exempt |
| `language_name_` | endonyms in the picker (`translatable="false"`) | exempt |

```kotlin
// app/src/test/java/<package>/l10n/LabelLengthTest.kt
import com.ibm.icu.text.BreakIterator

class LabelLengthTest {
    private val shortPrefixes = listOf("action_", "label_", "title_", "tab_", "nav_", "tooltip_")
    private val limits = mapOf("values" to 20, "values-ml" to 22, "values-sa" to 22)

    @Test
    fun shortStringsFitTheBudget() {
        val problems = mutableListOf<String>()
        StringResources.resDirs().forEach { res ->
            limits.forEach { (folder, limit) ->
                StringResources.strings(File(res, folder))
                    .filterKeys { name -> shortPrefixes.any { name.startsWith(it) } }
                    .forEach { (name, value) ->
                        val length = visibleLength(value)
                        if (length > limit) problems += "$res/$folder $name: $length > $limit"
                    }
            }
        }
        assertTrue(problems.joinToString("\n"), problems.isEmpty())
    }

    private fun visibleLength(text: String): Int {
        val iterator = BreakIterator.getCharacterInstance()
        iterator.setText(text)
        var count = 0
        while (iterator.next() != BreakIterator.DONE) count++
        return count
    }
}
```

### 8.7 Per-Feature Language Completeness

A feature is **not done** until it works fully in English, Malayalam and Sanskrit.

- No feature may ship with strings in `values/` only. Lint `MissingTranslation` (8.1) and the parity
  test below fail the build.
- No feature may show English under `ml` or `sa` — including snackbars, validation messages,
  notifications, widgets, share text, exported files a user reads, and the About screen.
- Content shipped as an asset (Markdown help pages, seed data a user reads) MUST exist in all three
  languages as `assets/content/<name>_en.<ext>`, `<name>_ml.<ext>`, `<name>_sa.<ext>`, and the
  screen MUST load the file for the current language.
- Screen tests run in all three locales (18.7). Screenshots for a release are taken in all three
  languages when the feature changes layout.

**Accepted exceptions.** Some text is drawn by Android or Google, not by the app, and follows the
system language: permission dialogs, the system share sheet, the notification shade's own labels,
the launcher icon label, Play in-app review and in-app update dialogs, and Google sign-in screens.
No app can change these. Everything the app itself draws is covered by this section.

#### 8.7.1 No resolved strings in ViewModels or repositories

A ViewModel outlives the language change, so a string it resolved earlier stays in the old
language. ViewModels and repositories MUST return string ids, never resolved text:

```kotlin
// l10n/UiText.kt
sealed interface UiText {
    data class Res(@StringRes val id: Int, val args: List<Any> = emptyList()) : UiText
    /** User-entered or stored data only — never UI chrome. */
    data class Data(val value: String) : UiText
}

@Composable
fun UiText.asString(): String = when (this) {
    is UiText.Res -> stringResource(id, *args.toTypedArray())
    is UiText.Data -> value
}
```

#### 8.7.2 Notifications, workers, widgets

- Code without an Activity builds its text from `context.localizedContext()` (8.4.2):

  ```kotlin
  val text = applicationContext.localizedContext()
  NotificationCompat.Builder(text, NotificationChannels.REMINDERS)
      .setContentTitle(text.getString(R.string.title_reminder))
      .setContentText(text.getString(R.string.body_reminder_due))
  ```

- **Notification channel names** are shown in system settings and keep their old language until
  the channel is registered again. Registering a channel with an existing id updates its name.
  Register channels in `Application.onCreate`, in `Application.onConfigurationChanged` (Android
  13+ language changes from system settings), and after every in-app change
  (`LanguageRepository.set`, 8.4.1):

  ```kotlin
  object NotificationChannels {
      const val REMINDERS = "reminders"

      fun register(context: Context) {
          if (Build.VERSION.SDK_INT < Build.VERSION_CODES.O) return
          val text = context.localizedContext()
          context.getSystemService(NotificationManager::class.java).createNotificationChannel(
              NotificationChannel(
                  REMINDERS,
                  text.getString(R.string.label_channel_reminders),
                  NotificationManager.IMPORTANCE_DEFAULT,
              ),
          )
      }
  }

  class MainApplication : Application() {
      override fun onCreate() {
          super.onCreate()
          NotificationChannels.register(this)
      }

      override fun onConfigurationChanged(newConfig: Configuration) {
          super.onConfigurationChanged(newConfig)
          NotificationChannels.register(this)
      }
  }
  ```

- App widgets re-render their text when the language changes (update them from the same places).

#### 8.7.3 Parity tests (required in every app)

The tests read the source `res/` folders of every module directly, so they run as plain JVM unit
tests. They need `systemProperty("projectRoot", rootDir.absolutePath)` on the test task and the
`org.json` test dependency (build guide, "Languages").

```kotlin
// app/src/test/java/<package>/l10n/StringResources.kt
object StringResources {
    val projectRoot = File(
        requireNotNull(System.getProperty("projectRoot")) {
            "Set systemProperty(\"projectRoot\", rootDir.absolutePath) on the test task."
        },
    )

    /** Every module's src/main/res folder, skipping build output. */
    fun resDirs(): List<File> = projectRoot.walkTopDown()
        .onEnter { it.name != "build" && !it.name.startsWith(".") }
        .filter { it.isDirectory && it.name == "res" && it.parentFile?.name == "main" }
        .toList()

    /** name -> text of translatable <string> elements in one values folder. */
    fun strings(valuesDir: File): Map<String, String> =
        elements(valuesDir, "string").associate { it.getAttribute("name") to it.textContent.trim() }

    /** Names of translatable <string>, <plurals> and <string-array> elements. */
    fun names(valuesDir: File): Set<String> =
        listOf("string", "plurals", "string-array").flatMap { tag ->
            elements(valuesDir, tag).map { "$tag:${it.getAttribute("name")}" }
        }.toSet()

    private fun elements(valuesDir: File, tag: String): List<Element> =
        valuesDir.listFiles { file -> file.extension == "xml" }.orEmpty().flatMap { file ->
            val nodes = DocumentBuilderFactory.newInstance().newDocumentBuilder()
                .parse(file).getElementsByTagName(tag)
            (0 until nodes.length).map { nodes.item(it) as Element }
        }.filter { it.getAttribute("translatable") != "false" }
}
```

```kotlin
// app/src/test/java/<package>/l10n/TranslationParityTest.kt
class TranslationParityTest {
    private val translated = listOf("values-ml", "values-sa")

    /** Strings allowed to equal English: brand names and symbols. Keep this list short. */
    private val sameAsEnglishAllowed = setOf<String>()

    @Test
    fun everyModuleHasTheSameStringsInAllThreeLanguages() {
        val problems = mutableListOf<String>()
        StringResources.resDirs().forEach { res ->
            val english = StringResources.names(File(res, "values"))
            if (english.isEmpty()) return@forEach
            translated.forEach { folder ->
                val other = StringResources.names(File(res, folder))
                (english - other).forEach { problems += "$res/$folder is missing $it" }
                (other - english).forEach { problems += "$res/$folder has extra $it" }
            }
        }
        assertTrue(problems.joinToString("\n"), problems.isEmpty())
    }

    @Test
    fun noTranslationIsACopyOfEnglish() {
        val problems = mutableListOf<String>()
        StringResources.resDirs().forEach { res ->
            val english = StringResources.strings(File(res, "values"))
            translated.forEach { folder ->
                StringResources.strings(File(res, folder)).forEach { (name, value) ->
                    val placeholderOnly = value.replace(Regex("%\\d+\\$[sd]"), "").isBlank()
                    if (value == english[name] && name !in sameAsEnglishAllowed && !placeholderOnly) {
                        problems += "$res/$folder $name is still English"
                    }
                }
            }
        }
        assertTrue(problems.joinToString("\n"), problems.isEmpty())
    }

    @Test
    fun aboutBadgeKeepsTheHeartMarker() {
        StringResources.resDirs().forEach { res ->
            listOf("values", "values-ml", "values-sa").forEach { folder ->
                val text = StringResources.strings(File(res, folder))["about_made_with_love"]
                    ?: return@forEach
                assertEquals("$res/$folder about_made_with_love", 1, Regex("%1\\$s").findAll(text).count())
            }
        }
    }

    @Test
    fun aboutConfigHasAllThreeLanguages() {
        val root = StringResources.projectRoot
        val file = File(root, "app/src/main/assets/config/app_config.json")
        if (!file.exists()) return // Pattern B app (guideline.md §1.2)
        val json = JSONObject(file.readText())
        val labels = StringResources.strings(File(root, "app/src/main/res/values")).keys
        val problems = mutableListOf<String>()

        fun checkLanguages(path: String, value: Any?) {
            if (value !is JSONObject) return
            listOf("en", "ml", "sa").forEach { lang ->
                if (value.optString(lang).isBlank()) problems += "$path.$lang is missing"
            }
        }

        checkLanguages("appName", json.opt("appName"))
        checkLanguages("description", json.opt("description"))
        val details = json.optJSONObject("details")
        details?.keys()?.forEach { id ->
            checkLanguages("details.$id", details.opt(id))
            val label = "about_detail_" + id.replace(Regex("[A-Z]")) { "_" + it.value.lowercase() }
            if (label !in labels) problems += "details.$id has no $label string"
        }
        assertTrue(problems.joinToString("\n"), problems.isEmpty())
    }

    @Test
    fun contentAssetsExistInAllThreeLanguages() {
        val content = File(StringResources.projectRoot, "app/src/main/assets/content")
        val problems = mutableListOf<String>()
        content.listFiles { file -> file.nameWithoutExtension.endsWith("_en") }.orEmpty().forEach { en ->
            listOf("ml", "sa").forEach { lang ->
                val twin = File(en.parentFile, en.name.replace("_en.", "_$lang."))
                if (!twin.exists()) problems += "${twin.name} is missing"
            }
        }
        assertTrue(problems.joinToString("\n"), problems.isEmpty())
    }
}
```

### 8.8 RTL Layout Support

- Never use `left` and `right` for padding, alignment, or positioning of UI elements. Use `start`
  and `end` equivalents in Compose layouts.
- None of our three languages is right-to-left, so this rule keeps the app ready rather than
  supporting a current language. Test RTL with the developer option "Force RTL layout direction" —
  do **not** add Arabic or Hebrew strings or locales, because the language set is fixed at `en`,
  `ml`, `sa` (8.3).

### 8.9 Locale-Sensitive Formatting

Use `java.time.format.DateTimeFormatter`, `java.text.DateFormat`, `java.text.NumberFormat` or
`android.text.format.DateUtils` for all locale-sensitive formatting. Never use `toString()` on
dates, numbers, or currencies in user-visible strings. Always pass `formattingLocale(...)` from
8.3.1, so Sanskrit uses English formatting data with Western digits instead of broken output.

```kotlin
val locale = formattingLocale(LocalConfiguration.current.locales[0])

DateTimeFormatter.ofLocalizedDate(FormatStyle.MEDIUM).withLocale(locale).format(date)
NumberFormat.getNumberInstance(locale).format(value)

// Sort user-visible text with a collator, not plain string comparison.
val collator = Collator.getInstance(locale)
val sorted = items.sortedWith(compareBy(collator) { it.title })
```

`java.time` needs API 26. With the baseline `minSdk 24`, enable core library desugaring or use
`java.text` / `DateUtils` instead.

---

## 9. App Lifecycle Management

### 9.1 ProcessLifecycleOwner

Use `ProcessLifecycleOwner` for app-level lifecycle events (app entering foreground/background).
This is the Kotlin/Android equivalent of Flutter's `WidgetsBindingObserver`.

```kotlin
class AppLifecycleObserver : DefaultLifecycleObserver {
    override fun onResume(owner: LifecycleOwner) {
        // App returned to foreground.
        // Re-validate app lock state.
        // Re-subscribe to data sources if needed.
    }

    override fun onPause(owner: LifecycleOwner) {
        // App moved to background.
        // Trigger app lock if security policy requires it.
        // Flush pending write buffers to disk.
    }

    override fun onStop(owner: LifecycleOwner) {
        // App fully in background.
        // Pause background timers.
    }
}

// In Application.onCreate():
ProcessLifecycleOwner.get().lifecycle.addObserver(AppLifecycleObserver())
```

### 9.2 Required Lifecycle Behaviors

| Event | Required Behavior |
|-------|-------------------|
| `onPause` | Flush unsaved data to DB; trigger app lock if `Sensitive Data Extension` |
| `onPause` | Pause looping animations and background timers |
| `onStop` | Cancel non-critical background work |
| `onResume` | Re-validate app lock; refresh time-sensitive UI state |
| `onDestroy` | Finalize any in-progress DB writes; close open file handles |
| `onTrimMemory` | Clear image cache; release non-critical in-memory buffers |

### 9.3 Database And File Handle Safety

- Room database instances SHOULD be kept as singletons for the app lifetime. Do not create a new
  database instance per operation.
- File handles opened for writing MUST be closed in a `finally` block or via `use { }`.
- Temporary files SHOULD be cleaned up on `onResume` if the previous session ended abnormally.

---

## 10. Performance And Rendering Optimization

### 10.1 Frame Budget

The target rendering budget is:

- **60 Hz displays**: 16 ms per frame.
- **90 Hz / 120 Hz displays**: 11 ms / 8 ms per frame.

Sustained jank above 5% of frames is a release-blocking regression.

### 10.2 Compose Recomposition Optimization

- Use `remember { }` to avoid unnecessary object allocations during recomposition.
- Use `derivedStateOf { }` for computed values that change less frequently than their inputs.
- Avoid passing unstable types to Composables. Prefer `@Immutable` or `@Stable` annotations on
  data classes used as Composable parameters.
- Use `key()` in `LazyColumn` / `LazyGrid` items to help Compose identify items across
  recompositions.

```kotlin
LazyColumn {
    items(todos, key = { it.id }) { todo ->
        TodoItem(todo = todo, onToggle = onToggle)
    }
}
```

### 10.3 List And Grid Performance

- MUST use `LazyColumn`, `LazyRow`, or `LazyVerticalGrid` for any list that is unbounded or can
  grow without a known upper limit. Using `Column` with `forEach` for long lists builds every item
  at composition time regardless of visibility.
- For lists with a known, fixed upper bound of roughly 20 items or fewer, `Column` with
  `forEach` is acceptable.
- Always provide a stable `key` in `items()` calls for correct recomposition behavior.
- Avoid loading all data into memory for very long lists. Implement pagination at the
  repository layer using `Paging 3` or manual cursor-based loading.

### 10.4 Image Handling And Memory

- Use Coil (recommended) or Glide for image loading. Never load images manually with
  `BitmapFactory` in the UI thread.
- Use `size()` modifier on `AsyncImage` to request appropriately sized images.
- For offline apps, use `ImageLoader.Builder(context).respectCacheHeaders(false)` to enable
  aggressive disk caching.
- Provide density-specific drawables (`hdpi`, `xhdpi`, `xxhdpi`, `xxxhdpi`) for raster assets.

### 10.5 Coroutines And Background Work

Any operation that risks blocking the main thread MUST be dispatched to an appropriate
dispatcher.

| Operation | Dispatcher |
|-----------|------------|
| Database reads/writes | `Dispatchers.IO` |
| Network calls | `Dispatchers.IO` |
| JSON parsing of large payloads | `Dispatchers.Default` |
| Image processing | `Dispatchers.Default` |
| File I/O | `Dispatchers.IO` |
| UI updates | `Dispatchers.Main` (default for ViewModel scope) |

- `viewModelScope` uses `Dispatchers.Main.immediate` by default — switch to IO/Default for
  heavy work.
- Use `withContext(Dispatchers.IO) { ... }` for one-off context switches.
- Use `flow { }.flowOn(Dispatchers.IO)` for flows that do I/O work.
- Never use `GlobalScope`. Use `viewModelScope` or a structured `CoroutineScope` with a
  defined lifecycle.

### 10.6 Startup Performance

- The cold startup time target for release builds is under **2 seconds** to first meaningful
  frame on a mid-range device.
- Defer non-critical initialization. Services not needed on the first screen SHOULD be
  initialized lazily.
- Avoid synchronous disk reads in `Application.onCreate()` or `MainActivity.onCreate()`.
  Database migrations and file reads MUST be async.
- Use the `App Startup` library for initializing ContentProviders lazily.
- Use Android Studio Profiler to measure startup phases during development.

### 10.7 App Size Budget

Under `Production App Extension`:

| Platform | Target | Hard Limit |
|----------|--------|------------|
| APK arm64-v8a | Under 30 MB | 50 MB |
| AAB download size | Under 20 MB | 40 MB |

Exceeding the hard limit requires a documented justification in the release checklist.

---

## 11. Error Handling Architecture

### 11.1 Global Error Boundaries

Every Android app SHOULD configure an uncaught exception handler in `Application.onCreate()`:

```kotlin
Thread.setDefaultUncaughtExceptionHandler { thread, throwable ->
    Log.e("UncaughtException", "Uncaught exception on ${thread.name}", throwable)
    // Optionally: write crash info to a file for next-launch reporting.
    // Do NOT show a UI here — the process is terminating.
}
```

For coroutine-level error handling, install a `CoroutineExceptionHandler` on app-level scopes:

```kotlin
val handler = CoroutineExceptionHandler { _, throwable ->
    Log.e("CoroutineError", "Uncaught coroutine exception", throwable)
}
```

### 11.2 Error Classification

Classify errors at the point of catch so that the correct response is taken.

| Class | Definition | Response |
|-------|-----------|----------|
| **Recoverable** | Operation failed but app state is intact | Show inline message, offer retry |
| **Degraded** | A feature is unavailable but the app is usable | Show banner, disable affected section |
| **Session** | The current session must be reset (lock triggered, corruption detected) | Navigate to safe state (lock screen or home), log detail |
| **Fatal** | App cannot continue safely | Show fatal error screen with restart action, log full detail |

Rules:
- Never escalate a recoverable error to a fatal error screen.
- Never silently swallow an error that changes application state.
- Error messages shown to the user MUST be human-readable and actionable. They MUST NOT contain
  stack traces, internal exception class names, or database error codes.

### 11.3 Repository And Service Layer Error Handling

- Repositories MUST catch data-layer exceptions (`SQLiteException`, `IOException`, etc.)
  and re-throw typed domain exceptions.
- Define a sealed domain exception hierarchy:

  ```kotlin
  sealed class AppException : Exception() {
      data class StorageException(
          override val message: String,
          override val cause: Throwable? = null,
      ) : AppException()

      data class ValidationException(
          val field: String,
          override val message: String,
      ) : AppException()
  }
  ```

- ViewModels MUST catch domain exceptions and translate them into UI state.

### 11.4 UI Error Presentation

- Use the standard four screen states (section 6.3) for asynchronous data loading errors.
- Inline field errors MUST appear below the relevant field, not as a Snackbar.
- Operation errors (save failed, delete failed) MUST use a `Snackbar` with a retry action where
  possible.
- Fatal error screens MUST provide: a human-readable description, a primary action (Restart or
  Go Home), and a secondary action to copy diagnostic info to the clipboard.

---

## 12. Code Generation

### 12.1 KSP (Kotlin Symbol Processing)

KSP is the standard code generation tool for Kotlin Android projects. It replaces the older
`kapt` annotation processor and is significantly faster.

Declare KSP in the project-level `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.google.devtools.ksp) apply false
}
```

And in the app module's `build.gradle.kts`:

```kotlin
plugins {
    alias(libs.plugins.google.devtools.ksp)
}

dependencies {
    "ksp"(libs.androidx.room.compiler)
    "ksp"(libs.moshi.kotlin.codegen) // If using Moshi code gen
}
```

### 12.2 Generated File Policy

Generated files from KSP (Room `*_Impl` classes) are always excluded from source control.
They are generated during the build process and MUST NOT be committed.

Add to `.gitignore`:

```gitignore
# KSP generated files
**/build/generated/ksp/
```

### 12.3 Room Entity Pattern

```kotlin
@Entity(tableName = "todos")
data class Todo(
    @PrimaryKey(autoGenerate = true) val id: Long = 0,
    val title: String,
    @ColumnInfo(name = "is_completed") val isCompleted: Boolean = false,
    @ColumnInfo(name = "created_at") val createdAt: Long = System.currentTimeMillis(),
    @ColumnInfo(name = "completed_at") val completedAt: Long? = null,
)
```

Rules:
- Room entities MUST be immutable `data class` types.
- Use `copy()` for all mutations.
- Use `@ColumnInfo(name = "...")` for column names that differ from the Kotlin property name.
- Use `snake_case` for database column names.

---

## 13. Database And Persistence Standard

### 13.1 Room Migration Strategy

Schema changes MUST go through versioned migrations. Never modify the entity structure without
adding a corresponding migration.

```kotlin
@Database(
    entities = [Todo::class],
    version = 3,
    exportSchema = true,
)
abstract class AppDatabase : RoomDatabase() {
    abstract fun todoDao(): TodoDao
}

// Migrations
val MIGRATION_1_2 = object : Migration(1, 2) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("CREATE INDEX IF NOT EXISTS idx_todos_created_at ON todos(created_at)")
    }
}

val MIGRATION_2_3 = object : Migration(2, 3) {
    override fun migrate(db: SupportSQLiteDatabase) {
        db.execSQL("ALTER TABLE todos ADD COLUMN priority INTEGER NOT NULL DEFAULT 0")
    }
}

// Database builder
Room.databaseBuilder(context, AppDatabase::class.java, "app_database")
    .addMigrations(MIGRATION_1_2, MIGRATION_2_3)
    .build()
```

Rules:
- Migrations are append-only. Never modify a migration that has already been shipped.
- Each migration SHOULD be atomic.
- Migrations MUST be covered by tests that exercise the upgrade path from the minimum
  supported version to the current version.
- The current schema version MUST be documented in `docs/architecture.md`.
- Enable `exportSchema = true` to generate JSON schema files for migration testing.

### 13.2 WAL Mode

Room enables WAL (Write-Ahead Logging) by default. Do not disable it unless you have a
documented reason.

### 13.3 Index Strategy

- Add an index on every column used in a `WHERE` clause that filters a table with more than
  approximately 1,000 rows.
- Add a composite index when queries filter on two or more columns together.
- Document indexes in the schema section of `docs/architecture.md`.

### 13.4 Data Integrity Rules

- Use `@ForeignKey` annotations with `onDelete` actions. Never rely on application code to
  clean up orphan records.
- Use `NOT NULL` constraints (non-nullable Kotlin types) on all columns that should never be null.
- Use `@ColumnInfo(defaultValue = "...")` for columns with default values.

---

## 14. Logging Infrastructure

### 14.1 Logging Levels

Use a consistent level taxonomy across the codebase:

| Level | When To Use |
|-------|-------------|
| `Log.v` (verbose) | Extremely detailed: individual DB rows, loop iterations. Dev-only. |
| `Log.d` (debug) | Useful dev context: function entry/exit, query parameters. Dev-only. |
| `Log.i` (info) | Normal significant events: app start, screen load, user action completed. |
| `Log.w` (warning) | Unexpected but recoverable: retry attempted, deprecated path used. |
| `Log.e` (error) | Operation failed: DB write failed, parse error, expected flow broke. |
| `Log.wtf` (fatal) | App cannot continue: unrecoverable state, data corruption detected. |

### 14.2 Recommended Logger Setup

Use consistent `TAG` constants across the codebase:

```kotlin
class TodoRepository {
    private companion object {
        const val TAG = "TodoRepository"
    }

    fun loadTodos(): List<Todo> {
        Log.d(TAG, "Loading todos from database")
        // ...
    }
}
```

For apps that need structured logging, Timber is acceptable:

```kotlin
// In Application.onCreate():
if (BuildConfig.DEBUG) {
    Timber.plant(Timber.DebugTree())
}
```

### 14.3 Logging Rules

- NEVER log: secrets, tokens, passwords, recovery codes, decrypted content, or full database rows
  that may contain PII.
- Log the operation name and error category, not raw exception messages that may contain user data.
- `Log.d` and `Log.v` MUST NOT produce output in production builds. Gate them behind
  `BuildConfig.DEBUG`.
- All `Log.e` and `Log.wtf` calls MUST include the throwable parameter when available.

### 14.4 Log Output In Production

For apps that need log output beyond Logcat:
- Limit the log file to a maximum of **5 MB**. Rotate to a new file when the limit is reached.
- Retain a maximum of **3 rotated log files** before deleting the oldest.
- Store log files in the app's cache directory (`context.cacheDir`), not the files directory.
- Provide a diagnostic log export action in settings UI so logs can be retrieved without
  requiring a device connection.

### 14.5 What To Log At Each Layer

| Layer | What To Log |
|-------|-------------|
| `Application.onCreate()` | Initialization steps, build type, app version |
| Repository | Operation name, record count, duration for slow queries (> 50 ms) |
| ViewModel | Significant state transitions, unexpected branch taken |
| Composable | Screen/feature entered (if needed), key user action completed |
| Error boundary | Full error class, message, and stack at `error` or `fatal` level |

---

## 15. Security Standard

### 15.1 Core Security Rules

These rules apply to all Kotlin Android apps.

- Never log secrets, tokens, private payloads, or decrypted sensitive data.
- Request only the permissions the app actually uses.
- Ask for dangerous permissions at point of use where the platform allows it.
- Production logs SHOULD avoid personal data unless operationally necessary.
- Production release builds SHOULD enable R8 code shrinking (`isMinifyEnabled = true`).

### 15.2 Sensitive Data Extension

Apply this section when the app handles authentication factors, private documents, health data,
financial data, recovery codes, or local encrypted stores.

- Sensitive values MUST NOT be stored in plain `SharedPreferences`.
- Use `EncryptedSharedPreferences` or Android Keystore for keys, tokens, or secret material.
- Use authenticated encryption such as AES-GCM for stored sensitive payloads.
- Never hardcode keys, IVs, salts, recovery passwords, or backup passwords.
- Cryptographic formats SHOULD be versioned so migrations remain possible.
- Screenshot and screen-recording protection MUST be enabled using `FLAG_SECURE`.
- App lock, background lock, and session-expiry behavior MUST be explicit in app state.
- Export of sensitive data SHOULD be encrypted by default; plaintext export, if allowed, MUST be
  explicit and user-confirmed.
- `android:allowBackup` MUST be set to `false` or backup content explicitly restricted.

### 15.3 OWASP Mobile Top 10 Compliance Checklist

Before each production release, verify the following OWASP Mobile Top 10 controls:

| ID | Risk | Control |
|----|------|---------|
| M1 | Improper Credential Usage | No hardcoded secrets; use EncryptedSharedPreferences / Keystore |
| M2 | Inadequate Supply Chain Security | Dependency audit; pin versions in `libs.versions.toml` |
| M3 | Insecure Authentication | App lock with proper background/foreground enforcement |
| M4 | Insufficient Input/Output Validation | Validate all user input; parameterized Room queries |
| M5 | Insecure Communication | TLS only for any network traffic; no cleartext HTTP |
| M6 | Inadequate Privacy Controls | Data inventory reviewed; no PII in logs |
| M7 | Insufficient Binary Protections | R8 `isMinifyEnabled = true` applied to release builds |
| M8 | Security Misconfiguration | `android:debuggable=false` verified; permissions minimal |
| M9 | Insecure Data Storage | No sensitive data in plain `SharedPreferences` or unencrypted files |
| M10 | Insufficient Cryptography | Versioned encrypted formats; secure key derivation |

### 15.4 Data Retention And Purge Policy

Every app that stores user-generated data MUST define and implement a retention policy.

- Document in `docs/security.md`: what data is stored, how long it is retained, and what triggers
  deletion.
- Provide a user-accessible "Delete all data" action.
- Temporary files MUST be deleted within the same session they are created.

---

## 16. Coding Standards

### 16.1 Formatting And Analysis

- Use ktlint or detekt for Kotlin code formatting and static analysis.
- Keep Android Lint at zero errors.
- New work MUST NOT introduce lint warnings.

Recommended baseline lint checks in `app/build.gradle.kts`:

```kotlin
android {
    lint {
        abortOnError = true
        warningsAsErrors = false   // Set to true for stricter enforcement
        checkReleaseBuilds = true
    }
}
```

### 16.2 Size And Complexity Guidance

These are prompts to review, not automatic failures.

| Metric | Guideline |
|--------|-----------|
| File length | Around 300 lines: consider splitting |
| File length | Around 500 lines: split or justify |
| Function length | Around 50 lines: consider extracting |
| Composable function | Around 120 lines: consider sub-Composables |
| Parameters per function | More than 5: consider a parameter object |

### 16.3 Naming

- `PascalCase` for files containing classes, `camelCase` acceptable for utility files.
- `PascalCase` for classes, `camelCase` for variables and functions.
- `SCREAMING_SNAKE_CASE` for constants (`const val` and `companion object` constants).
- `snake_case` for database table and column names.
- `lowercase` for package names (backtick `in` when it is a package segment).
- Prefer explicit names over abbreviations.

### 16.4 Kotlin Idioms

- Prefer `val` over `var`.
- Use `data class` for models and value types.
- Use `sealed class` or `sealed interface` for restricted hierarchies (state, errors).
- Use `when` expressions exhaustively (compiler-enforced for sealed types).
- Use `?.let { }` and `?.also { }` instead of null-check `if` blocks where readability improves.
- Use scope functions (`apply`, `let`, `run`, `with`, `also`) consistently — do not mix patterns.
- Use `use { }` for `Closeable` resources (file streams, cursors).

### 16.5 Comments And Documentation

- Comments SHOULD explain why, not restate what code does.
- KDoc (`/** ... */`) for public API documentation.
- TODOs SHOULD include an owner or issue reference.
- Do not swallow exceptions silently.

### 16.6 Dependencies

- Add dependencies only when they remove meaningful complexity.
- Prefer maintained libraries with clear ownership.
- Review transitive risk for packages that handle auth, storage, files, camera, or encryption.
- Remove unused dependencies promptly.
- For offline-only apps: audit every new dependency to verify it does not introduce transitive
  network activity. Run `./gradlew app:dependencies --configuration releaseRuntimeClasspath`
  and inspect for unexpected network packages.

### 16.7 Dependency Audit Cadence

Under `Production App Extension`:

- Run `./gradlew dependencyUpdates` (requires the Gradle Versions plugin) or check version
  catalog entries manually before each release.
- Review major version upgrades individually.
- Verify all dependency licenses are compatible with your distribution model.
- Pin critical security dependencies to exact versions in `libs.versions.toml` and update
  them deliberately after reviewing changelogs.

---

## 17. Asset Management

### 17.1 Image Format Policy

| Content Type | Required Format | Rationale |
|-------------|-----------------|-----------|
| Photographs, complex gradients | WebP (lossy) | 25–35% smaller than JPEG at equal quality |
| Logos, UI illustrations with transparency | WebP (lossless) or vector drawable | Smaller than PNG; vector preferred if possible |
| Icons (monochrome or multi-color) | Vector drawable (XML) | Resolution-independent |
| Raster fallback (when vector not viable) | PNG with density variants | Only when vector cannot achieve the result |

Never use JPEG for UI assets that require transparency.

### 17.2 Density Variants

Provide density-specific raster assets for all raster images used in the UI:

```text
res/
|-- drawable-hdpi/     # 1.5x
|-- drawable-xhdpi/    # 2x
|-- drawable-xxhdpi/   # 3x (most common phone density)
|-- drawable-xxxhdpi/  # 4x (high-end phones)
`-- drawable/          # Vector drawables (density-independent)
```

Missing density variants cause blurry rendering on devices at that density.

### 17.3 Vector Drawables

Prefer vector drawables (`res/drawable/*.xml`) over raster assets for icons and simple
illustrations:
- Resolution-independent — no density variants needed.
- Smaller APK size than equivalent raster assets at multiple densities.
- All vector drawables MUST work on `minSdk 24` (native vector drawable support).

### 17.4 Font Licensing

- Verify the license of every bundled font before first release.
- OFL (SIL Open Font License) fonts are generally safe for commercial use.
- For downloadable fonts (Google Fonts via the Android framework), verify the font download
  behavior for offline apps — downloadable fonts require network access.
- For offline apps, bundle font files in `res/font/` and reference them in XML or Compose.
- Malayalam and Devanagari fonts bundled for the three app languages (8.3.4) — for example
  Noto Sans Malayalam and Noto Sans Devanagari — are OFL-licensed. Record them and their licence
  in the app's licence notices, and bundle only the weights the app uses.

---

## 18. Testing Standard

### 18.1 Test Levels

| Level | Core Baseline | Production App Extension |
|-------|---------------|--------------------------|
| Unit tests (JUnit + Robolectric) | Required for business logic, models, parsing, validation | Required |
| Compose UI tests | Required for screens with meaningful UI logic | Required |
| Instrumented tests | Optional unless the app has critical end-to-end flows | Required for critical release paths |
| Screenshot tests (Roborazzi) | Optional | Optional but recommended for design systems |
| Performance tests | Optional | Required for screens with complex lists or animations |

### 18.2 Test Rules

- `test/` SHOULD mirror `main/java/` package structure.
- ViewModels, Repositories, and models with non-trivial logic SHOULD have corresponding tests.
- Critical parsing, migration, and security logic MUST use deterministic test vectors where
  available.
- Bug fixes SHOULD add regression tests when feasible.
- Run `./gradlew testDebugUnitTest` after code changes that affect behavior.
- Shared test scaffolding SHOULD live in `test/<package>/helpers/`.
- Database migration tests MUST cover the full upgrade path from version 1 to the current version.

### 18.3 Test Quality

- Tests MUST be independent.
- Use descriptive test names.
- Mock external systems, not the logic under test.
- Important test files SHOULD be runnable in isolation.
- Coverage trends are useful, but arbitrary percentage gates SHOULD NOT replace judgment.

### 18.4 Robolectric Configuration

Robolectric runs Android tests on the JVM without a device. Configuration:

```kotlin
// app/build.gradle.kts
testOptions { unitTests { isIncludeAndroidResources = true } }
```

For compileSdk 36 and later, Robolectric may require Java 21 for the test workers:

```kotlin
tasks.withType<Test>().configureEach {
    javaLauncher.set(
        javaToolchains.launcherFor { languageVersion.set(JavaLanguageVersion.of(21)) },
    )
}
```

### 18.5 Compose UI Testing

```kotlin
@get:Rule
val composeTestRule = createComposeRule()

@Test
fun `shows empty state when no todos exist`() {
    composeTestRule.setContent {
        TodoScreenContent(todos = emptyList(), onToggle = {})
    }
    composeTestRule.onNodeWithText("No todos yet").assertIsDisplayed()
}
```

Use `testTag` for non-text elements:

```kotlin
Modifier.testTag("add_todo_button")
// In test:
composeTestRule.onNodeWithTag("add_todo_button").performClick()
```

### 18.6 Roborazzi Screenshot Tests

Roborazzi captures Composable screenshots for visual regression testing on the JVM:

```kotlin
@RunWith(RobolectricTestRunner::class)
@GraphicsMode(GraphicsMode.Mode.NATIVE)
class TodoScreenRoborazziTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @get:Rule
    val roborazziRule = RoborazziRule(
        composeRule = composeTestRule,
        captureRoot = composeTestRule.onRoot(),
    )

    @Test
    fun todoScreen_emptyState() {
        composeTestRule.setContent {
            AppTheme { TodoScreenContent(todos = emptyList(), onToggle = {}) }
        }
    }
}
```

Commands:
```bash
./gradlew recordRoborazziDebug     # Record new baseline screenshots
./gradlew compareRoborazziDebug    # Compare against baselines
./gradlew verifyRoborazziDebug     # Fail if differences found
```

### 18.7 Localization Tests (Core Baseline)

Every app ships English, Malayalam and Sanskrit (section 8), so these tests are required:

| Test | What it checks | Section |
|---|---|---|
| `TranslationParityTest` (JVM) | Same `<string>`, `<plurals>`, `<string-array>` names in `values`, `values-ml`, `values-sa` of every module; no `ml`/`sa` value copied from English; `app_config.json` language maps complete; every About `details` id has a label string; help assets have `ml`/`sa` twins | 8.7 |
| `LabelLengthTest` (JVM) | Short-prefixed strings within the 8.6 budget | 8.6 |
| Badge format test (JVM) | `about_made_with_love` has `%1$s` exactly once in all three files | `guideline.md` §1.4 |
| Screen tests in three locales (Robolectric) | Each screen test runs under `en`, `ml`, `sa`; no crash, no clipped text, `assertIconOnlyControlsHaveLabels()` passes | 7.7, 8.3 |
| Picker test under `sa` | Date/time pickers render readable months and Western digits | 8.3.2 |

Run a screen test in each language with Robolectric qualifiers:

```kotlin
@RunWith(RobolectricTestRunner::class)
class SettingsScreenLocaleTest {
    @get:Rule
    val composeTestRule = createComposeRule()

    @Test @Config(qualifiers = "en")
    fun settings_english() = checkSettings()

    @Test @Config(qualifiers = "ml")
    fun settings_malayalam() = checkSettings()

    @Test @Config(qualifiers = "sa")
    fun settings_sanskrit() = checkSettings()

    private fun checkSettings() {
        composeTestRule.setContent { AppTheme { SettingsScreenContent(state = previewState) } }
        composeTestRule.assertIconOnlyControlsHaveLabels()
    }
}
```

When the project uses Roborazzi, record screenshots of changed screens in all three qualifiers.

---

## 19. CI Standard

### 19.1 Minimum CI

All active app repositories SHOULD have CI on pull requests or on the merge path to the protected
branch.

Minimum checks:

```yaml
steps:
  - run: ./gradlew lint
  - run: ./gradlew testDebugUnitTest
  - run: ./gradlew assembleDebug
```

### 19.2 Production App Extension

For shipped apps:

```yaml
steps:
  - run: ./gradlew lint
  - run: ./gradlew testDebugUnitTest
  - run: ./gradlew assembleRelease
  - run: ./gradlew bundleRelease
```

Additional recommended steps:
- Dependency license check.
- App size analysis: compare APK/AAB size against the project's size budget.
- Roborazzi screenshot verification: `./gradlew verifyRoborazziDebug`.

### 19.3 Pre-Commit

A pre-commit hook MAY run formatting and analysis locally, but CI remains the source of truth.

### 19.4 Language, Tooltip, And Lint Gates (All Apps)

These gates enforce rules that review alone misses. When the repository has CI, these steps MUST
be in it. Without CI, run them before every release (`release_process.md` §8).

```yaml
steps:
  - run: bash scripts/check_sanskrit.sh           # 8.5 — no Hindi markers in Sanskrit text
  - run: bash scripts/check_icon_buttons.sh       # 7.7 — icon buttons only via the wrappers
  - run: bash scripts/check_lint_suppressions.sh  # 8.1 — translation lint never silenced
  - run: ./gradlew lint testDebugUnitTest         # lint errors + parity and label-length tests
```

`scripts/check_sanskrit.sh` is given in 8.5.1.

`scripts/check_icon_buttons.sh`:

```bash
#!/usr/bin/env bash
# Fail when an icon-only button is drawn without the tooltip wrappers (standard 7.7).
set -u
if grep -rnE --include='*.kt' --exclude-dir=build \
     --exclude='TooltipIconButton.kt' --exclude='TooltipFab.kt' \
     '\b(IconButton|FilledIconButton|FilledTonalIconButton|OutlinedIconButton|IconToggleButton|FloatingActionButton|SmallFloatingActionButton|LargeFloatingActionButton)\(' .; then
  echo 'Use TooltipIconButton / TooltipFab for icon-only controls (standard 7.7).'
  exit 1
fi
exit 0
```

`scripts/check_lint_suppressions.sh`:

```bash
#!/usr/bin/env bash
# Fail when translation or string-format lint checks are suppressed or baselined (standard 8.1).
set -u
IDS='MissingTranslation|ExtraTranslation|StringFormatMatches|StringFormatCount'
if grep -rnE --exclude-dir=build --include='*.xml' --include='*.kt' --include='*.kts' \
     "(ignore=\"[^\"]*($IDS)|SuppressLint\([^)]*($IDS)|disable \+?=.*($IDS))" .; then
  echo 'Translation lint checks must not be suppressed.'
  exit 1
fi
if find . -path '*/build' -prune -o -name 'lint-baseline*.xml' -print | xargs -r grep -nE "id=\"($IDS)\""; then
  echo 'Translation lint issues must not be in a lint baseline.'
  exit 1
fi
exit 0
```

The gates use GNU `grep` (`-P` for the Sanskrit gate). Run them on a Linux CI runner, or locally
in Git Bash / WSL.

---

## 20. Git And Repository Hygiene

### 20.1 Branching And Commits

- Protect the main branch for team repositories.
- Use short-lived branches.
- Prefer conventional commit prefixes such as `feat:`, `fix:`, `refactor:`, `test:`, `docs:`,
  and `build:`.
- Keep commits cohesive.

### 20.2 Never Commit

- Build output such as `build/`, APKs, AABs.
- Secrets, keys, keystores, and signing material. (Keystore location, `keystore.properties`
  naming, and the `.gitignore` rules: see `guideline.md §2`.)
- Local machine configuration files containing credentials or machine-specific paths.
- R8 mapping files.
- IDE project files (`.idea/`, `*.iml`) unless team-shared settings are intentional.

### 20.3 Usually Commit

- `gradle/libs.versions.toml` (version catalog)
- `gradle.properties`
- `settings.gradle.kts`
- `.gitignore`
- `.editorconfig`
- `proguard-rules.pro`
- `gradle/wrapper/gradle-wrapper.properties`
- `gradle/wrapper/gradle-wrapper.jar`

### 20.4 Standard .gitignore

```gitignore
# Build output
build/
*.apk
*.aab

# R8/ProGuard mapping files
mapping.txt

# Machine-local state
.gradle/
local.properties

# IDE files
.idea/
*.iml
.DS_Store

# Signing — never commit
keystore.properties
*.jks
*.keystore

# KSP generated
**/build/generated/ksp/
```

---

## 21. Documentation Standard

### 21.1 Required Documents For App Repositories

| Document | Purpose |
|----------|---------|
| `CLAUDE.md` | Mandatory project-root AI instructions following `CLAUDE_MD_GUIDELINE.md` (MUST) |
| `AGENTS.md` | Mandatory project-root AI agent instructions following `AGENTS_MD_GUIDELINE.md` (MUST) |
| `README.md` | Setup, run, test, and build instructions |
| `docs/GUIDELINES_MANIFEST.md` | Portable pointer manifest indexing shared Kotlin guidelines |
| `docs/architecture.md` | Module boundaries, initialization sequence, schema version, major decisions |
| `docs/release_process.md` | Required for shipped apps |
| `docs/*` files | Per-app documentation following `DOCS_FOLDER_GUIDELINE.md` |
| `plans/` | One plan per change — MUST follow the privacy rule in 21.1.1 |
| `change_log/` | One log per change — MUST follow the privacy rule in 21.1.1 |

#### 21.1.1 Privacy Rule For `plans/` And `change_log/`

Files in `plans/` and `change_log/` are committed and may become public on the internet. They MUST
use relative repository paths only and MUST NOT contain any **local system details** — OS user
name, computer/host name, home or drive-letter paths (`C:\Users\...`, `l:\...`, `file:///...`),
network share names, LAN or internal IP addresses, local server URLs with ports, device serial
numbers, personal email addresses — or any secret (API key, token, password, keystore passphrase,
credential, PII).

Write them as if a stranger will read them. Nothing should reveal the machine they were written on.

| Do not write | Write instead |
|---|---|
| `l:\Android\MyApp\app\src\main\...` | `app/src/main/...` |
| `C:\Users\<name>\.gradle\gradle.properties` | "the local Gradle home" |
| `file:///l:/Android/MyApp/plans/x.md` | `../plans/x.md` |
| `\\OFFICE-PC\share\build` | "the shared build folder" |
| `192.168.1.42:8080` | "the local dev server" |
| `someone@example.com` | "the release owner" |
| `keystorePassword=hunter2` | "the keystore password (stored outside the repo)" |

### 21.2 Recommended Documents

- `CHANGELOG.md` for user-facing release history.
- `docs/security.md` for sensitive-data apps.

### 21.3 README Must Include

- Prerequisites (Kotlin version, Java version, Android SDK versions, Android Studio version).
- Setup steps from a clean clone to a running app.
- How to run tests.
- Build commands for debug and release.
- How to add a new database migration.
- Signing setup (pointing to `keystore.properties` requirements without printing secrets).

---

## 22. AI Coding Assistant Instructions

When this standard is supplied to an AI coding assistant, the assistant MUST:

### 22.1 Before Writing Code

- Read and adhere strictly to the project's root `CLAUDE.md` / `AGENTS.md` instructions.
- Read the existing code before modifying it.
- Identify whether the repo is Tier 1 or Tier 2 and follow the existing structure.
- Identify the existing state-management pattern and follow it.
- Identify which applicability profile is in force for the repository.
- Check the current database schema version before writing any migration.
- Write a plan to `plans/` and obtain explicit user approval before modifying project files.

### 22.2 While Writing Code

- Respect the current project structure unless the task explicitly includes restructuring.
- Do not introduce a second state-management system without a documented reason.
- Do not add boilerplate comments or type annotations to unchanged code.
- Do not invent abstractions for one-time operations.
- Apply the security profile in force; never log secrets or weaken cryptographic behavior.
- Ensure all `plans/` and `change_log/` entries follow the privacy rule in 21.1.1: **relative
  repository paths only**, **no local system details**, and **no secrets**.
- Put all user-visible strings in `strings.xml` and read them through `stringResource()` or
  `context.getString()` — never a raw string literal in a Composable. Every app ships English,
  Malayalam and Sanskrit: add each new string to `values/`, `values-ml/` and `values-sa/` in the
  same change, with a real translation (section 8).
- Write Sanskrit strings under the 8.5 rules and glossary — never Hindi. Mark every Malayalam or
  Sanskrit string you write as needing native-reader review in the change log.
- Keep menu, button, label, tab and tooltip text within the 8.6 budget in all three languages.
- Return `@StringRes` ids or `UiText` from ViewModels and repositories — never resolved strings
  (8.7).
- Never remove the language build settings (locale filter, `generateLocaleConfig`,
  `bundle.language.enableSplit = false`) or the `AppCompatActivity` base class (8.1).
- Always use `val` over `var` where possible.
- Never use `LazyColumn` without a `key` parameter.
- Never call heavy synchronous work on the main thread; use `withContext(Dispatchers.IO)`.
- Always use `Log.d`/`Log.e` with a consistent `TAG`, never `println()`.
- Always add a `contentDescription` to interactive icons and images, and draw icon-only buttons
  with `TooltipIconButton` / `TooltipFab` so they have a localized tooltip (7.7).
- Never remove or reword the "Made with ❤️ from India" badge on the About screen
  (`guideline.md` §1.4).

### 22.3 After Writing Code

- Run `./gradlew testDebugUnitTest` after code changes that affect behavior.
- Run `./gradlew lint` before considering the task complete.
- Add or update tests when logic changes.
- Write a change log to `change_log/` referencing the plan, using relative paths only and
  excluding all local system details and sensitive information (section 21.1.1).
- Re-read the new plan and change log once before finishing, purely to check for leaked local
  system details.
- Verify that no secrets, local machine files, or build artifacts are staged.
- Verify that any new database schema change is accompanied by a migration.

---

## 23. Definition Of Done

A task is complete only when all applicable items are true.

### 23.1 Core Baseline

- Architecture boundaries were respected.
- New code follows the repository's chosen state-management pattern.
- Tests were added or updated for changed logic where appropriate.
- `./gradlew lint` is clean for the change.
- `./gradlew testDebugUnitTest` passes for behavior-affecting code changes.
- No secrets, build output, or local machine files were added to git.
- All `plans/` and `change_log/` files use relative repository paths only and contain zero local
  system details and zero sensitive data — safe to publish on the internet (section 21.1.1).
- Every user-visible string added or changed by this task comes from string resources and
  exists, translated, in `values/`, `values-ml/` and `values-sa/` (sections 8.2, 8.7).
- The feature works fully in English, Malayalam and Sanskrit, including notifications and
  background text (section 8.7).
- Sanskrit text passes `scripts/check_sanskrit.sh`; short labels are within budget (8.5, 8.6).
- Every icon-only control added by this task has a localized tooltip (section 7.7).
- The About screen still ends with the "Made with ❤️ from India" badge (`guideline.md` §1.4).
- Google Play build-time items (`release_process.md` §9A.1–§9A.4) still hold.
- KSP-generated files were not manually edited.

### 23.2 Production App Extension

- Release builds verified when the change touched build, config, signing, or release behavior.
- Required CI checks pass.
- User-facing documentation was updated if behavior changed.
- No new jank frames introduced on the primary user flow (verified in profiler if the change
  touched rendering, lists, or animations).
- App size budget was checked if a new dependency was added.

### 23.3 Sensitive Data Extension

- Sensitive data handling was reviewed against the security section.
- Logging was reviewed for protected data exposure.
- Backup, import, export, migration, or recovery paths were tested if touched.
- OWASP checklist items affected by the change were re-verified.

---

## 24. Practical Guidance

- A thin `MainActivity` scales better than a smart one.
- Mirrored tests reduce search time and ownership confusion.
- `utils/` is acceptable only when its scope stays clear and small.
- Product flavors solve real problems, but not every app needs them on day one.
- Security requirements should be attached to product risk, not copied blindly.
- CI should enforce the boring rules so review can focus on behavior and design.
- Stable keys in `LazyColumn` are free performance; use them always.
- Performance regressions are easiest to catch immediately after the change that caused them.
  Profile before merging, not six months later.
- Accessibility failures discovered late in a project are expensive to fix. Add content
  descriptions as you build each Composable, not in a post-hoc pass.
- Every unhandled exception that reaches a user is a trust failure. Design error boundaries first.

Treat this document as a baseline plus extensions. Tighten it for higher-risk apps, and relax
optional guidance only with a deliberate reason.
