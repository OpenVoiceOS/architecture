# Locale Resource Formats Specification

**Spec ID:** OVOS-INTENT-2 · **Version:** 2 · **Status:** Draft

This document defines the **locale folder layout** and the **plain-text resource
file formats** a skill ships so a voice assistant can recognize what the user
says and produce what the assistant speaks. It is implementation-agnostic: any
assistant, in any language, can adopt this layout and these formats.

Every resource file is a list of **sentence templates** as defined by the
companion *Sentence Template Grammar Specification* (OVOS-INTENT-1). That
grammar has two facets — *expansion* (`(a|b)` / `[x]` / `<name>`) and *named slots*
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

These two formats are realized as **six resource roles**, identified by file
extension. The role tells a consumer how to use the file; the format tells a
loader how to parse it.

| Role | Extension | Format | Purpose |
|------|-----------|--------|---------|
| Intent | `.intent` | slot-bearing | Templates for utterances that trigger a skill action |
| Dialog | `.dialog` | slot-bearing | Phrases the assistant speaks in response |
| Entity | `.entity` | slot-free | Example values that can fill a named slot |
| Vocabulary | `.voc` | slot-free | A named set of localized phrasings |
| Blacklist | `.blacklist` | slot-free | Words that suppress an intent or must not fill a slot |
| Prompt | `.prompt` | whole-file verbatim | A localized language-model prompt (§4.4) |

The slot-bearing roles map onto the data path of a voice interaction:

- **`.intent`** — *ASR input*: templates matched against the user's speech;
  their named slots are filled by the engine at match time.
- **`.dialog`** — *TTS output*: templates for what the assistant says back;
  their named slots are filled by the caller before the text is spoken.

`.entity`, `.voc`, and `.blacklist` share the slot-free format; they are how a
developer encodes a set of natural-language phrasings for the assistant to use, and
differ only in *which component consumes them* (§4.3).

This specification covers the **folder layout**, the **common parsing rules**,
and the **two file formats** across their **six roles**. It does not cover
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
loader MAY fall back to the nearest available language, as measured by any
language-tag distance metric that treats close regional variants as usable
matches. Any such fallback is an **implementation choice**,
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
is the intent name. Lines in the file **MAY** declare **different sets of named
slots** (OVOS-INTENT-1 §5.5); the intent's slot set is the **union** of the
slots declared across its templates, and the engine extracts only the slots of
the template that matched (OVOS-INTENT-1 §5.5, OVOS-INTENT-3 §5.1). A phrasing
that needs different slots therefore MAY live in the same `.intent` file.

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
`.dialog` file recognizes **both** named-slot forms defined by OVOS-INTENT-1
§3.4 — `{name}` and the equivalent double-brace `{{name}}` — and treats them
identically, since the two forms denote the same slot.

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
phrasings for the assistant to consume.

They differ **only in role** — which component reads the file and what it does
with the expanded phrase set:

| Extension | Role | Consumed by | Pairing |
|-----------|------|-------------|---------|
| `.entity` | Example values that can fill a `{slot}` | Intent engine, as a slot value set (OVOS-INTENT-1 §5.4) | Base name = the `{slot}` name it supplies |
| `.voc` | A named set of localized phrasings | Keyword intent engines; skill runtime helpers; inline `<name>` references in templates (OVOS-INTENT-1 §3.7) | Base name = the vocabulary name |
| `.blacklist` | Words that either suppress an intent or must not fill a slot | Intent engine, as match suppression or slot-value exclusion | Base name = the `.intent` it suppresses, **or** the `.entity` / `{slot}` whose values it excludes |

How an `.entity` or `.voc` phrase set is *used* — slot constraint, keyword
test — is engine or skill policy, consistent with matching behaviour being out
of scope for these specifications.

A `.voc` may additionally be referenced inline from a template by the `<name>`
token (OVOS-INTENT-1 §3.7), which expands it in place; a `.voc` may itself
contain such references.

For the `.blacklist` role the suppression contract is defined: a `.blacklist`
file is paired by base name with exactly one `.intent`, and its expanded phrase
set scopes to that intent alone. A blacklist phrase **occurs** in an utterance
when its words appear there as a **contiguous sequence of whole words** — a
token subsequence, not a raw substring (the phrase `art` does not occur within
the word `start`). If any phrase from the set occurs in the user's utterance,
that intent is suppressed — a **hard, score-independent rejection**, not a
confidence penalty. A `.blacklist` does not affect any other intent.

