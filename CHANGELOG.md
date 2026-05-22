# Changelog

Each entry records a versioned change to a specification in this repository.
Each specification is versioned independently, starting at version 1. Every
pull request that alters normative content bumps the affected spec's `Version`
field and adds an entry here.

## OVOS-INTENT-1 — Sentence Template Grammar

### 2

**Breaking change.** Version 2 adds the `<name>` inline vocabulary reference
token. A template using `<name>` is not valid version-1 syntax — a version-1
tool does not recognize the token and cannot expand the template.

- §3.7 (new) — the `<name>` inline vocabulary reference: a token replaced
  during expansion by a named slot-free vocabulary (a `.voc`, OVOS-INTENT-2).
- §3 — `<name>` added to the grammar token table; §1 lists it under the
  expansion facet.
- §2 — `<` and `>` added to the structural metacharacters that cannot occur as
  literal input.
- §4.1 — expansion gains a first step that resolves `<name>` references
  recursively, before the `[x]` / `()` steps.
- §3.6 — new malformed forms: a reference to an undefined vocabulary, and a
  cyclic reference chain; `<` and `>` join the unbalanced-metacharacter rule.
- §7 — the Expander conformance role MUST resolve inline vocabulary references.

### 1

- Initial draft.

## OVOS-INTENT-2 — Locale Resource Formats

### 2

- §1, §4.3 — the `.voc` role description is broadened to "a named set of
  localized phrasings": a `.voc` is consumed as a keyword vocabulary and/or
  referenced inline via `<name>` (OVOS-INTENT-1 §3.7), and may itself contain
  such references.

### 1

- Initial draft.

## OVOS-INTENT-3 — Intent Definition

### 1

- Initial draft.
