# Release Process

Use this document for repositories that ship builds to QA, external testers, enterprise
distribution, or public app stores.

If the repository is not release-tracked yet, keep this file short and mark the current release
scope clearly.

---

## 1. Release Scope

- App: `<app name>`
- Release profile: `internal`, `beta`, `public`, or `not yet shipping`
- Supported release platform: `Android`
- Engineering standard profiles in force:
  - `Core Baseline`
  - `Production App Extension`
  - `Sensitive Data Extension` if applicable

> **Every app is always ready for Google Play**, even when its release profile is
> `not yet shipping`. The build-time items in §9A.1–§9A.4 apply from day one. The console items in
> §9A.5–§9A.8 MUST be done before the first upload.

---

## 2. Roles And Responsibilities

| Role | Responsibility | Owner |
|------|----------------|-------|
| Release owner | Coordinates release readiness and final sign-off | `<name/team>` |
| Engineering | Code freeze, fixes, validation | `<name/team>` |
| QA | Test execution and regression sign-off | `<name/team>` |
| Store or distribution owner | Uploads artifacts and manages release metadata | `<name/team>` |

---

## 3. Versioning Policy

- Version format: `versionName` = `MAJOR.MINOR.PATCH` or `MAJOR.MINOR`; `versionCode` = integer
- Source of truth: `app/build.gradle.kts` (`versionCode` and `versionName` in `defaultConfig`)
- Build-number increment rule: `<rule>`
- Git tag format: `vX.Y.Z`

---

## 4. Branch And Merge Policy

- Release branch strategy: `<main only / release branches / trunk-based>`
- Hotfix strategy: `<strategy>`
- Required checks before merge:
  - `<ci checks>`
  - `<review requirements>`

---

## 5. Environment And Build Matrix

| Build Type | Purpose | Example Command |
|------------|---------|-----------------|
| `debug` | Local development | `./gradlew assembleDebug` |
| `release` | Final release artifact | See section 9 for full commands |

If the project uses product flavors, add a row per flavor × build type combination.

---

## 6. Release Build Hardening

All production release builds SHOULD include the following settings. Omitting any of them
without documented justification is a review item.

### 6.1 R8 Code Shrinking And Obfuscation

```kotlin
buildTypes {
    release {
        isMinifyEnabled = true
        isShrinkResources = true
        proguardFiles(
            getDefaultProguardFile("proguard-android-optimize.txt"),
            "proguard-rules.pro",
        )
    }
}
```

R8 performs dead code elimination, obfuscation, and optimization. The `mapping.txt` file
produced at `app/build/outputs/mapping/release/mapping.txt` is required to decode stack traces
from production crash reports.

**Mapping file archive policy:**
- The mapping file MUST be archived securely after every production release build.
- Mapping files MUST be retained for the lifetime of the released version.
- Mapping files MUST NOT be committed to source control.
- Store them alongside the release artifact: e.g. `releases/v1.2.3/mapping.txt`.
- Without the mapping file, stack traces from that version are permanently unreadable.

### 6.2 ProGuard Rules

Verify `proguard-rules.pro` is present and covers:
- Room database classes (entities, DAOs)
- Moshi/Gson serialization annotations (if used)
- Any class accessed via reflection
- Kotlin coroutines internals (usually handled by the default rules)

Always perform a full release build test after adding a new dependency, as R8 can silently strip
classes only accessed via reflection. Symptoms: `ClassNotFoundException` or `NoSuchMethodException`
only in release builds.

### 6.3 App Size Analysis

Monitor APK size before every release:

```bash
# Build release APK
./gradlew assembleRelease

# Check APK size
ls -lh app/build/outputs/apk/release/
```

For detailed size analysis, use Android Studio: Build → Analyze APK → select the APK.

Size budgets (from engineering standard):

| Artifact | Target | Hard Limit |
|----------|--------|------------|
| APK arm64-v8a | < 30 MB | 50 MB |
| AAB download size | < 20 MB | 40 MB |

### 6.4 Debuggable Verification

Verify that `android:debuggable` is `false` in the merged release manifest before every
production release.

```bash
aapt2 dump badging app/build/outputs/apk/release/app-release.apk | grep -i debuggable
```

```powershell
aapt2 dump badging app\build\outputs\apk\release\app-release.apk | Select-String -Pattern debuggable
```

---

## 7. Signing And Secret Handling

> Keystore location, `keystore.properties` naming, and the `.gitignore` rules: see
> `guideline.md §2` (the source of truth).

