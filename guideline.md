# Common Conventions Guideline for Kotlin Android Apps

This guideline makes all of my native Android Kotlin apps follow the **same conventions** for the
common things they all share: About-screen constants and the "Made with ❤️ from India" badge, the
three app languages (English, Malayalam, Sanskrit), the release keystore, and the overall source
package layout.

New apps MUST follow it from the start. Existing apps SHOULD be migrated toward it over time
(migration is a separate task, not part of adopting this document).

Conformance words: **MUST** = required, **SHOULD** = expected default, **MAY** = optional.

---

## 1. About-screen constants (single source of truth)

All values shown on the **About** screen (app name, version, author, AI used, IDE used,
etc.) MUST come from a **single source**, not from hard-coded string literals scattered in
Composables. Changing About content is then a config edit, not a code change.

Every app ships English, Malayalam and Sanskrit
([kotlin_project_engineering_standard.md §8](kotlin_project_engineering_standard.md)), so About
content MUST follow the language the user picked — labels, prose values, and the badge (§1.4).

This guideline supports two standard patterns. Pick one per app and use it consistently.

### 1.1 Pattern A: JSON asset file (recommended for new apps)

The About values live in a JSON asset file, loaded at runtime.

#### The three fixed paths

| Purpose                | Path                                                      | What lives here                                                        |
| ---------------------- | --------------------------------------------------------- | ---------------------------------------------------------------------- |
| Data (source of truth) | `app/src/main/assets/config/app_config.json`              | The actual About values, human-editable                                |
| Typed model            | `<package>/config/AppConfig.kt`                           | `AppConfig` data class + `LocalizedText`: `fromJson` + `fallback`      |
| Loader                 | `<package>/config/ConfigService.kt`                       | `ConfigService`: loads the asset, degrades to fallback, checks version |

Every app using Pattern A MUST use exactly these paths and these class names (`AppConfig`,
`LocalizedText`, `ConfigService`).

#### The JSON file

`app/src/main/assets/config/app_config.json`:

```json
{
  "appName": {
    "en": "<App name in English>",
    "ml": "<App name in Malayalam>",
    "sa": "<App name in Sanskrit>"
  },
  "description": {
    "en": "One-line description of what the app does.",
    "ml": "<Malayalam translation>",
    "sa": "<Sanskrit translation>"
  },
  "version": "1.0.0",
  "build": "1",
  "details": {
    "author": "Your Name",
    "email": "<Email>",
    "license": {
      "en": "All libraries used are open source.",
      "ml": "<Malayalam translation>",
      "sa": "<Sanskrit translation>"
    },
    "aiUsed": "<AI Name>",
    "ideUsed": "<IDE Name>"
  }
}
```

