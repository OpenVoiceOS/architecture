# Intent Definition Specification

**Spec ID:** OVOS-INTENT-3 · **Version:** 1 · **Status:** Draft

This document defines what an **intent** is, the two methods a developer uses
to define one, how an intent is registered, and the input contract an **intent
engine** consumes. It is implementation-agnostic: any voice assistant, in any
language, can adopt this model.

It builds on the two companion specifications:

- the *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the template
  grammar used by template intents and by vocabularies;
- the *Locale Resource Formats Specification* (OVOS-INTENT-2) — the `.intent`,
  `.voc`, and `.entity` resource files an intent is defined from.

The specifications are numbered in dependency order: read OVOS-INTENT-1 first,
then OVOS-INTENT-2, then this document. Each builds on the ones before it, and
this one assumes both.

The key words **MUST**, **MUST NOT**, **SHOULD**, and **MAY** are used as in
RFC 2119.

---

## 1. What an intent is

An **intent** is a developer-defined, named binding from a **natural-language
command** to a specific **piece of code** — a **handler** — in the skill that
defines it. It expresses one thing: *"this is how to trigger me, this code, by
voice."*

Three invariants hold for every intent:

- **One owner.** An intent is owned by exactly **one skill** — the skill that
  defines it. No other component defines, redefines, or claims it.
- **One handler.** An intent is bound to exactly **one handler**: the code that
  runs when the command is recognized.
- **One unit.** The natural-language definition and the handler are **registered
  together**, as a single unit, by the owning skill (§6). One does not exist
  without the other. An intent MAY carry one such unit per definition method
  (§2) — a dual-method intent is two units bound to the same handler, not an
  exception to this invariant.

An intent is therefore **not an event**. It is not a message that any component
may emit, and not a topic that any component may subscribe to. It is not routed
by name to whoever happens to listen. An intent is inseparable from the code
that defines it; recognizing the command exists only to run that code.

### 1.1 Scope

This specification defines the intent concept, the two definition methods (§2),
the skill and intent identity model (§3), keyword intents (§4), template intents
(§5), registration and the intent-engine input contract (§6), the match result
(§7), and conformance (§8).

It does **not** define how an engine *matches* an utterance — searching,
scoring, confidence, fuzzy matching, and ranking competing intents are
engine-specific, as in OVOS-INTENT-1. It does **not** prescribe any
decorator or language-level syntax for binding a handler; that is an
implementation concern (§6).

---

## 2. The two definition methods

An intent is defined by one or both of two methods:

- a **keyword intent** — defined by keyword constraints over vocabularies (§4);
- a **template intent** — defined by sentence templates (§5).

| Method | Defined by | Resources (OVOS-INTENT-2) | Trigger style |
|--------|-----------|---------------------------|---------------|
| Keyword intent | Required / optional / one-of / excluded keyword constraints | `.voc` | Presence of keywords, word order free |
| Template intent | Example sentence templates with named slots | `.intent`, `.entity`, `.blacklist` | Whole phrasings; engine generalizes |

The two methods are **not interoperable**. A keyword intent definition cannot
be consumed by an engine that implements only template intents, and a template
intent definition cannot be consumed by an engine that implements only keyword
intents. There is no automatic conversion between them.

The two methods are **complementary**. They describe a command in fundamentally
different shapes — a keyword intent by *which words must be present*, a template
intent by *what whole sentences sound like* — and each suits triggers the other
does not. A skill **MAY** define some of its intents as keyword intents and
others as template intents. Each **individual** registration uses exactly one
method; the two definition shapes are never mixed within one registration. An
intent **MAY** carry one registration per method — two training-data
representations of the same handler (OVOS-INTENT-4 §3.2).

A developer chooses the method **per intent**, by what best describes that
command, and by which engines the target assistant runs (§6).

---

## 3. Skill and intent identity

Every **skill** — an *app*, a self-contained unit of assistant functionality —
has a **skill id**: an identifier **unique across the assistant**. Every
**intent** has an **intent name**: an identifier **unique within its owning
skill**. The skill id is part of an intent's definition — an intent is never
defined without the skill it belongs to.

Together the two give every intent a **globally unique qualified name**,
written `skill_id:intent_name`. This two-part name namespaces intent names
cleanly: two skills may each define an intent named `play`, and the qualified
names `music.skill:play` and `video.skill:play` keep them distinct. Neither a
skill id nor an intent name contains a `:`, so the qualified name always parses
unambiguously into its two parts.

