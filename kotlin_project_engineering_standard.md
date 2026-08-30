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
- Decorative images MUST use `contentDescription = null` to exclude them from TalkBack.
- Use `Modifier.semantics { stateDescription = ... }` for custom stateful components.

```kotlin
Icon(
    imageVector = Icons.Default.Delete,
    contentDescription = stringResource(R.string.cd_delete_todo),
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

---

## 8. Localization And Internationalization

This section is `Core Baseline`: it applies to every user-facing app repository, including apps
that ship only one language. Single-language apps MUST complete the minimum setup in 8.1 **and**
the string externalization in 8.2.

### 8.1 Minimum Setup (All Apps)

Every Android app MUST have `res/values/strings.xml` with all user-visible strings externalized.
This is mandatory even for a single-language app.

### 8.2 String Externalization (Mandatory, All Apps)

Every app MUST externalize its user-visible strings into `strings.xml`, **even if it supports only
one language**. This is not optional and does not wait for a translation request.

`strings.xml` is Android's native localization mechanism. Create it from day one so adding a
language later is only a new `values-xx/strings.xml` — not a rewrite of every screen.

Required for every app:

- `res/values/strings.xml` MUST exist.
- Every user-visible string MUST be defined in `strings.xml` and read through
  `stringResource(R.string.key)` in Compose or `context.getString(R.string.key)` in Kotlin.
  A raw string literal in a Composable is not allowed.

**Narrow exceptions** — these MAY stay as plain Kotlin literals, because a user never reads them:

| Allowed as a literal | Example |
|---|---|
| Log and debug messages | `Log.d(TAG, "cache miss for $id")` |
| Exception messages not shown in the UI | `throw IllegalStateException("db not initialized")` |
| Technical identifiers | Asset paths, route names, map/JSON keys, test tags |
| Developer-only screens | A debug menu that never ships to users |

Anything a real user reads — screen titles, buttons, labels, hints, error text shown on screen,
empty states, snackbars, dialogs, notification text — goes in `strings.xml`.

### 8.3 RTL Layout Support

- Never use `left` and `right` for padding, alignment, or positioning of UI elements. Use `start`
  and `end` equivalents in Compose layouts.
- Test RTL by switching the device locale to Arabic or Hebrew.

### 8.4 Locale-Sensitive Formatting

Use `java.text.DateFormat`, `java.time.format.DateTimeFormatter`, or
`android.text.format.DateUtils` for all locale-sensitive formatting. Never use `toString()` on
dates, numbers, or currencies in user-visible strings.

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
- Put all user-visible strings in `res/values/strings.xml` and read them through
  `stringResource()` or `context.getString()` — never a raw string literal in a Composable, even
  in a single-language app.
- Always use `val` over `var` where possible.
- Never use `LazyColumn` without a `key` parameter.
- Never call heavy synchronous work on the main thread; use `withContext(Dispatchers.IO)`.
- Always use `Log.d`/`Log.e` with a consistent `TAG`, never `println()`.
- Always add a `contentDescription` to interactive icons and images.

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
- `res/values/strings.xml` exists, and every user-visible string added or changed by this task
  comes from string resources (section 8.2).
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
