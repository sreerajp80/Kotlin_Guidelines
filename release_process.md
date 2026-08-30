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

### Product And Documentation

- [ ] `versionCode` and `versionName` updated in `app/build.gradle.kts`.
- [ ] Changelog or release notes updated.
- [ ] User-visible behavior changes documented.
- [ ] Required store metadata ready.

### Artifact Validation

- [ ] Intended release artifact built successfully.
- [ ] Artifact installs and launches correctly on a clean device / emulator.
- [ ] Version name and build number correct in the About screen.
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
8. Archive `mapping.txt` from `app/build/outputs/mapping/release/`.
9. Upload to the intended distribution channel.
10. Tag the release in git: `git tag v<version>` and push.

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
