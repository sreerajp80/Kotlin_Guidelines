# CLAUDE.md — Kotlin Guidelines Repository

This file is read by Claude Code at the start of every session in this repository.
Read it before making any changes to the guidelines, standards, or blueprint templates.

---

## Project Identity

| Field | Value |
|-------|-------|
| Repository name | Kotlin_Guidelines |
| Type | Central engineering standards and template repository for Kotlin Android projects |
| Target platform | Android (Kotlin + Jetpack Compose) |
| Toolchain baseline | Kotlin 2.1.x, AGP 8.x, Compose BOM 2025.x, JDK 17, Gradle 8.14+, compileSdk 36 |

---

## Repository Structure & Document Index

| Document | Purpose |
|----------|---------|
| `README.md` | Overview of the guidelines set, template explanation, and applicability profiles |
| `GUIDELINES_MANIFEST.md` | Portable pointer manifest copied into an app's `docs/` folder |
| `guideline.md` | Cross-app conventions (About-screen config and "Made with ❤️ from India" badge, English/Malayalam/Sanskrit languages, release keystore rules, source package layout) |
| `kotlin_project_engineering_standard.md` | Master 24-section project-agnostic engineering rulebook |
| `architecture.md` | Per-project architecture blueprint template |
| `kotlin_build_configuration_guide.md` | Gradle Kotlin DSL, signing, version catalogs, R8/ProGuard, and flavors reference |
| `release_process.md` | Production release runbook and checklist |
| `security.md` | Security blueprint and OWASP Mobile Top 10 checklist |
| `CLAUDE_MD_GUIDELINE.md` | Standard for creating and maintaining project-root `CLAUDE.md` in Kotlin apps |
| `AGENTS_MD_GUIDELINE.md` | Standard for creating and maintaining project-root `AGENTS.md` in Kotlin apps |
| `DOCS_FOLDER_GUIDELINE.md` | Standard for creating files in a project's `docs/` folder |
| `docs/*_README.md` | Plain-English explainers for dense documents |
| `plans/` | Plan files for changes to this repository |
| `change_log/` | Change log records for changes to this repository |

---

## Hard Rules

1. **Templates repository.** This repository contains templates and standards, not a deployed application. Documents must remain project-agnostic unless providing concrete reference implementations.
2. **Relative paths only.** All cross-links between guideline documents MUST use relative Markdown links.
3. **No secrets or machine details.** Never commit sensitive credentials, API keys, keystores, or machine-specific paths to this repository.

---

## Workflow Rules (Mandatory)

Every change follows plan-before-changing and log-after-changing:

1. **Plan before changing.** Write a full plan to `plans/` named `yyyymmdd_hhMMss_<short-slug>.md` with a `**Status:**` line, the files to change, the issue, and the fix. Then **STOP and get explicit approval** before editing/creating/deleting any project file (other than the plan). A question or ambiguous reply is not approval.
2. **Log after changing.** After implementing, write a change log to `change_log/` named `yyyymmdd_hhMMss_<short-slug>.md` describing what changed and referencing its plan.
3. **Relative paths & privacy only.** `plans/` and `change_log/` files are committed and may become public on the internet. They MUST use relative repository paths only (never absolute system paths like `C:\...`, `l:\...`, or `file:///...`). They MUST NOT contain any **local system details** — OS user name, computer/host name, home or drive-letter paths, network share names, LAN/internal IP addresses, local server URLs with ports, device serial numbers, personal email addresses — or any secret (API keys, tokens, passwords, keystore passphrases, credentials, PII). Write them as if a stranger will read them; nothing should reveal the machine they came from.

---

## Communication Rules

- **Always use simple English.** Write all responses, plans, change logs, and explanations in plain, simple English. Short sentences, common words. Explain any jargon you must use.
