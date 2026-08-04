# Sentence Template Grammar Specification

**Spec ID:** OVOS-INTENT-1 · **Version:** 2 · **Status:** Draft

This document defines the *sentence template* grammar used by example-based
intent engines and by the localized resource files of a skill. A sentence
template is a compact string that describes a set of sentences. The grammar is
implementation-agnostic: any tool, in any programming language, can claim
conformance by satisfying the requirements below.

It serves two roles:

1. The **authoring syntax** skill developers write in `.intent`, `.dialog`,
   `.entity`, `.voc`, and `.blacklist` files (see §1.1, and the companion
   *Locale Resource Formats Specification*, OVOS-INTENT-2).
2. The **wire contract** for training data passed from a skill to an intent
   pipeline plugin (§6).

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are used as in
RFC 2119.

---

## 1. Purpose and scope

A template uses a small set of grammar tokens to express, compactly, many
concrete sentences. The grammar has exactly two facets:

- **Expansion** — `(a|b)` alternatives, `[x]` optionals, and `<name>`
  vocabulary references — which make one template stand for many variant
  sentences (§3.2–§3.3, §3.7, §4).
- **Named slots** — `{name}` — placeholders that are *filled* with a value
  rather than written out (§3.4, §5).

A given file type uses one or both facets. This specification defines the
tokens, the expansion algorithm, the slot model, and the skill→pipeline
training-data contract. It does **not** cover matching, generalization, scoring,
confidence, or how an engine ranks competing intents — those are
engine-specific.

This specification is deliberately **unopinionated about slot value types** — see §5.3.

### 1.1 Where this grammar is used

The grammar is carried by **five** skill resource roles (OVOS-INTENT-2). They
differ along three axes: whether they use **expansion**, whether they use
**named slots**, and the **direction** of the text — *input* text matched
against speech recognition (ASR) output, or *output* text rendered to
text-to-speech (TTS).

| File | Expansion `(a\|b)` `[x]` | Named slots `{name}` | Direction | How slots are filled |
|------|:---:|:---:|---------|----------------------|
| `.intent` | yes | yes | input (vs ASR) | by the engine, at match time, from the utterance |
| `.dialog` | yes | yes | output (to TTS) | by the caller, before rendering |
| `.entity` | yes | no | input | — |
| `.voc` | yes | no | input | — |
| `.blacklist` | yes | no | input | — |

`.entity`, `.voc`, and `.blacklist` are grammatically **identical** — expansion
only, no named slots. They are the same file format and differ only in the role
OVOS-INTENT-2 assigns each: three ways a developer encodes a set of
natural-language phrasings for different OVOS components to consume.

---

## 2. Input model

Templates whose **direction is input** — `.intent`, `.entity`, `.voc`,
`.blacklist` — are matched against **ASR (speech recognition) transcription
output**. This is not a general-purpose text grammar, and the design relies on
the following input contract.

The text presented to an engine — both training samples and utterances — MUST,
by the time it reaches the engine, be **normalized** to:

- **lowercase** characters only;
- **alphanumeric word tokens** separated by **single spaces**;
- **no punctuation** and **no bracket characters** (`( ) [ ] { } | < >`).

Normalization (lowercasing, punctuation and apostrophe stripping, whitespace
collapsing, locale-specific transliteration) is performed **upstream** of the
intent engine and is **out of scope** for this grammar; it belongs to a
separate text-normalization specification. An engine MAY assume its input
already satisfies the contract.

Two consequences follow:

- **Input-direction templates MUST be authored in the same normalized form**:
  literal words are lowercase, alphanumeric, single-space separated.
- The grammar metacharacters `( ) [ ] { } | < >` **cannot occur as literal input**.
  They are therefore exclusively structural, and **no escape mechanism is
  needed or provided**. This is a deliberate consequence of the voice-input
  scope.

The input model does **not** apply to **output-direction** templates
(`.dialog`): those are spoken text and MAY contain mixed case and punctuation
(§5.2).

---

## 3. Grammar tokens

A template is literal text interspersed with the tokens below.

| Element | Syntax | Facet | Meaning |
|---------|--------|-------|---------|
| Literal word | `word` | — | Matched or spoken verbatim. |
| Alternatives | `(a\|b\|c)` | expansion | A choice of branches (§3.2). |
| Optional | `[x]` | expansion | An optional segment; equivalent to `(x\|)`. |
| Named slot | `{name}` or `{{name}}` | slot | A placeholder filled with a value; the two forms are equivalent (§3.4, §5). |
| Vocabulary reference | `<name>` | expansion | Expands to a named vocabulary (§3.7). |

