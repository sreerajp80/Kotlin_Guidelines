# Kotlin Build Configuration Guide

Use this as a reusable reference for Kotlin Android projects that define build types, product
flavors, signing configs, and Gradle conventions.

This guide documents the standard approach for each concern. Teams with different CI pipelines,
signing strategies, or flavor matrices should treat the examples here as a starting point and
document their deviations in `docs/architecture.md`.

> **Reflects AGP 8.x, Kotlin 2.1, Compose BOM 2025, and Gradle 8.14+ (mid 2026).** Where a
> constraint comes from a specific version, the version is named so you can verify against
> your toolchain.

---

## Toolchain Prerequisites

Before any build runs, the host toolchain MUST satisfy these minimums:

| Concern | Minimum | Notes |
|---------|---------|-------|
| Kotlin | 2.1.x | Pin to specific minor in version catalog |
| AGP (Android Gradle Plugin) | 8.x — **NOT 9.x** | AGP 9 audit is paused |
| Gradle | 8.14+ | Required by AGP 8.x |
| JDK | **Java 17** | Gradle 8.14+ and AGP 8.x require JDK 17 |
| compileSdk | 36 (API 36) | Or the latest stable Android SDK |
| targetSdk | 36 (API 36) | Must meet Play's current target API level policy — re-check before every release (`release_process.md` §9A.2) |
| minSdk | 24 (Android 7.0) | Or higher based on app requirements |
| Android Studio | Latest stable | For build toolchain compatibility |

---

## Build Types

### Debug

The `debug` build type is the default for local development. It has `isDebuggable = true`
automatically and uses the automatic debug keystore.

### Release

The `release` build type is for production artifacts. It SHOULD enable R8 code shrinking:

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro",
        )
        signingConfig = signingConfigs.getByName("release")
    }
}
```

---

## Signing Configuration

See `guideline.md §2` for the keystore source-of-truth.

### Standard Pattern

```kotlin
import java.util.Properties

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
        debug {
            // Optional: use the same keystore for debug to match signature
            signingConfig = signingConfigs.getByName("release")
        }
    }
}
```

### Alternative: `local.properties` Pattern

Some projects read signing secrets from `local.properties` (which is already git-ignored by
Android Studio). This is acceptable but has a tradeoff: the SDK path and signing secrets share
a file.

```kotlin
val localProps = Properties().apply {
    val f = rootProject.file("local.properties")
    if (f.exists()) f.inputStream().use { load(it) }
}
fun secret(key: String): String? = System.getenv(key) ?: localProps.getProperty(key)

android {
    signingConfigs {
        create("appKeystore") {
            val keystoreRelativePath = secret("KEYSTORE_PATH") ?: "keystore.jks"
            storeFile = rootProject.file(keystoreRelativePath)
            storePassword = secret("STORE_PASSWORD")
            keyAlias = secret("KEY_ALIAS") ?: "app-key"
            keyPassword = secret("KEY_PASSWORD")
        }
    }
}
```

### CI Signing

For CI pipelines, signing secrets MUST come from environment variables or a CI secret store,
not from committed files:

```kotlin
fun secret(key: String) = System.getenv(key) ?: keystoreProps.getProperty(key)
```

---

## Version Catalog

All dependencies MUST be declared in `gradle/libs.versions.toml`:

```toml
[versions]
agp = "8.9.0"
kotlin = "2.1.21"
composeBom = "2025.06.01"
room = "2.7.1"
ksp = "2.1.21-1.0.35"
secrets = "2.0.1"

[libraries]
androidx-compose-bom = { group = "androidx.compose", name = "compose-bom", version.ref = "composeBom" }
androidx-compose-material3 = { group = "androidx.compose.material3", name = "material3" }
androidx-room-runtime = { group = "androidx.room", name = "room-runtime", version.ref = "room" }
androidx-room-ktx = { group = "androidx.room", name = "room-ktx", version.ref = "room" }
androidx-room-compiler = { group = "androidx.room", name = "room-compiler", version.ref = "room" }

