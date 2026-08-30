# Explainer: Kotlin Build Configuration Guide

## What is this document?

`kotlin_build_configuration_guide.md` is a technical reference guide for configuring Android builds with Gradle Kotlin DSL (`build.gradle.kts`), managing dependencies with version catalogs (`libs.versions.toml`), configuring release signing, setting up R8/ProGuard, and managing build flavors.

---

## What does it cover?

1. **Toolchain Prerequisites**:
   - Kotlin 2.1.x, Compose BOM, AGP 8.x, Java 17 for build compilation, Java 21 for Robolectric SDK 36 test workers.
2. **Build Types & Signing**:
   - `debug` vs `release` setup.
   - Decoupled `keystore.properties` handling to keep signing credentials out of git.
   - CI environment variable fallback (`System.getenv()`).
3. **Dependency Management**:
   - Centralized `gradle/libs.versions.toml` version catalog rules.
4. **Code Shrinking & ProGuard**:
   - `isMinifyEnabled = true`, `isShrinkResources = true`.
   - Baseline rules for Room, Moshi, coroutines, and Compose.
5. **Product Flavors**:
   - Optional environment dimensions (`dev`, `prod`).
6. **Task Cheatsheet**:
   - Exact `./gradlew` commands for assembling, testing, installing, and running Roborazzi visual regression checks.

---

## When to Consult It

- Setting up a new Android app from scratch.
- Adding release signing configurations and generating signed APKs / AABs.
- Adding a new library that requires ProGuard rules or KSP code generation.
- Resolving Java version or toolchain conflicts (e.g. Robolectric requiring Java 21).