There is no slot-typing syntax, no digit token, and no legacy wildcard. A
named slot has two equivalent spellings, `{name}` and `{{name}}` (§3.4); the
double-brace spelling is a slot, not a brace-escaping form — the grammar
provides no way to write a literal brace, and none is needed (§2).

### 3.1 Literal words

Any run of characters that is not a grammar token is literal text.

```
turn on the lights
```

### 3.2 Alternatives `( | )`

Parentheses enclose **branches** separated by the pipe `|`. Each combination
takes exactly one branch from each group. A group SHOULD contain at least one
`|` (that is, at least two branches); a single-branch group is degenerate —
loaders accept it, warn, and fold it to the bare branch (§3.6) — and the
empty `()` is malformed (§3.6).

```
(turn on|switch on|enable) the lights
```

A branch MAY be empty. An empty branch contributes nothing; any double space
this would otherwise leave is removed by whitespace normalization (§4.1).

```
(please|) turn on the lights
```

### 3.3 Optional segments `[ ]`

Square brackets mark an optional segment. `[x]` is **exactly equivalent** to the
alternative group `(x|)`: one variant includes `x`, one omits it.

```
turn on [the] lights
```

### 3.4 Named slots `{ }`

A curly-brace token is a **named slot** — a placeholder that is not written out
but *filled* with a value. The same slot syntax is used everywhere a slot
appears; only *who fills it and when* differs by file type (§5.1).

A named slot MAY be written in **either** of two equivalent forms:

- single-brace — `{name}`;
- double-brace — `{{name}}`.

The two forms are **exactly equivalent**: a conformant tool folds `{{name}}`
to `{name}` and treats them identically thereafter. They denote the same slot,
fill the same way, and obey the same rules; there is no behavioural difference
between them. The double-brace form is a slot, **not** an escape: `{{` … `}}`
never produces a literal brace, and the grammar provides no brace-escaping form
(§2 makes brace characters impossible as literal input, so none is needed).

A slot **name** — the text inside the braces in either form — MUST consist only
of lowercase ASCII letters, digits, and underscores (`a`–`z`, `0`–`9`, `_`),
MUST NOT begin with a digit, and MUST NOT contain whitespace inside the braces.
These rules apply identically to `{name}` and `{{name}}`. A slot MAY appear
anywhere a literal word may, including inside an alternative or optional group:

```
(buy|sell) {item}
it is currently {temperature} degrees
(buy|sell) {{item}}
```

Slots are used only by `.intent` and `.dialog` files (§1.1).

### 3.5 Nesting

Expansion groups MAY be nested without limit. An optional group may contain
alternatives, and a branch of an alternative may contain optional segments:

```
turn on [(all|every) ]light[s]
```

### 3.6 Malformed forms

The following forms are **malformed**; a tool MUST reject any template that
contains one:

- **Unbalanced metacharacters** — an unmatched `(`, `)`, `[`, `]`, `{`, `}`,
  `<`, or `>`.
- **Empty group** — the empty `()`. A group expresses a *choice between
  branches*; with no branch there is nothing to choose.
- **Empty sample** — a template whose sample set (§4) contains the empty
  string: for some combination of branches it yields a sample with no literal
  words and no slots. The simplest cases are a template consisting only of
  `(|)` or `[x]`. An engine cannot train on an empty sample. This concerns the
  *whole sample only*: a group with an empty branch inside an otherwise
  non-empty template — such as the optional `[the]` — is valid and unaffected.
- **Slot-only template** — a template that is a single named slot and nothing
  else (`{name}`). A template MUST carry at least one literal word; a bare slot
  gives an engine no anchoring text to learn from or match against.
- **Adjacent slots** — two named slots with no literal word between them,
  whether written `{a}{b}` or separated only by whitespace (`{a} {b}`). With no
  literal token to delimit them, a matcher cannot tell where one slot's value
  ends and the next begins; the two would form a single capture, not two. A
  literal word MUST separate any two slots, and MUST do so in **every sample**:
  the check applies to the expanded sample set (§4), not only the template
  surface. A template such as `{a} [foo] {b}`, whose empty-`foo` branch yields
  the adjacent pair `{a} {b}`, is therefore malformed.
