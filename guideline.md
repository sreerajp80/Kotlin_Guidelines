# Common Conventions Guideline for Kotlin Android Apps

This guideline makes all of my native Android Kotlin apps follow the **same conventions** for the
common things they all share: About-screen constants, the release keystore, and the overall
source package layout.

New apps MUST follow it from the start. Existing apps SHOULD be migrated toward it over time
(migration is a separate task, not part of adopting this document).

Conformance words: **MUST** = required, **SHOULD** = expected default, **MAY** = optional.

---

## 1. About-screen constants (single source of truth)

All values shown on the **About** screen (app name, version, author, AI used, IDE used,
etc.) MUST come from a **single source**, not from hard-coded string literals scattered in
Composables. Changing About content is then a config edit, not a code change.

This guideline supports two standard patterns. Pick one per app and use it consistently.

### 1.1 Pattern A: JSON asset file (recommended for new apps)

The About values live in a JSON asset file, loaded at runtime.

#### The three fixed paths

| Purpose                | Path                                                      | What lives here                                                        |
| ---------------------- | --------------------------------------------------------- | ---------------------------------------------------------------------- |
| Data (source of truth) | `app/src/main/assets/config/app_config.json`              | The actual About values, human-editable                                |
| Typed model            | `<package>/config/AppConfig.kt`                           | `AppConfig` data class: `fromJson` + `fallback`                        |
| Loader                 | `<package>/config/ConfigService.kt`                       | `ConfigService`: loads the asset, degrades to fallback, checks version |

Every app using Pattern A MUST use exactly these paths and these class names (`AppConfig`,
`ConfigService`).

#### The JSON file

`app/src/main/assets/config/app_config.json`:

```json
{
  "appName": "My App Name",
  "description": "One-line description of what the app does.",
  "version": "1.0.0",
  "build": "1",
  "details": {
    "Author": "Your Name",
    "Email": "<Email>",
    "License": "All libraries used are open source.",
    "AI used": "<AI Name>",
    "IDE used": "<IDE Name>"
  }
}
```

- `appName`, `description`, `version`, `build` are required top-level string fields.
- `details` is a free `string → string` map. Add or remove rows as needed; the About
  screen renders each key/value as a labelled row.
- Keep `version` and `build` in sync with `versionName` / `versionCode` in
  `build.gradle.kts`. `ConfigService` will log a non-fatal debug note if they drift (see §1.5).

#### The `AppConfig` model — `<package>/config/AppConfig.kt`

Rules the model MUST follow:

- Immutable `data class` with `appName`, `description`, `version`, `build`, and
  `Map<String, String> details`.
- A `companion object` with a `val fallback` holding safe built-in values, so a missing or
  malformed config never crashes the app.
- A `fromJson(JSONObject)` factory function that reads field by field and falls back per field
  on a missing value or wrong type (never throws).

Reference implementation:

```kotlin
import org.json.JSONObject

/**
 * Typed values for the About screen, loaded from `assets/config/app_config.json`.
 * Changing About content is a config edit, not a code change.
 */
data class AppConfig(
    val appName: String,
    val description: String,
    val version: String,
    val build: String,
    val details: Map<String, String> = emptyMap(),
) {
    companion object {
        /** Safe built-in value used when the config file is missing or malformed. */
        val fallback = AppConfig(
            appName = "My App",
            description = "A Kotlin Android application.",
            version = "0.0.0",
            build = "0",
            details = mapOf("License" to "All libraries used are open source."),
        )

        fun fromJson(json: JSONObject): AppConfig {
            fun str(key: String, default: String): String =
                json.optString(key, default)

            fun parseStringMap(key: String): Map<String, String> {
                val raw = json.optJSONObject(key) ?: return fallback.details
                val out = mutableMapOf<String, String>()
                for (k in raw.keys()) {
                    val v = raw.opt(k)
                    if (v is String) out[k] = v
                }
                return out
            }

            return AppConfig(
                appName = str("appName", fallback.appName),
                description = str("description", fallback.description),
                version = str("version", fallback.version),
                build = str("build", fallback.build),
                details = parseStringMap("details"),
            )
        }
    }
}
```

#### The `ConfigService` loader — `<package>/config/ConfigService.kt`

Rules the loader MUST follow:

- Constant `ASSET_PATH = "config/app_config.json"`.
- `load(context)` reads the asset, decodes JSON, and returns `AppConfig.fallback` on **any**
  error (missing asset, bad JSON, wrong shape).
- `loadAndVerify(context)` additionally compares the config's `version`/`build` with
  `BuildConfig.VERSION_NAME` / `BuildConfig.VERSION_CODE` and logs a non-fatal debug note
  on mismatch.

Reference implementation:

