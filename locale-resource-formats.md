# Locale Resource Formats Specification

**Spec ID:** OVOS-INTENT-2 · **Version:** 1 · **Status:** Draft · **Basis:** [ovos-workshop](https://github.com/OpenVoiceOS/ovos-workshop) (closest existing implementation; does not yet fully conform)

This document defines the **locale folder layout** and the **plain-text resource
file formats** a skill ships so a voice assistant can recognize what the user
says and produce what the assistant speaks. It is implementation-agnostic: any
assistant, in any language, can adopt this layout and these formats.

Every resource file is a list of **sentence templates** as defined by the
companion *Sentence Template Grammar Specification* (OVOS-INTENT-1). That
grammar has two facets — *expansion* (`(a|b)` / `[x]`) and *named slots*
(`{name}`).

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are used as in
RFC 2119.

---

## 1. Purpose and scope

A *skill* is a self-contained unit of assistant functionality. Beyond its code,
a skill ships **resource files**, grouped by language.

There are **two file formats**, distinguished by whether named slots are
permitted:

- **slot-bearing** — expansion *and* named slots;
- **slot-free** — expansion only.

These two formats are realized as **five resource roles**, identified by file
extension. The role tells a consumer how to use the file; the format tells a
loader how to parse it.

| Role | Extension | Format | Purpose |
|------|-----------|--------|---------|
| Intent | `.intent` | slot-bearing | Templates for utterances that trigger a skill action |
| Dialog | `.dialog` | slot-bearing | Phrases the assistant speaks in response |
| Entity | `.entity` | slot-free | Example values that can fill a named slot |
| Vocabulary | `.voc` | slot-free | Localized keywords / substrings |
| Blacklist | `.blacklist` | slot-free | Words that suppress an intent |

The slot-bearing roles map onto the data path of a voice interaction:

- **`.intent`** — *ASR input*: templates matched against the user's speech;
  their named slots are filled by the engine at match time.
- **`.dialog`** — *TTS output*: templates for what the assistant says back;
  their named slots are filled by the caller before the text is spoken.

`.entity`, `.voc`, and `.blacklist` share the slot-free format; they are how a
developer encodes a set of natural-language phrasings for OVOS to use, and
differ only in *which component consumes them* (§4.3).

This specification covers the **folder layout**, the **common parsing rules**,
and the **two file formats** across their **five roles**. It does not cover
intent scoring, matching, or skill runtime behaviour.

---

## 2. Locale folder layout

All localized resources live under a single `locale/` directory at the skill
root, with **one subdirectory per language**:

```
my-skill/
└── locale/
    ├── en-US/
    │   ├── turn_on.intent
    │   ├── device.entity
    │   ├── turn_on.blacklist
    │   ├── thing.voc
    │   ├── confirm.dialog
    │   └── dialogs/
    │       └── greeting.dialog
    ├── pt-BR/
    │   └── …
    └── de-DE/
        └── …
```

A language directory MAY contain **subdirectories**; a skill author MAY use them
to organize resources in any way they choose. A loader resolves a resource by
searching the language directory **and all its subdirectories, recursively**.
Subdirectory names carry no meaning to a loader — they are an authoring
convenience only.

A resource is identified by the pair **(role, base name)** — its file extension
and its base name together. Two files MAY share a base name when their roles
differ: `confirm.intent` and `confirm.dialog` are distinct resources. Two files
with the **same extension** MUST NOT share a base name anywhere within one
language directory tree, since subdirectories do not distinguish resources. A
loader that nonetheless encounters two such files MUST treat the skill as
malformed.

A resource base name MUST consist only of lowercase ASCII letters, digits, and
underscores, and MUST NOT contain whitespace; file extensions are likewise
lowercase. Where a base name names a slot —
an `.entity` file naming the `{slot}` it supplies — it additionally obeys the
slot-name rule of OVOS-INTENT-1 §3.4 (lowercase letters, digits, and
underscores; not beginning with a digit).

Language directories are named with **BCP-47** language tags (`en-US`, `pt-BR`,
`zh-Hans`). Tag comparison is **case-insensitive**: `en-us` and `en-US` denote
the same language.

### 2.1 Resolution precedence

A resource may be provided from three places. When the same resource — the same
`(role, base name)` pair — exists in more than one, a loader resolves it in this
precedence order (first match wins):

1. **User overrides** — files under a per-skill directory in the platform user
   data path, laid out as `…/<skill_id>/locale/<lang>/`, where `<skill_id>` is
   the skill's unique identifier.
2. **Skill resources** — files bundled in the skill's own `locale/` directory.
3. **Core resources** — fallback files shipped by the assistant framework. The
   root directory holding them is assistant-defined; only the `locale/<lang>/`
   layout beneath that root is normative.

All three use the layout of §2. Overrides apply at **whole-file granularity**:
an override file replaces the corresponding lower-precedence file entirely.

### 2.2 Language fallback (non-normative)

This specification does not mandate behaviour when the requested language has no
directory. A loader **SHOULD** prefer an exact match. As a *suggestion*, a
loader MAY fall back to the nearest available language; the `langcodes` library
provides a `tag_distance()` function for this, and treats a distance below `10`
as a usable regional match. Any such fallback is an **implementation choice**,
not a requirement of this specification, because cross-region substitution can
produce wording a user would not expect.

---

## 3. Common parsing rules

All resource files are line-oriented and share one reader behaviour:

- the file is **UTF-8**; it SHOULD NOT begin with a byte-order mark, and a
  reader that encounters one MUST discard it;
- lines are terminated by `LF` or `CRLF`; a reader MUST accept both;
- lines are read in order, each **stripped** of leading and trailing whitespace;
- a **blank line** is skipped;
- a line whose first character is `#` is a **comment** and is skipped; there are
  no inline (end-of-line) comments.

Each surviving line is one **template** (OVOS-INTENT-1). Format-specific rules
below apply *after* this filtering.

---

## 4. File formats

A file's **format** is determined by its role (§1): `.intent` and `.dialog` are
**slot-bearing**; `.entity`, `.voc`, and `.blacklist` are **slot-free**. Every
file is a list of templates, one per line, parsed per §3; a resource is
identified by its `(role, base name)` pair (§2).

### 4.1 `.intent` — intent training samples

**Format.** Slot-bearing: each line is a template using the full OVOS-INTENT-1
grammar — expansion `(a|b)` / `[x]` and named slots `{name}`.

**Role.** Defines an intent: the templates whose expanded samples train the
engine to recognize one skill action. Matched against **ASR input**; named slots
are filled by the engine at match time (OVOS-INTENT-1 §5.1). The file base name
is the intent name. Every line in the file MUST declare the **same set of named
slots** (OVOS-INTENT-1 §5.5); a phrasing that needs different slots belongs in a
separate `.intent` file.

**Loads as.** The union of the sample sets of all lines (OVOS-INTENT-1 §4) —
training data for the intent, with named slots intact. The engine generalizes
beyond these samples; see OVOS-INTENT-1 §4.

```
# play.intent
(play|put on) {query}
(play|put on) {query} (on|using) {engine}
i want to listen to {query}
```

### 4.2 `.dialog` — spoken response phrases

**Format.** Slot-bearing: each line is a template using expansion `(a|b)` /
`[x]` for wording variety, and named slots `{name}`.

**Role.** Holds the phrases the assistant may speak for one response —
**TTS output**. The file base name is the dialog name. A phrase is output text,
not ASR input, so it MAY contain mixed case and punctuation.

**Slots.** Named slots in a dialog are filled by the **caller** — the skill —
*before* the phrase is rendered to TTS (caller-supplied fill, OVOS-INTENT-1
§5.1). The caller MUST fill **every** slot in the chosen phrase; a phrase with
an unfilled slot MUST NOT be sent to TTS. Every phrase in the file MUST declare
the **same set of named slots** (OVOS-INTENT-1 §5.5), so the caller supplies the
same values whichever phrase is chosen. This is the normal way to inject
**dynamic values** into a spoken response — a weather reading, the current time,
a computed result. An `.entity` value set is normally *not* involved in filling
a dialog slot, though an implementation MAY consult one.

**Limitation.** A `.dialog` phrase uses the metacharacters `( ) [ ] { } |`
structurally and therefore cannot contain any of them as literal spoken text.
This is an accepted constraint; spoken responses rarely require them. A
`.dialog` file recognizes only the single-brace slot form `{name}` (§4.1);
there is no `{{ }}` double-brace form.

**Rendering.** To render a dialog, an implementation selects one phrase, fills
its named slots with caller-supplied values, and expands its `(a|b)` / `[x]`
variety per OVOS-INTENT-1 §4 to pick one variant. *How* a phrase is selected
(random, round-robin, repetition avoidance) is implementation behaviour and is
out of scope here.

**Loads as.** The list of phrase strings. Unlike the other formats, a `.dialog`
file is **not** expanded at load time — expansion happens per-render, on the one
phrase chosen.

```
# weather_today.dialog
It is currently {temperature} degrees and {condition}.
Right now it's {condition}, {temperature} degrees.
(Currently|At the moment) it is {temperature} degrees.
```

### 4.3 Slot-free roles — `.entity`, `.voc`, `.blacklist`

`.entity`, `.voc`, and `.blacklist` share the **slot-free** format: a list of
templates using expansion `(a|b)` / `[x]` only, **no named slots**. They are
syntactically and semantically identical, and a loader parses all three the same
way — each **loads as** the union of the sample sets of all its lines
(OVOS-INTENT-1 §4). They are how a developer encodes a set of natural-language
phrasings for OVOS to consume.

They differ **only in role** — which component reads the file and what it does
with the expanded phrase set:

| Extension | Role | Consumed by | Pairing |
|-----------|------|-------------|---------|
| `.entity` | Example values that can fill a `{slot}` | Intent engine, as a slot value set (OVOS-INTENT-1 §5.4) | Base name = the `{slot}` name it supplies |
| `.voc` | Localized keywords / substrings | Keyword intent engines (e.g. Adapt); skill helpers such as `voc_match` | Base name = the vocabulary name |
| `.blacklist` | Words whose presence suppresses an intent | Intent engine, as match suppression | Base name = the `.intent` it suppresses |

How an `.entity` or `.voc` phrase set is *used* — slot constraint, keyword
test — is engine or skill policy, consistent with matching behaviour being out
of scope for these specifications.

For the `.blacklist` role the suppression contract is defined: a `.blacklist`
file is paired by base name with exactly one `.intent`, and its expanded phrase
set scopes to that intent alone. A blacklist phrase **occurs** in an utterance
when its words appear there as a **contiguous sequence of whole words** — a
token subsequence, not a raw substring (the phrase `art` does not occur within
the word `start`). If any phrase from the set occurs in the user's utterance,
that intent is suppressed — a **hard, score-independent rejection**, not a
confidence penalty. A `.blacklist` does not affect any other intent.

```
# weekday.entity        — values for the {weekday} slot
monday
(wednes|thurs|fri)day
```
```
# yes.voc               — a localized keyword vocabulary
yes
yeah
(sure|of course|absolutely)
```
```
# play_music.blacklist  — suppresses play_music.intent
(music|movie) (trailer|video)
trailer
```

---

## 5. Authoring a conformant loader

A loader for these resources, in any language, **MUST**:

1. **Discover languages** — list the language directories under `locale/`.
2. **Locate a file** — within the resolved language directory, searching its
   subdirectories recursively, find a file by its base name and extension,
   honouring the override precedence of §2.1.
3. **Apply the common reader** — UTF-8, accept `LF`/`CRLF`, strip lines, skip
   blanks and `#`-comments (§3).
4. **Apply the per-format rule**:
   - `.intent` and the slot-free roles (`.entity`, `.voc`, `.blacklist`) —
     expand each line to its sample set at load time via an
     OVOS-INTENT-1-conformant expander, leaving any named slots intact;
   - `.dialog` — retain each line as a phrase string; expand per-render (§4.2).
5. **Reject an empty file** — a resource file of any role that yields no
   templates after step 3 MUST be treated as malformed: every file MUST
   contribute at least one template. (Each template must in turn expand to at
   least one non-empty sample, or it is itself malformed — OVOS-INTENT-1 §3.6.)

A loader **MAY** cache parsed results and **MAY** implement a language-fallback
policy per §2.2, but **MUST NOT** change the meaning of the formats defined
here, and **MUST NOT** introduce additional resource file roles under this
specification.

---

## See also

- *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the template
  grammar, its two facets (expansion and named slots), expansion semantics, and
  the slot fill modes. All five resource roles — `.intent`, `.dialog`,
  `.entity`, `.voc`, `.blacklist` — are lists of templates written in this
  grammar.