- **Repeated slot name** — using the same `{name}` more than once in one
  template (`{x} and {x}`). A template defines each slot name exactly once.
- **Undefined vocabulary reference** — a `<name>` (§3.7) for which no
  vocabulary `name` is available to the expander.
- **Cyclic vocabulary reference** — a chain of inline vocabulary references
  that includes itself; its resolution would not terminate.

A **single-branch group** — a parenthesised group with no `|`, e.g.
`(word)` — is degenerate but **not** malformed: loaders MUST accept it,
SHOULD warn, and MUST treat it as exactly the bare branch (`(word)` ≡
`word`). The folding is semantically lossless — the template denotes the
same sample set with or without the parentheses — and a construct with an
unambiguous meaning is a stylistic slip, not an error; the warning is the
author's cue to write the branch as plain literal text.

**Structural adjacent-slot detection.** Because the adjacent-slots check is
defined over the expanded sample set, a naive implementation must expand the
full template first — potentially expensive (§4.3). A tool MAY instead
detect adjacency **structurally**, pre-expansion: two slots are adjacent iff
every token between them on the template surface can reduce to nothing (an
optional segment, a group with an empty branch, or an inline reference
resolving to such). A structural check that accepts and rejects exactly the
same templates as the full-expansion check is a conformant alternative.

Empty lines and `#`-comment lines are removed by the file reader before a
template reaches the grammar (OVOS-INTENT-2 §3); they are not part of a
template.

### 3.7 Inline vocabulary reference `< >`

An angle-bracket token `<name>` is an **inline vocabulary reference**. During
expansion it is replaced by a named **vocabulary** — a slot-free phrase set,
supplied as a `.voc` resource (OVOS-INTENT-2) or as inline data. The reference
expands to a choice over the vocabulary's members, exactly as if those members
had been written as an alternative group in its place.

`name` obeys the same charset as a slot name (§3.4): lowercase ASCII letters,
digits, and underscores, and MUST NOT begin with a digit.

A vocabulary is itself written in this grammar (slot-free), so a referenced
vocabulary MAY contain further `<…>` references; resolution recurses (§4.1). A
`<name>` reference MAY appear anywhere a literal word may, including inside an
alternative or optional group; it MUST NOT appear inside a named slot, and it
never introduces one — a vocabulary is slot-free.

An inline vocabulary reference is purely an **authoring** convenience: the
expander resolves it before producing the sample set (§4.1), so a `<name>`
token never reaches an intent engine. Resolving a reference requires the
referenced vocabulary; an expander is given the vocabularies alongside the
template.

```
<greeting> [there] {name}
```

Given a vocabulary `greeting` whose members are `hello`, `hi`, and
`good morning`, this template is equivalent to:

```
(hello|hi|good morning) [there] {name}
```

---

## 4. Expansion

A template **expands to a sample set**: a finite set of **sample sentences**. A
sample sentence is a finite sequence of **terms**, where each term is either a
**literal word** or a **named slot**.

Expansion resolves only the `(a|b)` / `[x]` facet. It defines the **shape of the
training data** a template contributes — and nothing more. It does **not**
define which utterances an intent engine accepts at match time. A template is a
*generator of sample sentences*, not a matcher.

An engine consumes the expanded sample set as training data. A capable engine is
expected to **generalize beyond it**: it should recognize utterances that are
not literally present in the sample set — different word order, synonyms, filler
words, partial phrasings — and it may decline some utterances that are.
Generalization, scoring, and the accept/reject decision at match time are
engine-specific and deliberately outside the scope of this specification.

What this specification *does* mandate is that every conformant tool expand a
given template to exactly the same sample set, so that data is portable and
reproducible across tools.

### 4.1 Reference enumeration

The sample set is obtained by:

1. **Resolve inline vocabulary references.** Replace each `<name>` (§3.7) with
   the alternative group `(m₁|m₂|…|mₖ)`, where `m₁…mₖ` are the members of the
   referenced vocabulary's sample set. A referenced vocabulary is itself a
   template set expanded by this same procedure, so a vocabulary may contain
   further `<…>` references; resolution recurses. A reference to an unavailable
   vocabulary, or a reference cycle, is malformed (§3.6).
2. Replace every `[x]` with `(x|)`. The **working set** is the single resulting
   string.
3. While any string in the working set still contains `(`: for each such
   string, locate its **innermost** groups — each a `(...)` containing no
   nested parentheses — split each group's interior on `|` into branches (a
   branch may be empty), and replace that string with the **Cartesian product**
   of substituting each branch for each of its groups. The working set becomes
   the union of all strings so produced.