```kotlin
import android.content.Context
import android.content.pm.PackageManager
import android.os.Build
import android.util.Log
import androidx.core.content.pm.PackageInfoCompat
import org.json.JSONObject

class ConfigService {
    companion object {
        private const val ASSET_PATH = "config/app_config.json"
        private const val TAG = "ConfigService"

        fun load(context: Context): AppConfig = try {
            val text = context.assets.open(ASSET_PATH).bufferedReader().use { it.readText() }
            AppConfig.fromJson(JSONObject(text))
        } catch (e: Exception) {
            AppConfig.fallback
        }

        fun loadAndVerify(context: Context): AppConfig {
            val config = load(context)
            try {
                val info = if (Build.VERSION.SDK_INT >= Build.VERSION_CODES.TIRAMISU) {
                    context.packageManager.getPackageInfo(
                        context.packageName,
                        PackageManager.PackageInfoFlags.of(0L),
                    )
                } else {
                    @Suppress("DEPRECATION")
                    context.packageManager.getPackageInfo(context.packageName, 0)
                }
                val versionCode = PackageInfoCompat.getLongVersionCode(info).toString()
                val mismatch = info.versionName != config.version ||
                    versionCode != config.build
                if (mismatch) {
                    Log.d(
                        TAG,
                        "version/build in app_config.json " +
                            "(${config.version}+${config.build}) does not match the build " +
                            "(${info.versionName}+$versionCode).",
                    )
                }
            } catch (e: Exception) {
                // Package info unavailable — ignore.
            }
            return config
        }
    }
}
```

### 1.2 Pattern B: BuildConfig fields via `about.properties`

The About values live in a properties file at `app/about.properties`, read by Gradle at build
time, and injected as `BuildConfig` fields.

#### The properties file

`app/about.properties`:

```properties
author=Your Name
ide=Android Studio
aiVersion=Claude Opus 4
```

#### Gradle injection

In `app/build.gradle.kts`:

```kotlin
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale
import java.util.Properties

val aboutPropsFile = file("about.properties")
val aboutProps = Properties().apply {
    if (aboutPropsFile.exists()) aboutPropsFile.inputStream().use { load(it) }
}
fun aboutValue(key: String, default: String): String =
    "\"${aboutProps.getProperty(key, default).replace("\"", "\\\"")}\""
val buildDate: String = SimpleDateFormat("yyyy-MM-dd HH:mm", Locale.US).format(Date())

android {
    buildFeatures {
        buildConfig = true
    }

    defaultConfig {
        buildConfigField("String", "AUTHOR", aboutValue("author", "Unknown"))
        buildConfigField("String", "IDE", aboutValue("ide", "Android Studio"))
        buildConfigField("String", "AI_VERSION", aboutValue("aiVersion", "Unknown"))
        buildConfigField("String", "BUILD_DATE", "\"$buildDate\"")
    }
}
```

#### Accessing in code

```kotlin
val author = BuildConfig.AUTHOR
val buildDate = BuildConfig.BUILD_DATE
```

Pattern B is simpler for small apps. Pattern A is more flexible (no rebuild needed to change
About content, supports arbitrary key/value pairs).

### 1.3 About screen — render `details` dynamically (Pattern A)

The About screen MUST be **data-driven**: it iterates over `AppConfig.details` and renders
one row per entry, in order. Whatever key/value fields you add to `details` in
`app_config.json` MUST appear on the About screen automatically — adding or removing a key
in the JSON is the **only** change needed. The screen MUST NOT hard-code field names like
`Author` or `Email`.

Rules:

- Loop over `config.details.entries`; render one row per entry.
- Skip any entry whose key or value is blank after trimming.
- Optional nicety: if a key equals `email` (case-insensitive), make its row tappable to
  open `mailto:<value>`.

> **Note on other constants.** This JSON/BuildConfig pattern is only for **About-screen**
> metadata. Technical constants (database names, preference keys, thresholds, channel IDs)
> do NOT go in the JSON. Keep those in a plain Kotlin file such as
> `<package>/config/AppConstants.kt` (object with `const val` values only, no logic).

---

## 2. Release keystore + `keystore.properties`

Every app that ships a signed release MUST follow this.

### 2.1 Locations and names

| Item               | Location     | Name                                                                    |
| ------------------ | ------------ | ----------------------------------------------------------------------- |
| Keystore           | project root | `<name>.jks` — **name is the user's choice per app**                    |
| Signing properties | project root | `keystore.properties` — **recommended fixed name**                      |

- The keystore MUST live at the project root. Its filename is free per app (e.g.
  `keystore.jks`, `release-keystore.jks`, `vfkeystore.jks`) — whatever you choose,
  `keystore.properties` points to it.
- The properties file SHOULD be named `keystore.properties`. Existing apps that use
  `local.properties` for keystore secrets MAY keep that convention but MUST NOT store SDK path
  and keystore secrets in the same properties file in new projects.