When an intent definition reaches an intent engine it is identified by this
qualified name; the qualified form is also the **label** an engine's classifier
assigns (§6.2) and the intent name carried in the match result (§7). How an
engine forms or exploits the qualified name internally — for example,
hierarchical or domain-grouped matching strategies keyed on the `skill_id`
portion — is an implementation detail. What this specification requires is only
that both parts exist, that the skill id is unique and is part of the
definition, and that the intent name is unique within its skill.

Regardless of method, every intent therefore carries:

- a **skill id** — the unique identifier of the owning skill;
- an **intent name** — unique within that skill;
- a **language** — a BCP-47 tag (OVOS-INTENT-2 §2). An intent is defined
  **per language**: the same intent in two languages is two definitions that
  share a qualified name and a handler;
- a **definition** — the keyword constraints (§4) or the templates (§5);
- a **handler** — the code the intent triggers (§1, §6).

An intent definition is therefore identified by the quadruple
`(skill id, intent name, language, method)`. The handler is shared across an
intent's languages and across its methods; the definition is not. A
dual-method intent (§2) is **two** definitions — one keyword, one template —
sharing the first three components and differing only in `method`; both
still bind the same handler (§6.1).

---

## 4. Keyword intents

A **keyword intent** is defined by **keyword constraints** over
**vocabularies**.

### 4.1 Vocabularies

A **vocabulary** is a named set of natural-language phrasings. It is supplied
either as an OVOS-INTENT-2 `.voc` resource file or as an inline list of
phrasings — the same file-or-inline choice the training-data contract offers
(OVOS-INTENT-1 §6.1). Each entry is a **slot-free** sentence template
(OVOS-INTENT-1 §1.1); the vocabulary is the union of the entries' expanded
sample sets.

A vocabulary **phrasing occurs** in an utterance when its words appear there as
a contiguous sequence of whole words (the same notion of occurrence as
OVOS-INTENT-2 §4.3). A vocabulary **occurs** in an utterance when at least one
of its phrasings does.

Within a single phrasing the words must occur **contiguously**; **between**
vocabularies there is no order requirement. A keyword intent constrains *which*
vocabularies occur, not where in the utterance or in what order — this is what
makes it word-order free.

### 4.2 Constraint roles

A keyword intent lists vocabularies under four **constraint roles**:

| Role | Meaning |
|------|---------|
| **required** | Every required vocabulary MUST occur in the utterance. |
| **optional** | Captured if it occurs; its absence does not prevent a match. |
| **one-of** | A group of vocabularies; **at least one** member MUST occur. An intent MAY declare several independent one-of groups. |
| **excluded** | If any excluded vocabulary occurs, the intent MUST NOT match. |

These roles are the **definition** of the keyword intent. A conformant engine
**MUST NOT** report a keyword intent as matched when any of them is violated:

- a required vocabulary does not occur; or
- some one-of group has no member occurring; or
- an excluded vocabulary occurs.

Whether an utterance that *satisfies* all constraints is reported, and how it
ranks against other satisfiable intents, is engine-specific (§1.1). The
constraint roles fix only what the intent *requires*, never how an engine
searches or scores.

A keyword intent **MUST** declare at least one **required** or **one-of**
constraint: an intent with only optional and excluded constraints has nothing
that must be present and is malformed.

A vocabulary **MUST** appear under at most one role within a single intent.
Listing the same vocabulary under two roles — for example as both required and
excluded — is contradictory and malformed.

The `excluded` role is a keyword intent's built-in **suppression** mechanism.
A keyword intent therefore needs no separate `.blacklist` artifact for
per-intent suppression and does not use one for that role; `.blacklist` is the
template-intent counterpart (§5.5). This is distinct from the entity-level
exclusion role a `.blacklist` may also carry (§5.5), which applies regardless
of intent kind.

### 4.3 Captured values — each vocabulary is a slot

When a keyword intent matches, every vocabulary it lists as **required**,
**optional**, or **one-of** that **occurred** in the utterance yields a
**captured value**: the phrase that occurred, keyed by the **vocabulary name**.

Each such vocabulary therefore **doubles as a slot** — the vocabulary name is
the slot key, the phrase that occurred is its value. This is the keyword-intent
counterpart of a template intent's named slot (§5.2): the constraint roles of
§4.2 decide *whether* the intent matches, and the vocabularies that occurred
become the *slots* in the result (§7).

**Excluded** vocabularies are never captured; their only role is to block a
match. An optional vocabulary that did not occur contributes no slot. If more
than one of a vocabulary's phrasings occurs in the utterance, which one becomes
the captured value is engine-specific.