4. **Normalize whitespace** in each string: replace every run of one or more
   spaces with a single space, and strip leading and trailing spaces.
5. Remove duplicates. The remaining distinct strings are the sample set.

Named slots `{...}` are opaque throughout: they are carried through unchanged
and are **never** expanded.

A template whose sample set contains the empty string is **malformed** (§3.6);
an engine cannot train on an empty sample.

### 4.2 Worked example

```
(turn|switch) [the] (light|fan)
```

After replacing `[the]` with `(the|)`, three groups of 2 branches each give
`2 × 2 × 2 = 8` combinations. When the empty `the` branch is taken, whitespace
normalization (step 4) collapses the resulting double space. The sample set is:

```
switch fan          turn fan
switch light        turn light
switch the fan      turn the fan
switch the light    turn the light
```

### 4.3 Sample-set size and limits

The sample set size is the product of the branch counts of all groups —
`kⁿ` for `n` groups of `k` branches. A large sample set inflates data volume and
training time.

- Authors **SHOULD** keep templates focused, split unrelated phrasings into
  separate templates rather than nesting many large groups, and rely on an
  engine's generalization (§4) rather than enumerating every phrasing.
- An expander **MAY** refuse a template whose expansion exceeds an
  implementation-defined sample-count limit, and **SHOULD** document that limit.

---

## 5. Named slots

A named slot is a placeholder filled with a **value**. What a slot *means* is
defined here; how a matcher delimits a slot's span in an utterance is
engine-specific.

### 5.1 Slot filling

Every slot is filled by a **filler**. The grammar defines two fill modes; which
applies is determined by the file type the slot appears in (§1.1):

- **Match-time fill — `.intent`.** The slot is filled by the **intent engine**
  while matching the user's utterance: the engine captures the corresponding
  span of speech and returns it keyed by the slot name. This is the input
  direction.

- **Caller-supplied fill — `.dialog`.** The slot is filled by the **caller**
  (the skill) *before* the text is rendered to TTS. The caller MUST supply a
  value for **every** slot in the chosen phrase; a phrase with an unfilled slot
  MUST NOT be sent to TTS. This is the output direction, and is the normal way
  to inject **dynamic values** into spoken responses — a weather reading, the
  current time, a computed result.

A slot's `{name}` is identical in both modes; only the filler differs.

A slot MAY appear inside an optional or alternative group (§3.4), making it an
**optional slot**. In a sample where that group's slot-free branch is taken the
slot is simply absent: under match-time fill the engine returns no value for
that slot name; under caller-supplied fill no value is needed, because the slot
is not part of the chosen phrase.

### 5.2 Slot values

A slot value is a **sequence of one or more words**, returned as text. Under
match-time fill the engine captures a span of the utterance; under
caller-supplied fill the value is whatever string the caller provides.

### 5.3 Slot value types — deliberately unspecified

This specification does **not** define slot *value types* (numbers, dates,
durations, enumerations) and does **not** define any coercion of a slot value.
A slot value is an opaque sequence of words, as in §5.2.

Interpreting a slot value as a typed datum is inseparable from **text
normalization** of ASR output — for example, whether a spoken `"forty two"`
should become the integer `42` depends entirely on how numerals are normalized
upstream, which this grammar does not prescribe (§2). Specifying typing
without first specifying normalization would be incoherent.

Slot value types are therefore **out of scope** for this specification, as is
the normalization they depend on (§2). There is exactly one slot form,
`{name}`, with no `{name:type}` variant.

### 5.4 Value sets

A skill MAY supply a set of example values for a named slot through an `.entity`
file (OVOS-INTENT-2 §4.3) — a file named after the slot it supplies — whose
lines are expansion-only templates (§1.1). An engine
MAY use that set to constrain or score a match-time-filled slot. A value set is
an **optional refinement**: a slot with no `.entity` file still fills per §5.2,
and a slot referencing an undefined value set is **not** an error.

### 5.5 Slot consistency across a definition

A `.dialog` file — and equivalently any set of inline caller-supplied-fill
phrases treated by a dialog renderer as one definition — defines **one**
dialog. This specification does not define how such a definition is
delivered to a renderer; that is a renderer-specific or host-application
concern. Every template in
that definition MUST declare the **identical set of slot names**. A `.dialog`
definition MUST NOT mix templates that declare different slots, and MUST NOT mix
slot-bearing templates with slot-free ones.

