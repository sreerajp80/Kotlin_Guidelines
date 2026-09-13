# Plan: Sanskrit Button Form — Cover Verbs That Take No Object

**Status:** completed (see `change_log/20260913_193500_sanskrit-button-form-intransitive-verbs.md`)

**Date:** 2026-09-13

## 1. What the issue is

Engineering standard §8.5.1 ("Form conventions") says:

> A button or menu item (action commanding the app): polite imperative passive (`कर्मणि लोट्`, `-ताम्`),
> e.g. `रक्ष्यताम्` (Save), `अन्विष्यताम्` (Search).

This is correct for verbs that take an object (Save *something*, Search *something*). But some
buttons use verbs that take no object, such as Exit (`निष्क्रम्यताम्`, reviewed and confirmed by the
owner on 2026-09-13). Their `-ताम्` form is the impersonal imperative (`भावे लोट्`), not the passive
(`कर्मणि लोट्`). The rule does not mention this case, so a reader could think `निष्क्रम्यताम्` breaks
the rule, or label such forms wrongly.

The Flutter guidelines repository has the same sentence in its §8.5.1.

## 2. Files to be changed

| Repository | File | Change |
|---|---|---|
| Kotlin (this repo) | `kotlin_project_engineering_standard.md` §8.5.1 | Extend the button form bullet (one line) |
| Flutter guidelines | `flutter_project_engineering_standard.md` §8.5.1 | Same one-line change |
| Kotlin (this repo) | `change_log/` | New change log covering both repositories |

## 3. The fix

Replace the bullet in both files with:

> - A button or menu item (action commanding the app): polite `-ताम्` imperative (`लोट्`). For a verb
>   that takes an object it is passive (`कर्मणि`), e.g. `रक्ष्यताम्` (Save), `अन्विष्यताम्` (Search); for a
>   verb that takes no object it is impersonal (`भावे`), e.g. `निष्क्रम्यताम्` (Exit).

No glossary words change. No other rule changes.

## 4. After implementation

Write `change_log/20260913_hhMMss_sanskrit-button-form-intransitive-verbs.md` referencing this plan,
set this plan's `Status:` to `completed`, and leave both repositories uncommitted.