### 2.2 `keystore.properties` format

`keystore.properties`:

```properties
storePassword=<store password>
keyPassword=<key password>
keyAlias=<key alias>
storeFile=<name>.jks
```

`storeFile` is the keystore filename you chose in §2.1 (relative to the project root).

### 2.3 Never commit secrets

Both files MUST be git-ignored. Add to the app's `.gitignore`:

```gitignore
# Signing — never commit
keystore.properties
*.jks
*.keystore
```

Keep a secure, offline backup of each app's keystore. Losing it means you can no longer
publish updates under the same signature.

---

## 3. Standard source package structure

Use this baseline layout so every app feels the same. Small apps use a subset; larger apps
MAY add more packages — but the **About pattern (§1) and keystore rules (§2) are fixed and
MUST NOT change**.

```
app/src/main/java/<package>/
  config/          # AppConfig + ConfigService (About). REQUIRED, fixed path.
                   # Also AppConstants for technical constants.
  data/            # data models, Room database, DAOs, repository
    model/         # data classes / entities
    db/            # Room database, DAOs, migrations
    repository/    # data access abstraction
  ui/              # Composable screens, components, theme
    screens/       # full-page Composable screens (incl. the About screen)
    components/    # reusable UI Composables
    theme/         # Color, Typography, Theme, Shape definitions
  services/        # platform + business services (optional)
  utils/           # small helpers, extensions (optional)
  receiver/        # BroadcastReceivers (optional, if applicable)
  widget/          # app widgets (optional, if applicable)
  MainActivity.kt  # single Activity entry point
  MainApplication.kt  # Application subclass (optional, if needed)
```

Rules:

- Root `CLAUDE.md` MUST exist at the project root and follow [CLAUDE_MD_GUIDELINE.md](CLAUDE_MD_GUIDELINE.md).
- Root `AGENTS.md` MUST exist at the project root and follow [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md).
- `plans/` and `change_log/` files MUST use relative repository paths only and MUST NOT contain
  **local system details** (OS user name, computer/host name, home or drive-letter paths, network
  share names, LAN/internal IPs, local server URLs with ports, device serial numbers, personal
  email addresses) or any secret (keys, tokens, passwords, keystore passphrases, credentials, PII).
  These files are committed and may become public — write them as if a stranger will read them.
  Full rule and bad → good examples:
  [kotlin_project_engineering_standard.md §21.1.1](kotlin_project_engineering_standard.md).
- `res/values/strings.xml` MUST exist — for **every** app, even one that ships a single language.
  All user-visible text MUST come from string resources (`stringResource()` in Compose or
  `context.getString()` in code), never a raw string literal in a Composable. `strings.xml` is
  Android's native localization mechanism: create it from day one so adding a language later is
  only a new `values-xx/strings.xml` folder.
- `config/` MUST exist and hold `AppConfig` + `ConfigService` (or the BuildConfig equivalent)
  exactly as in §1.
- The About screen lives under `ui/screens/` (e.g. `ui/screens/AboutScreen.kt`) and reads its
  values from `ConfigService` / `AppConfig` or `BuildConfig` — it MUST NOT hard-code About text.
- Pick **one** architecture pattern per app (MVVM with ViewModel is the default) and use it
  consistently.
- Larger apps MAY introduce a feature-first structure (e.g. `features/<feature>/data/`,
  `features/<feature>/ui/`). When they do, the `config/` About pattern and the keystore
  rules still apply unchanged.

---

## 4. Quick checklist for a new (or migrated) app

- [ ] Root `CLAUDE.md` exists at project root and follows [CLAUDE_MD_GUIDELINE.md](CLAUDE_MD_GUIDELINE.md) (**MUST**).
- [ ] Root `AGENTS.md` exists at project root and follows [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md) (**MUST**).
- [ ] `plans/` and `change_log/` files use relative repository paths only and contain zero local
      system details and zero sensitive data — safe to publish on the internet (**MUST**).
- [ ] `res/values/strings.xml` exists and all user-visible text uses string resources (**MUST**,
      even for a single-language app).
- [ ] No hard-coded user-visible strings — all screen text comes from string resources (**MUST**).
- [ ] About-screen values come from a single source (Pattern A or Pattern B from §1).
- [ ] `config/AppConfig.kt` exists with `fromJson` + `fallback` (Pattern A) OR `BuildConfig`
      fields are defined in `build.gradle.kts` (Pattern B).
- [ ] About screen reads from config, not hard-coded strings.
- [ ] About screen renders `details` dynamically (Pattern A: loops the map, no hard-coded field
      names).
- [ ] Release keystore is at `<name>.jks` at project root; `keystore.properties` points to it.
- [ ] `keystore.properties`, `*.jks`, `*.keystore` are git-ignored.
- [ ] Source package follows the baseline layout in §3 (subset is fine for small apps).
