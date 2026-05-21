# Changelog

Each entry records a versioned change to a specification in this repository.
Every pull request that alters normative content bumps the affected spec's
`Version` field and adds an entry here.

## OVOS-INTENT-1 — Sentence Template Grammar

### 1.1

- §3.6 — adjacent named slots and repeated slot names are now **malformed**; a
  tool MUST reject them. The adjacency check applies to every expanded sample,
  not only the template surface, so `{a} [foo] {b}` (adjacent when `foo` is
  empty) is malformed.
- §3.6 — a template whose sample set contains the empty string is **malformed**
  ("empty sample"); this never affects a group with an empty branch inside an
  otherwise non-empty template, such as the optional `[the]`.
- §3.6 — a single-branch group (`(word)`, `()`) and a slot-only template
  (`{name}` alone) are now **malformed**: a group MUST offer a choice, and a
  template MUST carry at least one literal word.
- §4.1 — the expansion algorithm is reworded to operate on a working *set* of
  strings; the iteration condition is "while any string in the set contains
  `(`", not "while the template contains `(`".
- §5.1 — added explicit handling of optional slots (`[{x}]`): when the
  slot-free branch is taken the slot is absent (no value under match-time fill;
  no value required under caller-supplied fill).
- §5.5 (new) — slot consistency: every template in one intent or dialog
  definition MUST declare the identical set of slot names; mixed-slot or
  mixed slot-bearing/slot-free definitions MUST be rejected.
- §6.2, §7 — engine and dialog-renderer obligations now include verifying slot
  consistency (§5.5).
- §3.4 — slot names may now contain digits (`a`–`z`, `0`–`9`, `_`); a name
  MUST NOT begin with a digit.
- §6.1 — a registration's name is unique within the registering skill.
- §7 — clarified that the Intent engine conformance role covers slot-bearing
  `.intent` templates; a tool consuming only slot-free input resources
  (`.voc`, `.entity`, `.blacklist`) needs only the Expander role.

### 1

- Initial draft.

## OVOS-INTENT-2 — Locale Resource Formats

### 1.1

- §2 — a resource is identified by the pair **(role, base name)**. Files MAY
  share a base name across roles (`confirm.intent` and `confirm.dialog`
  coexist); only files with the same extension must have distinct base names.
- §2.1 — the root directory of core resources is assistant-defined; only the
  `locale/<lang>/` layout beneath it is normative.
- §4.2 — `.dialog` recognizes only the single-brace slot form `{name}`; there
  is no `{{ }}` double-brace form.
- §4.3 — the `.blacklist` suppression contract is now defined: scoped to the
  one `.intent` it is paired with by base name, applying a hard,
  score-independent rejection. A phrase "occurs" as a contiguous whole-word
  subsequence, not a raw substring.
- §4.1, §4.2 — every line in a `.intent` or `.dialog` file MUST declare the
  same set of named slots (OVOS-INTENT-1 §5.5).
- §2 — a language directory MAY contain subdirectories; a loader resolves a
  resource by searching the directory and all subdirectories recursively.
  Same-extension base names must be unique across the whole language tree;
  a loader encountering a duplicate MUST treat the skill as malformed.
- §5 — a loader MUST reject any empty resource file, of any role; every file
  MUST contribute at least one non-empty template.
- Intro — removed the "skill manifest" paragraph (an editorial deletion of a
  non-normative scope pointer; no requirement changed).
- §2.1 — note that `<skill_id>` in the override path is the skill's unique
  identifier.

### 1

- Initial draft.

## OVOS-INTENT-3 — Intent Definition

### 1

- Initial draft.