### 4.4 No regular expressions

A keyword intent is defined **purely by vocabularies**. Regular-expression
entity extraction is deliberately **not** part of this specification, and
defining an intent with regular expressions is **recommended against**:

- An intent definition is not the right place for a regular expression. When a
  command carries free-form text that vocabularies cannot enumerate, that text
  is a **slot** — define the intent as a template intent (§5) and let the
  engine's slot extractor capture it.
- Regular expressions are notoriously hard to localize. A pattern written for
  one language rarely transfers to another, which conflicts with the
  per-language definition model (§3) and with the i18n goals of OVOS-INTENT-2.

This is consistent with OVOS-INTENT-2, which defines no regex resource role.

`.entity` value-set hints (§5.2) apply only to template intents. A keyword
intent takes no `.entity` hints; its vocabularies are its only inputs.

### 4.5 Example

A keyword intent `set_brightness`, in prose:

- **required:** `set` vocabulary (`set`, `change`, `adjust`), `brightness`
  vocabulary (`brightness`, `light level`);
- **one-of:** the group { `up` vocabulary, `down` vocabulary };
- **excluded:** `question` vocabulary (`what is`, `how`).

`change the brightness up` matches: `set`, `brightness`, and one member of the
one-of group all occur, and no excluded vocabulary occurs. `what is the
brightness` does not: the `question` vocabulary occurs.

---

## 5. Template intents

A **template intent** is defined by **sentence templates** — example phrasings
of the command, written in the grammar of OVOS-INTENT-1, with named slots
`{name}` for the parts that vary.

### 5.1 Definition

The templates are delivered exactly as the OVOS-INTENT-1 training-data contract
specifies (§6.1): either as inline samples or as a path to a `.intent` resource
file (OVOS-INTENT-2 §4.1). Templates in one intent MAY declare **different
sets of named slots**; the engine extracts only the slots declared by the
template that best matches (OVOS-INTENT-1 §5.5).

Unlike a keyword intent, a template intent is **not** a presence test. The
templates are *training data*: a capable engine generalizes beyond them and
recognizes unseen phrasings — different word order, synonyms, filler words
(OVOS-INTENT-1 §4). The templates describe what the command *sounds like*, not
an exhaustive list of accepted sentences.

**The `#` character has no meaning inside a template (non-normative).** A
leading `#` marks a comment line in a resource file, and the file reader
removes such lines before templates are parsed (OVOS-INTENT-1). Beyond
that, this document defines no role for `#`, and none should be assumed. One legacy engine historically treated an inline
`#` as a wildcard for a single digit character; templates must not rely on
that behavior. A value that varies belongs in a `{slot}`, and interpreting the
value it captures — for example, a number that speech recognition may deliver
as digits or as words — is the consumer's concern, not template syntax.

### 5.2 Captured values

The captured values of a template intent are its **named slots**, filled by the
engine at match time (OVOS-INTENT-1 §5.1), each keyed by its slot name.

A skill **MAY** supply an `.entity` value set for a slot (OVOS-INTENT-1 §5.4).
An `.entity` file is an **optional hint**, not part of the intent definition: it
gives the engine example values a slot is expected to take. Such hints are
**strongly recommended** — they help an engine resolve a slot's span and its
meaning, and improve classification — but an intent is fully defined without
them, and a slot with no `.entity` file still fills normally. How an engine uses
a hint is engine-specific: it may augment its training data with the values,
ignore them, or use them to score a candidate match — but it MUST NOT treat
them as a closed set: a slot filled with a value outside the set MUST remain
matchable (OVOS-INTENT-1 §5.4). A value set is always a refinement, never a
requirement.

### 5.3 Required slots

A template intent MAY declare **required slots**: a list of slot names that the
engine **MUST** extract for a match to be valid. If any required slot is
absent from the match result, the engine **MUST** treat the intent as
unmatched for that utterance — the intent does not fire. The engine **MAY**
internally consider other templates of the same intent before concluding
that the intent is unmatched.

Required slots are an **optional** opt-in guarantee for the handler. When
present, the handler MAY rely on those slots being populated; when absent,
the handler must defend against missing slots per §7.1.

A required slot MUST be declared by at least one template in the intent.
Declaring a required slot that no template mentions is malformed: the intent
can never match, and a tool MUST reject the definition at registration time.

Required slots do not change how templates are authored. A template that does
not declare a required slot is still valid training data; the engine simply
cannot return a match from that template unless the required slot is also
present (for example, through a separate template that does declare it).

