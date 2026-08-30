# Kotlin Guidelines

This repository is the shared guideline set for my native Android Kotlin apps. It keeps every
app consistent — same conventions, same architecture language, same release and security
practices.

## These are templates

This repository is a **source collection of templates**, not a deployed app. When you adopt
these guidelines in a real project, most of the documents are copied into that app's `docs/`
folder and referenced from its `CLAUDE.md` and `AGENTS.md`; the index/README sits at the app's project root.

That is why cross-references inside the documents use a `docs/` prefix — for example
`docs/architecture.md`. The prefix means "once the file lives in your app's `docs/` folder",
**not** a path inside this repository (here the files are flat, side by side).

## The documents

| Document | What it is |
|---|---|
| [guideline.md](guideline.md) | My personal cross-app conventions: About-screen config, the release keystore rules, and the baseline source package layout. **This is the source of truth for keystore rules.** |
| [kotlin_project_engineering_standard.md](kotlin_project_engineering_standard.md) | The master, project-agnostic rulebook — rules that apply to *every* app (structure, UI, accessibility, performance, database, logging, security, CI, git, Definition of Done). |
| [architecture.md](architecture.md) | A per-project architecture blueprint template. You fill it in with one app's actual decisions. |
| [kotlin_build_configuration_guide.md](kotlin_build_configuration_guide.md) | A technical reference for Gradle Kotlin DSL, product flavors, build types, signing, R8/ProGuard, and version catalogs. |
| [release_process.md](release_process.md) | A step-by-step release runbook — versioning, hardening, signing, build commands, distribution, rollback. |
| [security.md](security.md) | A per-project security blueprint template — threat model, sensitive-data inventory, crypto design, OWASP checklist. |
| [CLAUDE_MD_GUIDELINE.md](CLAUDE_MD_GUIDELINE.md) | Mandatory guideline for creating and maintaining the project-root `CLAUDE.md` for every Kotlin project (**MUST**). |
| [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md) | Mandatory guideline for creating and maintaining the project-root `AGENTS.md` for other LLMs and AI agents (**MUST**). |
| [DOCS_FOLDER_GUIDELINE.md](DOCS_FOLDER_GUIDELINE.md) | How to create files in a project's `docs/` folder (local vs submodule, naming rules, file anatomy, catalog of recognized doc types). |
| [GUIDELINES_MANIFEST.md](GUIDELINES_MANIFEST.md) | Single, portable manifest file copied into a project's `docs/` folder to index all shared guidelines. |

## Where do I start?

- **Writing / maintaining project root `CLAUDE.md` & `AGENTS.md` (MUST)** — follow [CLAUDE_MD_GUIDELINE.md](CLAUDE_MD_GUIDELINE.md) and [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md).
- **Starting a new app** — read [guideline.md](guideline.md) (the conventions to follow
  from day one) and [kotlin_project_engineering_standard.md](kotlin_project_engineering_standard.md)
  (the rules to build to).
- **Structuring project `docs/` files** — follow [DOCS_FOLDER_GUIDELINE.md](DOCS_FOLDER_GUIDELINE.md).
- **Adding guidelines to an existing app** — copy [GUIDELINES_MANIFEST.md](GUIDELINES_MANIFEST.md) to your app's `docs/` folder.
- **Designing one app's structure** — fill in [architecture.md](architecture.md) for that app.
- **Setting up build configuration** — see [kotlin_build_configuration_guide.md](kotlin_build_configuration_guide.md).
- **Shipping a release** — follow [release_process.md](release_process.md).
- **Handling sensitive data** — fill in [security.md](security.md) for that app.

## What applies where (by profile)

The engineering standard defines three applicability profiles. A document (or a marked section
of one) switches on only when its profile applies, so a small app is never forced into
release-process or high-security rules that do not fit it. Pick your app's profiles, then read
across the row.

| Profile | Applies to | Documents / sections in force |
|---|---|---|
| `Core Baseline` | Every app | Root `CLAUDE.md` (via [CLAUDE_MD_GUIDELINE.md](CLAUDE_MD_GUIDELINE.md), **MUST**); Root `AGENTS.md` (via [AGENTS_MD_GUIDELINE.md](AGENTS_MD_GUIDELINE.md), **MUST**); [guideline.md](guideline.md); the Core Baseline rules of [kotlin_project_engineering_standard.md](kotlin_project_engineering_standard.md); [architecture.md](architecture.md) (fill in what applies); [DOCS_FOLDER_GUIDELINE.md](DOCS_FOLDER_GUIDELINE.md) |
| `Production App Extension` | Apps shipped to real users / QA / stores | The above **plus** [release_process.md](release_process.md), [kotlin_build_configuration_guide.md](kotlin_build_configuration_guide.md) (if using flavors), and the `Production App Extension` sections of the engineering standard |
| `Sensitive Data Extension` | Apps handling secrets, PII, health, finance, or local encrypted stores | The above **plus** [security.md](security.md) and the `Sensitive Data Extension` sections of the engineering standard |

Profiles stack: a shipped password manager is in all three; a small internal tool is in
`Core Baseline` only.

## Plain-English explainers

Several of the technical documents have a matching `<name>_README.md` explainer in the
[docs/](docs/) folder that describes, in simple English, what the document says and how to use
it. Open the explainer first if a document looks dense. The available explainers are:

- [docs/architecture_README.md](docs/architecture_README.md)
- [docs/kotlin_project_engineering_standard_README.md](docs/kotlin_project_engineering_standard_README.md)
- [docs/kotlin_build_configuration_guide_README.md](docs/kotlin_build_configuration_guide_README.md)
- [docs/release_process_README.md](docs/release_process_README.md)
- [docs/security_README.md](docs/security_README.md)

`guideline.md` is short enough to read directly and has no separate explainer.