- `appName`, `description`, `version`, `build` are required top-level fields.
- **Localized values.** `appName`, `description` and every `details` value is either:
  - a **plain string** — the same text in every language. Use it only for values that do not
    change with language (a person's name, an email, a version, a tool name); or
  - a **language map** `{"en": …, "ml": …, "sa": …}` — MUST be used for any prose a user reads
    (description, licence text). All three keys MUST be present and non-empty; the parity test
    in [engineering standard §8.7](kotlin_project_engineering_standard.md) checks this.
- If the app name is a brand that stays the same in every language, `appName` MAY be a plain
  string. Record that choice in `docs/architecture.md` §16.
- `details` is a free map of **stable ids** → value. Ids are `camelCase` (`author`, `email`,
  `license`, `aiUsed`, `ideUsed`). The row label shown to the user is **not** the id — it comes
  from `strings.xml` (see §1.3). Add or remove rows as needed.
- Keep `version` and `build` in sync with `versionName` / `versionCode` in
  `build.gradle.kts`. `ConfigService.loadAndVerify` logs a non-fatal debug note if they drift
  (see the `ConfigService` rules below).

#### The `AppConfig` model — `<package>/config/AppConfig.kt`

Rules the model MUST follow:

- Immutable `data class` with `appName: LocalizedText`, `description: LocalizedText`,
  `version: String`, `build: String`, and `details: Map<String, LocalizedText>`.
- `LocalizedText` holds either one string for every language or one string per language, and
  `resolve(languageCode)` falls back to English when a language is missing.
- A `companion object` with a `val fallback` holding safe built-in values, so a missing or
  malformed config never crashes the app. (The fallback is English-only on purpose: it is a crash
  guard, not content. A shipped app always has a valid `app_config.json`.)
- A `fromJson(JSONObject)` factory function that reads field by field, accepts both the plain
  string and the language-map shape, and falls back per field on a missing value or wrong type
  (never throws). Old files with plain strings keep working.

Reference implementation:

```kotlin
import org.json.JSONObject

/** Text that is either the same in every language or given per language code. */
sealed interface LocalizedText {
    fun resolve(languageCode: String): String

    data class Plain(val value: String) : LocalizedText {
        override fun resolve(languageCode: String): String = value
    }

    data class ByLanguage(val values: Map<String, String>) : LocalizedText {
        override fun resolve(languageCode: String): String =
            values[languageCode]?.takeIf { it.isNotBlank() } ?: values["en"].orEmpty()
    }

    companion object {
        /** Accepts a JSON string or a {"en","ml","sa"} object; never throws. */
        fun from(raw: Any?, fallback: String = ""): LocalizedText = when (raw) {
            is String -> Plain(raw)
            is JSONObject -> {
                val map = buildMap {
                    for (key in raw.keys()) (raw.opt(key) as? String)?.let { put(key, it) }
                }
                if (map.isEmpty()) Plain(fallback) else ByLanguage(map)
            }
            else -> Plain(fallback)
        }
    }
}

/**
 * Typed values for the About screen, loaded from `assets/config/app_config.json`.
 * Changing About content is a config edit, not a code change.
 */
data class AppConfig(
    val appName: LocalizedText,
    val description: LocalizedText,
    val version: String,
    val build: String,
    val details: Map<String, LocalizedText> = emptyMap(),
) {
    companion object {
        /** Safe built-in value used when the config file is missing or malformed. */
        val fallback = AppConfig(
            appName = LocalizedText.Plain("My App"),
            description = LocalizedText.Plain("A Kotlin Android application."),
            version = "0.0.0",
            build = "0",
            details = emptyMap(),
        )

        fun fromJson(json: JSONObject): AppConfig {
            fun str(key: String, default: String): String =
                (json.opt(key) as? String) ?: default

            fun parseDetails(key: String): Map<String, LocalizedText> {
                val raw = json.optJSONObject(key) ?: return fallback.details
                val out = linkedMapOf<String, LocalizedText>()
                for (k in raw.keys()) {
                    val v = raw.opt(k)
                    if (v is String || v is JSONObject) out[k] = LocalizedText.from(v)
                }
                return out
            }

            return AppConfig(
                appName = LocalizedText.from(json.opt("appName"), "My App"),
                description = LocalizedText.from(json.opt("description"), "A Kotlin Android application."),
                version = str("version", fallback.version),
                build = str("build", fallback.build),
                details = parseDetails("details"),
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
- `loadAndVerify(context)` additionally compares the config's `version`/`build` with the
  installed package's `versionName` / `versionCode` and logs a non-fatal debug note on mismatch.

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

**Language rule for Pattern B.** A `BuildConfig` field holds one string, so it cannot change with
the user's language. Pattern B MUST therefore hold **only language-independent values** (author
name, IDE name, AI name, build date). Every row label and every piece of prose (app description,
licence text) MUST come from `strings.xml` in all three languages. An app that needs translated
About prose from a config file uses Pattern A.

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
About content, supports arbitrary key/value pairs and translated prose).

### 1.3 About screen — render `details` dynamically, in the chosen language

The About screen MUST be **data-driven**: it iterates over `AppConfig.details` and renders
one row per entry, in order. Whatever fields you add to `details` in `app_config.json` MUST
appear on the About screen automatically. The screen MUST NOT hard-code a fixed list of rows.

Rules:

- Read the current language from `LocalConfiguration.current.locales[0].language`, so the screen
  redraws when the user changes the language.
- Loop over `config.details`; render one row per entry, with the value from
  `value.resolve(language)`.
- **Row labels are localized.** The label for id `<id>` comes from the string
  `about_detail_<id in snake_case>` (`author` → `about_detail_author`, `aiUsed` →
  `about_detail_ai_used`), defined in all three `strings.xml` files. The wording comes from the
  "About screen" glossary in [engineering standard §8.5.4](kotlin_project_engineering_standard.md).
- A new `details` id MUST get its `about_detail_<id>` string in all three languages in the same
  change. The raw id is shown only as a last-resort fallback, so a row is never lost; the parity
  test (standard §8.7) fails when a `details` id has no label string.
- `about_detail_` strings are exempt from the short-label budget (standard §8.6) — an About row
  label may wrap to two lines.
- Skip any entry whose id or resolved value is blank after trimming.
- Optional nicety: if the id is `email`, make its row tappable to open `mailto:<value>`.
- The screen ends with the badge from §1.4.

Reference:

```kotlin
@Composable
fun AboutScreenContent(config: AppConfig, modifier: Modifier = Modifier) {
    val language = LocalConfiguration.current.locales[0].language
    Column(
        modifier = modifier
            .fillMaxSize()
            .verticalScroll(rememberScrollState())
            .padding(horizontal = 16.dp),
    ) {
        Text(config.appName.resolve(language), style = MaterialTheme.typography.headlineSmall)
        Text(config.description.resolve(language), style = MaterialTheme.typography.bodyMedium)
        AboutRow(
            label = stringResource(R.string.about_detail_version),
            value = "${config.version} (${config.build})",
        )
        config.details.forEach { (id, value) ->
            val text = value.resolve(language).trim()
            if (id.isBlank() || text.isEmpty()) return@forEach
            AboutRow(label = aboutDetailLabel(id), value = text)
        }
        MadeWithLove() // §1.4 — always last
    }
}

@Composable
private fun aboutDetailLabel(id: String): String = when (id) {
    "author" -> stringResource(R.string.about_detail_author)
    "email" -> stringResource(R.string.about_detail_email)
    "license" -> stringResource(R.string.about_detail_license)
    "aiUsed" -> stringResource(R.string.about_detail_ai_used)
    "ideUsed" -> stringResource(R.string.about_detail_ide_used)
    else -> id // last-resort fallback; the parity test flags a missing label string
}
```

> **Note on other constants.** This JSON/BuildConfig pattern is only for **About-screen**
> metadata. Technical constants (database names, preference keys, thresholds, channel IDs)
> do NOT go in the JSON. Keep those in a plain Kotlin file such as
> `<package>/config/AppConstants.kt` (object with `const val` values only, no logic).

### 1.4 About screen — the "Made with ❤️ from India" badge (fixed, every app)

Every app's About screen MUST end with the same signature badge, rendered **below all other
About content**, horizontally centered, in the language the user has selected:

```text
English    Made with ❤️ from India
Malayalam  സ്നേഹത്തോടെ ❤️ ഇന്ത്യയിൽ നിന്ന്
Sanskrit   सस्नेहं निर्मितम् ❤️ भारततः
```

Rules:

- **Mandatory and identical in every app.** It is not configurable, does not come from
  `app_config.json` or `about.properties`, and MUST NOT be removed, reworded, or replaced per app.
- **It is the last element** on the About screen, after the `details` rows, with at least 24 dp of
  space above it and navigation-bar padding below.
- **The words** use `MaterialTheme.colorScheme.onSurfaceVariant` at `MaterialTheme.typography.bodySmall`.
- **The heart is a red icon, not an emoji character.** Many Android builds draw the ❤ character
  from the colour emoji font, which ignores text colour. The badge therefore draws
  `Icons.Filled.Favorite` inline, one text-line high, tinted `Color(0xFFE53935)`. It looks the same
  on every device.
- **The words are localized, the heart is not.** The strings contain a `%1$s` marker where the
  heart goes, so the same Composable places the heart correctly in all three languages. Do not move
  the marker to the start or end to make the code simpler.
- **It is screen-reader friendly**: TalkBack reads `about_made_with_love_a11y` ("Made with love from
  India"), not an icon name.
- The badge is **not** a link and has no tap action.

Strings (all three files are required; the wording is **fixed** — copy it verbatim into every
app):

```xml
<!-- res/values/strings.xml -->
<!-- About-screen signature badge. %1$s is where the red heart icon is drawn. Fixed wording. -->
<string name="about_made_with_love">Made with %1$s from India</string>
<!-- Screen-reader text for the About badge. Fixed wording. -->
<string name="about_made_with_love_a11y">Made with love from India</string>
```

```xml
<!-- res/values-ml/strings.xml -->
<string name="about_made_with_love">സ്നേഹത്തോടെ %1$s ഇന്ത്യയിൽ നിന്ന്</string>
<string name="about_made_with_love_a11y">സ്നേഹത്തോടെ ഇന്ത്യയിൽ നിന്ന്</string>
```

```xml
<!-- res/values-sa/strings.xml -->
<string name="about_made_with_love">सस्नेहं निर्मितम् %1$s भारततः</string>
<string name="about_made_with_love_a11y">सस्नेहं निर्मितम् भारततः</string>
```

Reference Composable — `ui/components/MadeWithLove.kt`:

```kotlin
private const val HEART_ID = "heart"
private const val SENTINEL = "\u0000" // never appears in translated text
private val HeartRed = Color(0xFFE53935)

/**
 * The fixed "Made with ❤️ from India" badge shown at the bottom of every About
 * screen. The words are localized; the heart is a red icon in every language.
 */
@Composable
fun MadeWithLove(modifier: Modifier = Modifier) {
    val parts = stringResource(R.string.about_made_with_love, SENTINEL).split(SENTINEL)
    val a11y = stringResource(R.string.about_made_with_love_a11y)
    val text = buildAnnotatedString {
        append(parts.first())
        appendInlineContent(HEART_ID, "♥")
        if (parts.size > 1) append(parts[1])
    }
    val inlineContent = mapOf(
        HEART_ID to InlineTextContent(
            Placeholder(width = 1.em, height = 1.em, PlaceholderVerticalAlign.TextCenter),
        ) {
            Icon(
                imageVector = Icons.Filled.Favorite,
                contentDescription = null,
                tint = HeartRed,
                modifier = Modifier.fillMaxSize(),
            )
        },
    )
    Text(
        text = text,
        inlineContent = inlineContent,
        style = MaterialTheme.typography.bodySmall,
        color = MaterialTheme.colorScheme.onSurfaceVariant,
        textAlign = TextAlign.Center,
        modifier = modifier
            .fillMaxWidth()
            .padding(top = 24.dp)
            .navigationBarsPadding()
            .clearAndSetSemantics { contentDescription = a11y },
    )
}
```

> `Icons.Filled.Favorite` lives in `androidx.compose.material:material-icons-core`. Newer
> Material 3 versions no longer pull that library in for you — add it to the version catalog, or
> use an equivalent heart vector drawable from `res/drawable/`.

**Checks:**

- Lint `StringFormatMatches` and `StringFormatCount` are errors (see
  [kotlin_build_configuration_guide.md](kotlin_build_configuration_guide.md) "Languages"), so a
  translation that drops or duplicates `%1$s` fails the build.
- A JVM test asserts `about_made_with_love` contains `%1$s` exactly once in all three files.
- The About screen screenshot test (standard §18.7) runs in `en`, `ml` and `sa`, so the heart's
  position is reviewed in every language.

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
MAY add more packages — but the **About pattern (§1), the badge (§1.4), the three languages, and
the keystore rules (§2) are fixed and MUST NOT change**.

```
app/src/main/java/<package>/
  config/          # AppConfig + LocalizedText + ConfigService (About). REQUIRED, fixed path.
                   # Also AppConstants for technical constants.
  data/            # data models, Room database, DAOs, repository
    model/         # data classes / entities
    db/            # Room database, DAOs, migrations
    repository/    # data access abstraction
  l10n/            # LanguageRepository, UiText, formattingLocale, localizedContext. REQUIRED.
  ui/              # Composable screens, components, theme
    screens/       # full-page Composable screens (incl. the About and Settings screens)
    components/    # reusable UI Composables (incl. MadeWithLove, TooltipIconButton)
    theme/         # Color, Typography, Theme, Shape definitions
  services/        # platform + business services (optional)
  utils/           # small helpers, extensions (optional)
  receiver/        # BroadcastReceivers (optional, if applicable)
  widget/          # app widgets (optional, if applicable)
  MainActivity.kt  # single Activity entry point — extends AppCompatActivity
  MainApplication.kt  # Application subclass (optional, if needed)

app/src/main/res/
  values/strings.xml      # English — default. REQUIRED.
  values-ml/strings.xml   # Malayalam. REQUIRED.
  values-sa/strings.xml   # Sanskrit (Devanagari). REQUIRED.
  resources.properties    # unqualifiedResLocale=en. REQUIRED.
  values/themes.xml       # XML theme parent: Theme.AppCompat.DayNight.NoActionBar

scripts/                  # project root: check_sanskrit.sh, check_icon_buttons.sh,
                          # check_lint_suppressions.sh (standard §19.4). REQUIRED.
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
- **Every app ships three languages: English, Malayalam and Sanskrit.** No app is
  single-language. `res/values/strings.xml` (English, default), `res/values-ml/strings.xml` and
  `res/values-sa/strings.xml` MUST exist in every module that has user-visible text. All
  user-visible text MUST come from string resources (`stringResource()` in Compose or
  `context.getString()` in code), never a raw string literal in a Composable. Full rules:
  [kotlin_project_engineering_standard.md §8](kotlin_project_engineering_standard.md).
- **String parity is mandatory**: every string exists in all three files with a real translation.
  A feature is not done until it works in English, Malayalam and Sanskrit (standard §8.7).
- **Sanskrit means Sanskrit**, not Hindi written in Devanagari. `values-sa/strings.xml` MUST follow
  the Sanskrit quality rules and glossary in standard §8.5, and `scripts/check_sanskrit.sh` MUST
  pass.
- **The app language is user-selectable.** The default is the system language (English when the
  system language is none of the three). A language picker in Settings MUST let the user override
  it through `AppCompatDelegate.setApplicationLocales`; the choice persists and applies at once
  (standard §8.4).
- **Menu, label, button, tab and tooltip text MUST be short** in all three languages (standard
  §8.6). Only descriptive text (help, empty-state explanations, About description, error detail)
  may be long.
- **Every icon-only control MUST have a localized tooltip**, through the shared
  `TooltipIconButton` / FAB wrapper (standard §7.7).
- `config/` MUST exist and hold `AppConfig` + `LocalizedText` + `ConfigService` (or the
  BuildConfig equivalent) exactly as in §1.
- The About screen lives under `ui/screens/` (e.g. `ui/screens/AboutScreen.kt`), reads its
  values from `ConfigService` / `AppConfig` or `BuildConfig` — it MUST NOT hard-code About text —
  and MUST end with the fixed "Made with ❤️ from India" badge (§1.4).
- **Every app is always ready for Google Play.** The build-time items of
  [release_process.md §9A](release_process.md) (§9A.1–§9A.4) apply from day one; the console items
  (§9A.5–§9A.8) MUST be done before the first upload.
- Pick **one** architecture pattern per app (MVVM with ViewModel is the default) and use it
  consistently.
- Larger apps MAY introduce a feature-first structure (e.g. `features/<feature>/data/`,
  `features/<feature>/ui/`). When they do, the `config/` About pattern, the three-language rules
  (every feature module has all three `strings.xml` files), and the keystore rules still apply
  unchanged.

---

## 4. Quick checklist for a new (or migrated) app

- [ ] Root `CLAUDE.md` exists at project root and follows [CLAUDE_MD_GUIDELINE.md](CLAUDE_MD_GUIDELINE.md) (**MUST**).
- [ ] Root `AGENTS.md` exists at project root and follows [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md) (**MUST**).
- [ ] `plans/` and `change_log/` files use relative repository paths only and contain zero local
      system details and zero sensitive data — safe to publish on the internet (**MUST**).
- [ ] `values/`, `values-ml/` and `values-sa/` `strings.xml` exist in every module with UI text,
      and all user-visible text uses string resources (**MUST**).
- [ ] String parity holds — every string, plural and string-array exists and is translated in all
      three files; lint `MissingTranslation` is an error and never suppressed (**MUST**, standard §8.7).
- [ ] Build setup done: locale filter `en`/`ml`/`sa`, `generateLocaleConfig` + `resources.properties`,
      `bundle.language.enableSplit = false`, AppCompat + `Theme.AppCompat` parent (**MUST**, standard §8.1).
- [ ] Settings has a language picker (System default / English / മലയാളം / संस्कृतम्) that persists
      and applies without a restart (**MUST**, standard §8.4).
- [ ] Sanskrit strings pass `scripts/check_sanskrit.sh` and use the glossary terms (**MUST**, standard §8.5).
- [ ] Menu / label / button / tab / tooltip text is within the length budget in all three languages
      (**MUST**, standard §8.6).
- [ ] Every icon-only control has a localized tooltip via `TooltipIconButton`; `scripts/check_icon_buttons.sh`
      passes (**MUST**, standard §7.7).
- [ ] No hard-coded user-visible strings — all screen text comes from string resources (**MUST**).
- [ ] About-screen values come from a single source (Pattern A or Pattern B from §1).
- [ ] `config/AppConfig.kt` exists with `LocalizedText`, `fromJson` + `fallback` (Pattern A) OR
      `BuildConfig` fields hold only language-independent values (Pattern B).
- [ ] Every prose value in `app_config.json` uses the `{"en","ml","sa"}` language map (§1.1).
- [ ] About screen reads from config, not hard-coded strings.
- [ ] About screen renders `details` dynamically (loops the map, no hard-coded row list), with
      localized `about_detail_<id>` labels and language-resolved values (§1.3).
- [ ] About screen ends with the fixed "Made with ❤️ from India" badge — red heart icon, centered,
      localized words (**MUST**, §1.4).
- [ ] Release keystore is at `<name>.jks` at project root; `keystore.properties` points to it.
- [ ] `keystore.properties`, `*.jks`, `*.keystore` are git-ignored.
- [ ] Source package follows the baseline layout in §3 (subset is fine for small apps).
- [ ] Google Play build-time items pass from day one; console items done before the first upload
      ([release_process.md §9A](release_process.md)) (**MUST**).