### 5.4 Example

A template intent `play_music`, defined by the templates:

```
(play|put on) {query}
(play|put on) {query} (on|using) {engine}
i want to listen to {query}
```

`play some jazz` is recognized and fills `{query}` with `some jazz`. `put on
the beatles using spotify` fills `{query}` and `{engine}`. A phrasing not among
the templates, such as `could you play something relaxing`, is still expected
to match — the engine generalizes.

### 5.5 Suppression — the `.blacklist`

The OVOS-INTENT-1 template grammar is purely **generative**: a template
describes utterances the intent *should* match and has no way to express
utterances it should *not*. A template intent therefore expresses exclusion
through a separate **`.blacklist`** (OVOS-INTENT-2 §4.3): a slot-free phrase
set, paired with the intent, whose occurrence in an utterance **suppresses**
the intent — a hard, score-independent rejection.

A `.blacklist` is optional and applies **only to template intents**. It is the
template-intent counterpart of a keyword intent's `excluded` constraint role
(§4.2): a keyword intent expresses exclusion inside its definition, a template
intent through this paired artifact. When present, a `.blacklist` is delivered
with the intent's templates as part of the same registration (§6.1).

This per-intent suppression role is a separate mechanism from the
entity-level slot-value-exclusion role a `.blacklist` file may also carry
(OVOS-INTENT-2 §4.3), which is instead scoped to an entity registration
(OVOS-INTENT-4 §7.1) and applies independently of any intent's kind.

---

## 6. Registration and the intent-engine input contract

### 6.1 Registration

An intent enters the system through a **registration**: the owning skill
submits the intent's **definition and its handler together**, as one unit. The
binding between them is made at this moment and is not separable afterwards.

A registration carries:

- the **skill id** and the **intent name** (§3), which together form the
  qualified name the engine identifies the intent by;
- the **language** (§3);
- the **definition** — keyword constraints (§4), or templates and any paired
  `.blacklist` (§5);
- a **reference to the handler** — whatever the orchestrator needs to invoke the
  handler later. Its concrete form (an in-process callback, a method, an
  address) is implementation-specific and **not** prescribed here.

The definition is passed to an intent engine (§6.2). The handler reference is
retained by the **orchestrator** — the intent system that owns the engines — and is
invoked with the match result (§7) when the engine reports a match. An engine
never invokes a handler directly; it reports matches, and the orchestrator routes.

Registering an intent whose `(skill id, intent name, language, method)`
quadruple (§2, §3) matches an existing registration **replaces** it: the
previous definition and handler binding for that method are discarded and
wholly superseded by the new one. A registration under the other method for
the same `(skill id, intent name, language)` triple is untouched
(OVOS-INTENT-4 §3.2).

**Deregistration** ("detach") removes an intent: its definition is withdrawn
from the engine and its handler reference is released by the orchestrator. Definition
and handler are removed **together**, as they were registered. When a skill is
removed, all of its intents are deregistered.

### 6.2 The intent engine

An **intent engine** is a tool that consumes intent definitions and, given an
utterance, identifies which registered intent it triggers. In this architecture
an intent engine is realized as an intent pipeline plugin (OVOS-PIPELINE-1).

Conceptually, every intent engine is a **classifier paired with a slot
extractor**. Given an utterance — ASR-normalized text, as in OVOS-INTENT-1 §2 —
it produces a **label** (the qualified intent name `skill_id:intent_name`, §3)
and a set of **extracted slot values**. The two definition methods differ only
in how the engine is told what to classify and what to extract; the output is
the same shape either way (§7). This is why the result is uniform even though
keyword and template definitions are not interoperable.

An intent engine **declares which definition method(s) it accepts** — keyword,
template, or both. The contract is:

- Every conformant intent engine **MUST** accept **at least one** of the
  two methods.
- An engine **MAY** accept both.
- An engine **MUST reject** a registration whose method it does not accept,
  rather than silently ignoring it, so the orchestrator can route the intent to an
  engine that does accept it.

For a given utterance an engine reports **at most one** matched intent. An
engine **MAY** find that no registered intent matches; it then reports no
result and no handler runs. Producing a result is not guaranteed — only that,
when one is produced, it has the shape of §7.

Because the two methods are not interoperable (§2), an intent functions only
when the assistant runs an engine that accepts that intent's method. If no
loaded engine accepts an intent's method, that intent cannot be triggered. A
skill author **SHOULD** therefore choose the method per the engines the target
assistant is expected to run.