- Signing config location: `<keystore.properties / local.properties / CI secret store>`
- Keystore or certificate ownership: `<owner>`
- Secret rotation process: `<brief process>`
- Rules:
  - Signing material must not live in source control.
  - Local signing helpers must not expose secrets in committed files.
  - CI logs must not print signing secrets.
  - Keystore files MUST be backed up in at least two separate secure locations.
    Losing the keystore means being unable to publish updates to the Play Store for that app.

---

## 8. Release Checklist

Complete these items before every release.

### Code And Quality

- [ ] Required CI checks passed.
- [ ] `./gradlew lint` passed with zero errors.
- [ ] `./gradlew testDebugUnitTest` passed.
- [ ] Instrumented tests passed if applicable.
- [ ] No critical or release-blocking bugs remain open.
- [ ] KSP code generation is current: Room `*_Impl` files are up to date.

### Performance

- [ ] Release build profiled for jank on primary user flow.
- [ ] App size analyzed and within budget (see section 6.3).
- [ ] Startup time verified under 2 seconds on a mid-range device.

### Security

- [ ] R8 `isMinifyEnabled = true` applied to release build type (or documented exemption).
- [ ] Mapping file (`mapping.txt`) archived securely for this version.
- [ ] ProGuard rules verified.
- [ ] `android:debuggable=false` confirmed in merged release manifest.
- [ ] Manifest and permission review completed — no unnecessary permissions.
- [ ] OWASP Mobile Top 10 checklist reviewed (see `security.md`).
- [ ] Secrets, keys, and backup settings reviewed if applicable.

### Localization

- [ ] `values/`, `values-ml/` and `values-sa/` `strings.xml` present in every module with UI text;
      `TranslationParityTest` passes; lint `MissingTranslation` clean and not suppressed.
- [ ] No untranslated English value left in the Malayalam or Sanskrit files.
- [ ] `scripts/check_sanskrit.sh` passes and glossary terms are used (engineering standard §8.5).
- [ ] New or changed Malayalam/Sanskrit wording reviewed by a fluent reader.
- [ ] `LabelLengthTest` passes — short labels within budget in all three languages (§8.6).
- [ ] Every screen opened in `en`, `ml` and `sa` on a clean device — no missing glyphs, no
      overflow, no clipped Malayalam/Devanagari letters (§8.3.4).
- [ ] In-app language picker works: System default / English / മലയാളം / संस्कृतम्, persists across
      restart, applies without restart, and on Android 13+ matches system "App languages" (§8.4).
- [ ] Date and time pickers checked under `sa` — readable months, Western digits (§8.3.2).
- [ ] A notification (if the app has any) shows in the chosen language on Android 12 or older (§8.7).
- [ ] `scripts/check_icon_buttons.sh` passes — every icon-only control has a localized tooltip (§7.7).
- [ ] About screen ends with the "Made with ❤️ from India" badge, red heart, localized, centered
      (`guideline.md` §1.4).
- [ ] Generated locale config lists exactly `en`, `ml`, `sa` (build guide, "Languages").

### Google Play Store Readiness

- [ ] Full §9A gate completed for this release.
- [ ] `targetSdk` meets Play's current target API level policy (re-checked, not assumed).
- [ ] `versionCode` strictly greater than every previously uploaded build.
- [ ] App Bundle built with language split disabled; Play App Signing enabled; `mapping.txt` uploaded.
- [ ] Permissions justified; `AD_ID` removed if unused; sensitive-permission declarations completed.
- [ ] Privacy policy URL live; Data safety form matches actual behavior; content rating done.
- [ ] Store listing assets ready at the required sizes (icon, feature graphic, screenshots).
- [ ] English and Malayalam listings complete with screenshots in that language.
- [ ] Internal-testing upload done; pre-launch report clean; Play-served build checked in all three
      languages, including one different from the phone's language.
- [ ] Staged rollout percentage chosen and Android vitals monitoring planned.

### Product And Documentation

- [ ] `versionCode` and `versionName` updated in `app/build.gradle.kts`.
- [ ] Changelog or release notes updated.
- [ ] User-visible behavior changes documented.
- [ ] Required store metadata ready.

### Artifact Validation

- [ ] Intended release artifact built successfully.
- [ ] Artifact installs and launches correctly on a clean device / emulator.
- [ ] Version name and build number correct in the About screen.
- [ ] About screen ends with the "Made with ❤️ from India" badge in all three languages.
- [ ] Release build tested end-to-end (not just debug build).

---

## 9. Android Release Steps

1. Pull the intended release commit and verify it is clean (`git status`).
2. Verify `versionCode` and `versionName` in `app/build.gradle.kts`.
3. Run lint and test checks.
4. Build the required Android artifacts.
5. Run size analysis and record output.
6. Verify `android:debuggable=false` in the merged manifest.
7. Verify artifact naming, installability, and environment on a physical or emulated device.
8. Run the language gates (`scripts/check_sanskrit.sh`, `scripts/check_icon_buttons.sh`,
   `scripts/check_lint_suppressions.sh`) and check the app in `en`, `ml` and `sa`.