[plugins]
android-application = { id = "com.android.application", version.ref = "agp" }
kotlin-android = { id = "org.jetbrains.kotlin.android", version.ref = "kotlin" }
kotlin-compose = { id = "org.jetbrains.kotlin.plugin.compose", version.ref = "kotlin" }
google-devtools-ksp = { id = "com.google.devtools.ksp", version.ref = "ksp" }
secrets = { id = "com.google.android.libraries.mapsplatform.secrets-gradle-plugin", version.ref = "secrets" }
```

### Version Catalog Rules

- All production dependencies MUST be declared in the version catalog.
- Version references (`version.ref`) MUST be used for libraries that share versions (e.g. Room
  runtime, KTX, and compiler all share `room`).
- Test-only and debug-only dependencies SHOULD also be declared in the catalog for consistency.
- The version catalog file MUST be committed to source control.

---

## Compose Compiler Configuration

The Compose compiler is configured via the `kotlin-compose` plugin:

```kotlin
plugins {
    alias(libs.plugins.kotlin.compose)
}
```

For advanced configuration (stability configuration, metrics reporting):

```kotlin
composeCompiler {
    // Generate Compose compiler metrics for performance analysis
    // metricsDestination = layout.buildDirectory.dir("compose-metrics")
    // reportsDestination = layout.buildDirectory.dir("compose-reports")
}
```

---

## Languages (English, Malayalam, Sanskrit)

Every app ships `en`, `ml` and `sa` with an in-app language picker
(`kotlin_project_engineering_standard.md` section 8). These build settings are **mandatory**.
Each one prevents a specific failure, so none of them may be removed.

### `app/build.gradle.kts`

```kotlin
android {
    defaultConfig {
        // Keep only our three languages — library strings in other languages are
        // stripped, so no Hindi (or other) text can appear in the app.
        // On AGP versions that provide androidResources.localeFilters, use that
        // instead (it replaces the deprecated resourceConfigurations).
        resourceConfigurations += listOf("en", "ml", "sa")
    }

    androidResources {
        // Android 13+ lists the app under system "App languages". Needs AGP 8.1+
        // and res/resources.properties (below).
        generateLocaleConfig = true
        // localeFilters += listOf("en", "ml", "sa")   // newer AGP: use this line
    }

    bundle {
        language {
            // Play would otherwise install only the phone's language, and the
            // in-app picker could not switch to the other two.
            enableSplit = false
        }
    }

    lint {
        error += setOf(
            "MissingTranslation",
            "ExtraTranslation",
            "StringFormatMatches",
            "StringFormatCount",
        )
    }
}

tasks.withType<Test>().configureEach {
    // Lets the JVM parity and label-length tests find every module's res/ folder.
    systemProperty("projectRoot", rootDir.absolutePath)
}

dependencies {
    implementation(libs.androidx.appcompat)            // per-app language API
    implementation(libs.androidx.compose.material.icons.core) // About badge heart icon
    testImplementation(libs.icu4j)                     // label-length test (8.6)
    testImplementation(libs.org.json)                  // JVM parsing of app_config.json (8.7)
}
```

### `app/src/main/res/resources.properties`

```properties
unqualifiedResLocale=en
```

### Version catalog entries

```toml
[versions]
appcompat = "1.7.1"    # Example — needs 1.6+; pin the current stable line.
icu4j = "77.1"         # Example — pin the current stable line.
orgJson = "20250517"   # Example — pin the current stable line.

[libraries]
androidx-appcompat = { group = "androidx.appcompat", name = "appcompat", version.ref = "appcompat" }
androidx-compose-material-icons-core = { group = "androidx.compose.material", name = "material-icons-core" }
icu4j = { group = "com.ibm.icu", name = "icu4j", version.ref = "icu4j" }
org-json = { group = "org.json", name = "json", version.ref = "orgJson" }
```

`material-icons-core` takes its version from the Compose BOM.

### `AndroidManifest.xml`

The baseline `minSdk` is 24. Below Android 13, AppCompat saves the user's language only when this
service is declared:

```xml
<application ...>
    <service
        android:name="androidx.appcompat.app.AppLocalesMetadataHolderService"
        android:enabled="false"
        android:exported="false">
        <meta-data
            android:name="autoStoreLocales"
            android:value="true" />
    </service>
</application>
```

Do not also add `android:localeConfig` by hand while `generateLocaleConfig = true`.

### Activity and theme

- `MainActivity` MUST extend `AppCompatActivity` (Compose `setContent { }` works the same).
  Below Android 13, `AppCompatDelegate.setApplicationLocales` does nothing for a plain
  `ComponentActivity`.
- The XML theme in `res/values/themes.xml` MUST have an AppCompat parent. The default Compose
  project theme (`android:Theme.Material.Light.NoActionBar`) crashes `AppCompatActivity` at launch
  with "You need to use a Theme.AppCompat theme".

```xml
<style name="Theme.MyApp" parent="Theme.AppCompat.DayNight.NoActionBar" />
```

### Verify the generated locale config

Once per app, and again after an AGP upgrade: build the release bundle or APK, open it in
Android Studio (**Build → Analyze APK**), open `AndroidManifest.xml`, follow the
`android:localeConfig` attribute to its XML file, and confirm it lists exactly `en`, `ml` and `sa`.
If it does not, set `generateLocaleConfig = false`, write `res/xml/locales_config.xml` by hand
with those three locales, and point `android:localeConfig="@xml/locales_config"` at it.

---

## Product Flavors

Product flavors are optional. Use them when the app needs distinct environments or configurations.

### Environment-Based Flavors

```kotlin
android {
    flavorDimensions += "environment"
    productFlavors {
        create("dev") {
            dimension = "environment"
            applicationIdSuffix = ".dev"
            versionNameSuffix = "-dev"
            // Dev-specific BuildConfig fields
            buildConfigField("String", "BASE_URL", "\"https://dev-api.example.com\"")
        }
        create("prod") {
            dimension = "environment"
            buildConfigField("String", "BASE_URL", "\"https://api.example.com\"")
        }
    }
}
```

### Build Commands With Flavors

```bash
# Debug
./gradlew assembleDevDebug
./gradlew installDevDebug