A template *declares* a slot name if that name appears anywhere in the
template. Optionality does not change this: a slot inside an optional group
(`[{x}]`, §5.1) is still declared, so a template `say [{x}]` and a template
`say {x}` declare the **same** slot set and may coexist in one definition.

This guarantees that a dialog's required fill values are the same regardless
of which phrase is chosen. If two phrasings genuinely need different slots,
they are two different dialogs: place them in **separate files** and render
them individually.

A `.intent` file does **not** impose this constraint. A template intent (§6.1)
is a collection of training samples; the engine matches against individual
templates and extracts only the slots declared by the template that best
matches. Templates in one `.intent` file MAY declare **different slot sets**;
the union of all declared slot names is the intent's available slot set. A
tool MUST NOT reject a `.intent` definition because its templates declare
different slots.

A tool MUST reject a `.dialog` definition whose templates do not all declare
the same slot set.

---

## 6. Training-data contract

This grammar is the contract for handing intent and entity training data from a
skill to an intent pipeline plugin. This section covers only the
**input-direction, engine-trained** resource roles — `.intent`, `.entity`,
`.voc`, and `.blacklist` (§1.1). `.dialog` is output-direction and
caller-filled (§5.1); no intent engine trains on it, so it is out of scope
here. The obligations a `.dialog` consumer takes on are defined in §7's
Dialog renderer role instead.

### 6.1 Delivery

A skill registers an intent or an entity by providing **either**:

- a list of **inline samples** — strings, each a template; **or**
- a **path** to an `.intent` / `.entity` resource file.

A registration carries at least a **name** unique within the registering skill,
a **language** code, and the samples or file path. Vocabulary (`.voc`) and blacklist (`.blacklist`) data,
where an engine consumes it, are delivered in the same shape — a name, a
language, and inline samples or a file path.

### 6.2 Engine obligations

On receiving training data a conformant engine **MUST**:

1. Read the file or take the inline samples.
2. Verify the templates conform to §2–§3 (normalized form, valid tokens).
3. For `.intent` training data, accept templates with differing slot sets
   (§5.5); `.entity`, `.voc`, and `.blacklist` data carry no named slots and
   are unaffected by §5.5.
4. Expand each template to its sample set per §4.
5. Use the resulting samples as training data; for `.intent` data, treat
   `{...}` slots as match-time-filled (§5.1–§5.2). How the engine learns from
   and generalizes beyond those samples is its own concern (§4).

---

## 7. Conformance

This specification is implemented by tools in three roles; a single tool MAY
fill more than one role. Conformance constrains how a template is *parsed,
expanded, and filled* — never how an engine *matches*.

- **Expander.** A tool that turns a template into its sample set. It MUST accept
  the token set of §3, resolve inline vocabulary references (§3.7, §4.1 step 1),
  reject the malformed forms of §3.6, produce exactly the sample set defined by
  §4, and never expand `{...}` slots.

- **Intent engine.** A tool that consumes **slot-bearing** input templates
  (`.intent`). It MUST embed a conformant expander, assume the input model of
  §2, honour the training-data contract of §6, treat `{name}` as a
  match-time-filled slot (§5.1–§5.2), and treat value sets as optional
  refinements (§5.4). Matching, generalization, and scoring are deliberately
  unconstrained — an engine MAY add fuzzy matching, neural classification, or
  any scoring strategy. A tool that consumes only **slot-free** input resources
  (`.voc`, `.entity`, `.blacklist`) — for example a keyword-based engine — does
  not take on this role; it needs only the **Expander** role above.

- **Dialog renderer.** A tool that consumes `.dialog` templates. It MUST read
  the `.dialog` file or take the inline phrases, embed a conformant expander,
  verify that all phrases in a dialog definition declare the same slot set
  (§5.5), fill `{name}` slots by caller-supplied values before rendering, and
  MUST NOT emit a phrase containing an unfilled slot (§5.1). Unlike the
  Intent engine role, no slot in a `.dialog` template is ever filled at
  match time (§5.1) — dialog training data is out of §6's scope.

No tool may change the meaning of the tokens defined here.

---

## See also

- *Locale Resource Formats Specification* (OVOS-INTENT-2) — the locale folder
  layout and the six resource roles, five of which — `.intent`, `.entity`,
  `.voc`, `.dialog`, `.blacklist` — carry templates written in this grammar
  (§1.1).
