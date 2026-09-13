## What does `architecture.md` say?

It's a **living blueprint template** for your Kotlin Android app. It has 22 sections covering every architectural decision an Android project needs to make explicit:

- What the app is and what SDKs it targets (Section 1–2)
- How the codebase is structured — Tier 1 flat vs Tier 2 feature-first (Section 3–4)
- The exact startup sequence in `Application.onCreate()` and `MainActivity.onCreate()` and why order matters (Section 5)
- How the app behaves when backgrounded, killed, or memory-pressured (Section 6)
- Whether the app needs internet and how that's enforced in the manifest (Section 7)
- State management choice (ViewModel + StateFlow) and boundaries (Section 8)
- How data flows from Composable → ViewModel → Repository → Room / Network (Section 9)
- How errors are classified and surfaced via sealed classes (Section 10)
- Your Room schema, migration history, and indexes (Section 11)
- DI (Hilt / manual), navigation, persistence, build types/flavors, UI system, logging, testing (Sections 12–18)
- Performance constraints, key decisions, and known risks (Sections 19–21)

Right now it's a **blank template** — every field says `<placeholder>`. You fill it with your project's actual decisions.

---

## How do you use it in a Kotlin Android project?

Think of it as three things simultaneously:

**1. A decision-forcing checklist before you write code**
Going through each section forces you to make explicit choices early — state management, error strategy, Room schema, offline enforcement — rather than discovering conflicts mid-build.

**2. A living reference during development**
Every time a major decision is made or changed — new migration, new screen added, new risk identified — you update the relevant section. It stays current with the codebase.

**3. An onboarding document**
Anyone (including your future self six months later) can read it and understand the entire system without reading the code. Section 5 alone tells you the exact initialization sequence needed to avoid release-only crashes.

---

## What should you fill out BEFORE starting the project?

These sections must be decided and written **before writing a single line of code**, because they shape every file you create:

| Priority | Section | Why Before Coding |
|----------|---------|-------------------|
| 🔴 Must | **§1 Scope** | Locks down minSdk, targetSdk, profiles in force |
| 🔴 Must | **§2 Goals / Non-Goals** | Prevents scope creep decisions mid-build |
| 🔴 Must | **§3 Architecture Summary** | Your one-paragraph north star |
| 🔴 Must | **§4 Repository Structure** | Tier choice + package layout — affects every file |
| 🔴 Must | **§5 Initialization Sequence** | Wrong order = release-only crashes |
| 🔴 Must | **§7 Offline Behavior** | Often the hardest constraint — shapes manifest and dependencies |
| 🔴 Must | **§8 State Management** | Choose ViewModel + StateFlow — document boundaries now |
| 🔴 Must | **§9 Data Flow** | Composable → ViewModel → Repo → Room pattern |
| 🔴 Must | **§11 Domain Model** | Room schema v1, entity list, and migration plan |
| 🔴 Must | **§13 Navigation** | Navigation approach (Compose navigation vs single-Activity tab switching) |
| 🟡 Soon | **§10 Error Handling** | Sealed `AppException` hierarchy — before first repository |
| 🟡 Soon | **§12 Dependency Injection** | DI approach (Hilt vs manual) before writing services |
| 🟡 Soon | **§14 Persistence** | Room WAL mode, foreign key enforcement |
| 🟡 Soon | **§15 Environment & Build** | `debug`/`release` config, signing strategy |
| 🟡 Soon | **§16 UI System** | Theme tokens, color palettes, and accessibility constraints |
| 🟡 Soon | **§19 Operational Constraints** | Performance budget, memory targets, and team conventions |
| 🟢 Later | **§6 Lifecycle** | Fill as you implement `DefaultLifecycleObserver` |
| 🟢 Later | **§17 Logging** | Fill when logging tagging is finalized |
| 🟢 Later | **§18 Testing** | Fill as test suite (Robolectric / Compose UI) grows |
| 🟢 Later | **§20 Decisions** | Record as tradeoffs are made |
| 🟢 Later | **§21 Risks** | Populate from your project's risk register |
| 🟢 Later | **§22 Related Documents** | Sibling and submodule references |

---

## How can AI use this document? What do you need to do?

### How AI uses it
When `architecture.md` is populated and placed in your project, an AI assistant reads it and can:
- Know the exact package structure before creating any file (no wrong-tier mistakes)
- Know the initialization order before writing `onCreate()` (no ordering bugs)
- Know the offline constraint and refuse to suggest packages or manifest permissions that perform network access
- Know the Room schema version before writing any migration (no version conflicts)
- Know the error hierarchy before writing any repository method
- Know which state-management pattern is in force (StateFlow vs LiveData)
- Know which navigation style is in force and use the correct pattern

Without it, the AI has to guess or ask — and sometimes guesses wrong silently.

### What you need to do

**Step 1 — Place it in the right location**
```
<project_root>/docs/architecture.md
```
Your `CLAUDE.md` and `AGENTS.md` should already reference it, but if not, ensure rule references:
```
Rule: Read docs/architecture.md before creating any file, writing any migration,
      or adding any dependency.
```

**Step 2 — Fill out the 🔴 Must sections completely before coding**
Once that's done, the document is your project's ground truth.

**Step 3 — Keep it current as the project evolves**
Every time you add a migration, tell the AI: *"Update §11 schema version to 2, migration adds the new column to its table."* This keeps the document accurate so future sessions don't make stale decisions.

**Step 4 — Reference it explicitly when asking for code**
Instead of:
> *"Write a repository for X"*

Say:
> *"Following docs/architecture.md §9 data flow and §11 schema, write the XRepository"*

---

## How the guideline documents work together

| Document | Answers |
|----------|---------|
| `guideline.md` | *What* common conventions (About config and badge, the three app languages, release keystore, baseline layout) apply across all apps? |
| `kotlin_project_engineering_standard.md` | *How* should all Kotlin Android code be written? Universal rules for every project. |
| `kotlin_build_configuration_guide.md` | *How* exactly do build types, flavors, R8, and signing wire into Gradle? |
| `architecture.md` | *What* did this specific project decide? Tier, packages, Room schema, routes, signing strategy. |
| `security.md` | *What* does this specific project protect? What is sensitive, what is never logged, how is data encrypted? |
| `release_process.md` | *How* does this specific project ship? The exact `./gradlew` commands, checklist, and evidence trail. |
| `CLAUDE_MD_GUIDELINE.md` & `AGENTS_MD_GUIDELINE.md` | *How* should root AI instruction files (`CLAUDE.md` / `AGENTS.md`) be created and maintained? |
| `DOCS_FOLDER_GUIDELINE.md` | *How* should project documentation files in `docs/` be structured and named? |
| `GUIDELINES_MANIFEST.md` | Portable pointer manifest indexing all shared guidelines from `docs/`. |