9. Archive `mapping.txt` from `app/build/outputs/mapping/release/`.
10. Complete the Google Play readiness gate (§9A) before uploading to Play.
11. Upload to the intended distribution channel.
12. Tag the release in git: `git tag v<version>` and push.

### Android Build Commands

```bash
# Validate
./gradlew lint
./gradlew testDebugUnitTest

# Build release APK
./gradlew assembleRelease

# Build App Bundle for Google Play
./gradlew bundleRelease

# Install release APK on connected device
adb install -r app/build/outputs/apk/release/app-release.apk
```

If the project uses product flavors:

```bash
./gradlew assembleProdRelease
./gradlew bundleProdRelease
```

### Split APKs

To generate per-ABI split APKs, add to `android {}` block in `app/build.gradle.kts`:

```kotlin
splits {
    abi {
        isEnable = true
        reset()
        include("armeabi-v7a", "arm64-v8a", "x86", "x86_64")
        isUniversalApk = false
    }
}
```

---

## 9A. Google Play Store Readiness (Mandatory Gate)

Every app is built to be publishable on Google Play at any time.

- **Build-time items (§9A.1–§9A.4)** live in the code and build files. They MUST hold from day
  one, even before the first release, and are re-checked before every release.
- **Console-time items (§9A.5–§9A.8)** happen in the Play Console. They MUST be done before the
  first upload and re-checked before every production release. Items marked *(one-time)* are set
  up once and only re-verified afterwards.

Play policies change. Values marked "at the time of writing" MUST be re-checked against the
current Play Console Help pages before each release.

### 9A.1 Application identity and versioning (build-time)

| Item | Requirement |
|---|---|
| `applicationId` *(one-time)* | Reverse-DNS, owned domain, lowercase, permanent. It can never be changed after the first publish. Flavors may append a suffix (`.dev`), but the production id MUST have no suffix. |
| `versionCode` | Strictly increasing integer on every upload, never reused — even for a rejected or rolled-back build. |
| `versionName` | Matches the version shown on the About screen (`guideline.md` §1). |
| App name | `@string/app_name` in all three `strings.xml` files (or `translatable="false"` for a brand name, recorded in `docs/architecture.md`), ≤ 30 characters to match the store title. |
| Package visibility | If the app looks up other installed apps, declare `<queries>`. Play restricts `QUERY_ALL_PACKAGES`. |

### 9A.2 API level, ABI, and compatibility (build-time)

- `targetSdk` MUST meet Play's current target API level policy. At the time of writing, new apps
  and app updates must target API 36 from 31 August 2026. Play raises this every year, so check
  the current requirement before each release rather than trusting the value already in the
  project.
- `compileSdk` ≥ `targetSdk`.
- `minSdk` is a deliberate product decision, recorded in `docs/architecture.md` §19.
- 64-bit native code is mandatory when the app or any dependency ships native `.so` files. The App
  Bundle includes `arm64-v8a` automatically.
- 16 KB page-size compliance is required for apps targeting Android 15+ **when the app or any
  dependency ships native `.so` files** (engineering standard §5.8). Apps with no native code pass
  automatically.
- Edge-to-edge behavior verified (engineering standard §6.7).

### 9A.3 Build and signing (build-time)

- Ship an **Android App Bundle (`.aab`)** from `./gradlew bundleRelease` (or `bundleProdRelease`),
  not an APK, to Play.
- R8 enabled for the release build (§6.1).
- **Language split disabled** (`bundle { language { enableSplit = false } }`) and the generated
  locale config verified (build guide, "Languages"). Without this, a Play install contains only the
  phone's language and the in-app language picker cannot work.
- **Play App Signing** MUST be enabled *(one-time)*. Keep the upload key backed up offline in at
  least two places (§7). A lost upload key can be reset through Play support; a lost
  pre-App-Signing release key cannot be recovered.
- Signing config reads `keystore.properties` (see `guideline.md` §2), which is **never** committed.
- Upload `mapping.txt` with every bundle so crash traces de-obfuscate. Upload native debug symbols
  (`ndk { debugSymbolLevel = "FULL" }`) only when the app has native code.

### 9A.4 Manifest, permissions, and policy (build-time)

- Every permission in the merged manifest is justified and used. Remove anything a dependency
  adds that the app does not need with `tools:node="remove"`.