# Release
./gradlew assembleProdRelease
./gradlew bundleProdRelease

# Tests
./gradlew testDevDebugUnitTest
```

---

## R8 / ProGuard Configuration

### `proguard-rules.pro`

A baseline ProGuard rules file for Kotlin Android Compose projects:

```proguard
# Keep application entry points
-keep public class * extends android.app.Application
-keep public class * extends android.app.Activity

# Room — keep entities and DAOs
-keep @androidx.room.Entity class * { *; }
-keep @androidx.room.Dao class * { *; }

# Moshi — keep JSON adapters (if using Moshi code gen)
-keep class **JsonAdapter { *; }
-keepclassmembers @com.squareup.moshi.JsonClass class * { *; }

# Kotlin serialization (if used)
-keepattributes *Annotation*, InnerClasses
-dontnote kotlinx.serialization.AnnotationsKt

# Kotlin coroutines
-keepnames class kotlinx.coroutines.internal.MainDispatcherFactory {}
-keepnames class kotlinx.coroutines.CoroutineExceptionHandler {}
-keepclassmembers class kotlinx.coroutines.** {
    volatile <fields>;
}

# Keep BuildConfig
-keep class **.BuildConfig { *; }
```

### Testing R8

Always test the release build on a device after enabling R8 for the first time or after adding
a new dependency. Symptoms of missing ProGuard rules:
- `ClassNotFoundException` only in release builds
- `NoSuchMethodException` for methods accessed via reflection
- Crash on JSON deserialization (missing Moshi/Gson adapters)

---

## Common Build Tasks Reference

| Task | Command |
|------|---------|
| Build debug APK | `./gradlew assembleDebug` |
| Build release APK | `./gradlew assembleRelease` |
| Build App Bundle | `./gradlew bundleRelease` |
| Install debug on device | `./gradlew installDebug` |
| Run unit tests | `./gradlew testDebugUnitTest` |
| Run a single test class | `./gradlew testDebugUnitTest --tests "com.example.MyTest"` |
| Run instrumented tests | `./gradlew connectedDebugAndroidTest` |
| Android lint | `./gradlew lint` |
| List dependencies | `./gradlew app:dependencies --configuration releaseRuntimeClasspath` |
| Clean build | `./gradlew clean` |
| Roborazzi record | `./gradlew recordRoborazziDebug` |
| Roborazzi compare | `./gradlew compareRoborazziDebug` |
| Roborazzi verify | `./gradlew verifyRoborazziDebug` |

---

## Secrets Gradle Plugin

For apps that need environment variables at build time (API keys, feature flags), the
[Secrets Gradle Plugin](https://github.com/google/secrets-gradle-plugin) reads a properties
file and injects values into `BuildConfig` or the manifest:

```kotlin
plugins {
    alias(libs.plugins.secrets)
}

secrets {
    propertiesFileName = ".env"
    defaultPropertiesFileName = ".env.example"
}
```

Create `.env` (git-ignored) with real values and `.env.example` (committed) with placeholder
values.

---

## JVM Toolchain For Tests

For compileSdk 36 and later, Robolectric may require Java 21 for test workers. Pin the test
JVM separately from the compilation JVM:

```kotlin
// Compilation uses Java 17
compileOptions {
    sourceCompatibility = JavaVersion.VERSION_17
    targetCompatibility = JavaVersion.VERSION_17
}

// Test workers use Java 21 (for Robolectric + SDK 36)
tasks.withType<Test>().configureEach {
    javaLauncher.set(
        javaToolchains.launcherFor { languageVersion.set(JavaLanguageVersion.of(21)) },
    )
}
```

---

## About-Screen Build Date

To inject a build timestamp into `BuildConfig`:

```kotlin
import java.text.SimpleDateFormat
import java.util.Date
import java.util.Locale

android {
    buildFeatures {
        buildConfig = true
    }

    defaultConfig {
        val buildDate = SimpleDateFormat("yyyy-MM-dd HH:mm", Locale.US).format(Date())
        buildConfigField("String", "BUILD_DATE", "\"$buildDate\"")
    }
}
```

Access in code:

```kotlin
val buildDate = BuildConfig.BUILD_DATE
```