A `.blacklist` paired by base name with an `.entity` (or with a `{slot}` /
vocabulary of that name) instead defines a **slot-value exclusion**: its phrase
set lists values that **MUST NOT** fill that slot. When a candidate value the
utterance would bind to the slot occurs (same whole-word-sequence rule) in the
exclusion set, the engine **MUST NOT** bind it — the slot is left unresolved, as
though the utterance had not supplied it. The canonical use is preventing
anaphoric pronouns from filling a referential slot: a `person.blacklist` of
`he`, `she`, `they` leaves `{person}` unresolved for *"how tall is he"* so a
later stage — OVOS-CONTEXT-1 §7 context fill, or a re-prompt — supplies the
value. Pronoun sets are language-specific, so this keeps them in per-language
`locale/<lang>/` resources rather than engine code; a `.blacklist` MAY reference
a shared `.voc` inline via the `<name>` token (§4.3).

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
```
# person.blacklist       — values that must not fill the {person} slot
(he|she|they|him|her|them)
it
```

### 4.4 `.prompt` — language-model prompt

**Format.** Prompt: the **whole file**, verbatim, is one prompt. It is plain
text — **not** a template in the OVOS-INTENT-1 grammar — so `(`, `)`, `[`,
`]`, `<`, `>`, `|`, every newline, a single `{` or `}`, and all other
characters are literal. There is no expansion and no line filtering: a
`.prompt` is read whole (§3), `#` lines and blank lines included, because every
character is part of the prompt. The **only** special handling a `.prompt`
receives is the **`{{name}}`** substitution described below; nothing else in
the file is interpreted. In particular there is **no comment handling**: an
HTML-style `<!-- … -->` sequence is ordinary literal text and reaches the
language model unchanged.

**Role.** The localized prompt a skill feeds to a language model. Like every
other resource it is shipped per language under `locale/<lang>/` and resolved
through the override precedence of §2.1, so a prompt can be translated,
adjusted per region, or overridden by a user.

**Substitution.** The one special construct is the **`{{name}}`** substitution
point — the **double-brace** form only. A slot **name** consists of lowercase
ASCII letters, digits, and underscores, and MUST NOT begin with a digit. At
render time a caller supplies values keyed by name; an occurrence of
`{{name}}` is replaced by the caller-supplied value **only when both hold**:

1. it forms a complete, well-formed `{{name}}` — a double-brace pair enclosing
   a valid name;
2. the caller supplied a value for that name.

A **single** `{name}`, a lone `{` or `}`, and literal JSON or markup such as
`{}`, `{ }`, or `{"key": 1}` are **never** substitution points — they pass
through unchanged. This single-brace pass-through is precisely **why** a
`.prompt` requires the double-brace form: prompts routinely contain literal
single braces (JSON examples, code, set notation), and reserving substitution
to `{{name}}` lets that literal text survive untouched. (This is the opposite
convention to `.intent` / `.dialog`, where `{name}` and `{{name}}` are
equivalent per OVOS-INTENT-1 §3.4, because those templates cannot contain a
literal brace at all.)

**Slots are optional.** A `{{name}}` for which the caller supplied no value is
left **as literal text** — an unfilled slot is not an error. This is the
deliberate opposite of `.dialog` (§4.2), where the caller MUST fill every slot
and an unfilled one MUST NOT be rendered. A prompt is free-form text that may
legitimately contain brace sequences the author never intended as slots, so
substitution is conservative: it touches only the `{{name}}` occurrences whose
names the caller explicitly provides.

**Loads as.** The single whole-file string, with substitution applied per the
rules above.

The full content of a file `weather_report.prompt`, rendered with the caller
value `{"query": "weather in Lisbon"}` (a `#` line and an `<!-- … -->`
sequence are both ordinary prompt text here — neither is stripped):

````
# Weather assistant
<!-- author note: keep this terse -->

You are a concise weather assistant. Answer the user's question.

User asked: {{query}}

Reply as JSON shaped like {"summary": "...", "temp_c": 0}. The {response}
single-brace placeholder below is illustrative only:

```
{"summary": "{response}", "temp_c": 18}
```
````

`{{query}}` is substituted; the `# Weather assistant` heading and the
`<!-- … -->` line are kept verbatim, the literal single-brace JSON is left
untouched, and the single-brace `{response}` is not a substitution point and so
is unchanged (whether or not it sits in a code block). A `{{tone}}` slot the
caller passed no value for would likewise stay literal.


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
  the slot fill modes. Five resource roles — `.intent`, `.dialog`,
  `.entity`, `.voc`, `.blacklist` — are lists of templates written in this
  grammar. The `.prompt` role (§4.4) is not a template file; it is verbatim
  text with optional `{{name}}` double-brace substitution only.