- Avoid sensitive permissions; where one is truly needed, it requires a declaration in the console
  and is often rejected: all-files access, exact alarms, accessibility service, SMS/call log,
  background location, `QUERY_ALL_PACKAGES`, broad photo/video access (use the system photo picker
  instead of `READ_MEDIA_IMAGES` / `READ_MEDIA_VIDEO`).
- `com.google.android.gms.permission.AD_ID`: remove it with `tools:node="remove"` unless the app
  shows ads or uses analytics that need the advertising ID. Libraries such as Firebase add it
  silently.
- Foreground services declare a `foregroundServiceType` (and need a use-case declaration in the
  console).
- `android:debuggable=false` (§6.4); `android:allowBackup` and `android:dataExtractionRules`
  chosen deliberately; no cleartext traffic; no accidental `android:exported="true"`.
- Show a clear in-app disclosure, and get consent, before collecting any personal or sensitive data.

### 9A.5 Play Console declarations (console-time)

- **Privacy policy URL** — reachable, public, app-specific. Required for every app, whether or not
  it collects data.
- **Data safety form** — matches what the app actually does, including anything a bundled SDK
  collects. A mismatch is a policy violation.
- **Advertising ID declaration** — answered to match the manifest (§9A.4).
- **Content rating questionnaire** — completed.
- **Target audience and content** — declared. If children may be in the audience, the Families
  policy applies.
- **Ads, government, financial features, and health declarations** — where they apply.
- **Account deletion** — if the app lets users create an account, an in-app and a web deletion
  path MUST exist and be declared.
- **Developer contact details** complete. A new personal developer account must run a closed test
  before it can publish to production (at the time of writing, at least 12 testers for 14 days
  in a row).

### 9A.6 Store listing assets (console-time)

| Asset | Requirement |
|---|---|
| App icon | 512 × 512 PNG, 32-bit, up to 1024 KB |
| Feature graphic | 1024 × 500, JPEG or 24-bit PNG (no alpha) |
| Phone screenshots | 2–8, JPEG or 24-bit PNG; each side 320–3840 px; the long side at most 2× the short side |
| Tablet screenshots | Required if the app is offered on tablets (7-inch and 10-inch sets) |
| App title | ≤ 30 characters; no keyword stuffing, ranking claims, or price in the title |
| Short description | ≤ 80 characters |
| Full description | ≤ 4000 characters |

### 9A.7 Listing languages (console-time)

The app itself ships English, Malayalam and Sanskrit (engineering standard section 8).

- The Play listing MUST be provided in **English** and **Malayalam (`ml-IN`)**, each with
  screenshots of the real app UI in that language — not English screenshots under the Malayalam
  listing.
- **Sanskrit is not a Play listing language** (at the time of writing). It ships inside the app
  only. Do not drop Sanskrit from the app because the store cannot list it.

### 9A.8 Pre-launch verification (console-time)

- Upload to **internal testing** first. Run the Play Console **pre-launch report** and fix all
  crashes, ANRs, and flagged accessibility and security issues.
- Install the build **from Play** (not a local APK) and check the primary flow in English,
  Malayalam and Sanskrit, including switching to a language that differs from the phone's language.
  This catches a missing language split setting (§9A.3).
- Production starts as a **staged rollout** (e.g. 10% → 50% → 100%) with Android vitals (crash
  rate, ANR rate) checked at each step.

---

## 10. Distribution Channels

| Channel | Artifact | Audience | Notes |
|---------|----------|----------|-------|
| `<channel>` | `<apk/aab>` | `<audience>` | `<notes>` |
| `<channel>` | `<artifact>` | `<audience>` | `<notes>` |

---

## 11. Rollback And Hotfix Process

- Rollback trigger: `<what forces rollback>`
- Rollback method: `<store halt / phased rollout pause / hotfix release>`
- Hotfix branch naming: `<pattern>`
- Verification after rollback or hotfix:
  - Full release checklist MUST be completed even for hotfixes.
  - Mapping files for the hotfix build MUST be archived.

---

## 12. Release Evidence

Store links or references to release evidence here after each release.

- CI run: `<url or identifier>`
- Test report: `<url or identifier>`
- Size analysis output: `<location>`
- Mapping file archive: `<secure location>`
- Built artifact: `<location>`
- Release notes: `<location>`
- Store submission or rollout record: `<location>`
- OWASP checklist sign-off: `<signed by / date>`

---

## 13. Post-Release Checks

- [ ] Crash and error monitoring reviewed (if applicable; for offline apps: post-install test on
      clean device).
- [ ] User-reported issues triaged.
- [ ] Release tag created and pushed: `git tag v<version> && git push origin v<version>`.
- [ ] Mapping file confirmed in secure archive.
- [ ] Follow-up tasks recorded.