**Which engines are loaded is a deployment matter (non-normative).** This
specification defines the shape and contract of an intent and its engine
interface; it does not govern how an installation is assembled. Which intent
engines are loaded is chosen by the deployer. A conformant deployment is
expected to provide at least one engine of each kind — one accepting keyword
intents and one accepting template intents — but the platform is open: an
installation may load whatever engines it likes, including an LLM- or
chatbot-based engine in place of the usual ones. Because of this, a skill
developer is advised to document which definition method (or specific engine)
the skill relies on, so deployers can ensure a compatible engine is loaded.
This is an implementation and deployment detail, outside the scope of this
specification.

---

## 7. The match result

Whichever method defined an intent, a successful match yields one uniform
**result**:

- the **qualified intent name** `skill_id:intent_name` (§3) — identifying which
  intent, and therefore which skill and which handler;
- a **slot map** — a mapping of names to extracted text values. For a keyword
  intent the keys are vocabulary names (§4.3); for a template intent the keys
  are slot names (§5.2). A slot map **MAY** be empty.

Slot values are **opaque sequences of words**, returned as text. This
specification defines **no** value typing or coercion — consistent with
OVOS-INTENT-1 §5.2–§5.3, which places slot typing out of scope alongside
the text normalization it depends on.

In classifier terms the qualified intent name is the **label** and the slot
map is the **payload**: the label selects which handler runs, the payload is
the data that handler is given. The orchestrator routes the result to the **one handler** bound to
that intent at registration, and to nothing else. The handler always receives
the slot map — a set of key-value pairs, possibly empty — and that is the
only data it receives from the intent layer. Running that handler is the whole
purpose of the intent: it is the point at which a recognized natural-language
command becomes the execution of the developer's code.

A result **MAY** be accompanied by engine metadata such as a confidence score;
such metadata is engine-specific and is **not** defined by this specification.

### 7.1 Handlers must cope with missing optional slots

A handler **MAY** rely on slots listed in the intent's `required_slots` (§5.3):
when present, the engine guarantees those slots are populated in every match
result. For all other slots, a handler **MUST NOT** assume they are present.
The map may be empty or partial: an optional slot may simply not have occurred
(§4.2, §5.1), and classification is never perfect — an engine may extract
fewer slots than the handler expects, or route a misclassified utterance to it.
The intent layer guarantees only the label and whatever slots were extracted;
it does not guarantee that every non-required slot a handler relies on is
present or correct.

Coping with a missing or implausible slot is therefore the **handler's**
responsibility. A handler **SHOULD** prompt the user for data it needs but did
not receive, and **MAY** decline — treating the call as a misclassification —
when the slot values do not make sense for it. This recovery behaviour is
skill logic and is out of scope for this specification; what the specification
requires is only that a handler not assume a complete slot map.

---

## 8. Conformance

**A skill** that defines intents **MUST**, for each intent:

- define each of its registrations by exactly one method — keyword (§4) or
  template (§5) — with at most one registration per method per intent (§2);
- give it an intent name unique within the skill — the skill itself having an
  assistant-unique skill id (§3) — a language, and a handler;
- ensure the definition conforms to §4 or §5 and to OVOS-INTENT-1 and
  OVOS-INTENT-2;
- register the definition and the handler together as one unit (§6.1);
- write the handler so it copes with a missing or partial slot map (§7.1).

**An intent engine** **MUST**:

- accept at least one definition method and declare which (§6.2);
- reject a registration whose method it does not accept (§6.2);
- expand vocabulary entries (§4.1) and templates (§5.1) with an
  OVOS-INTENT-1-conformant expander — both are written in that grammar;
- for keyword intents, honour the constraint semantics of §4.2 — never
  reporting a match that violates a required, one-of, or excluded constraint;
- for template intents, honour the training-data contract of OVOS-INTENT-1 §6,
  and apply any paired `.blacklist` as match suppression (§5.5);
- report at most one matched intent per utterance (§6.2);
- produce the match result of §7 and report it for the orchestrator to route to the
  bound handler.

Matching, generalization, scoring, and the ranking of competing intents are
deliberately **unconstrained**: an engine MAY use any strategy.

---

## See also

- *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the template
  grammar used by template intents (§5) and by keyword vocabularies (§4.1), the
  named-slot model, and the training-data contract.
- *Locale Resource Formats Specification* (OVOS-INTENT-2) — the `.intent`,
  `.voc`, and `.entity` resource files an intent is defined from.
