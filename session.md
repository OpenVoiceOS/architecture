# Session Specification

**Spec ID:** OVOS-SESSION-1 · **Version:** 1 · **Status:** Draft

This document defines the **wire shape** of the `session` carrier —
the JSON object that travels inside `Message.context.session` — and
the rules consumers follow when reading and propagating it.

Its scope is narrow on purpose: the **shape on the wire** and **how
it may be consumed**. Lifecycle (when a session begins, ends,
expires, resumes), storage, authorization, and the semantics of
fields owned by other specifications are out of scope.

This specification is **prescriptive, not descriptive**. The field set
it lists in §3 is the closed set of fields with normative meaning in
this version. A field that no normative specification claims is not a
field of `session`; a consumer that encounters such a field treats it
per §2.3.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and
**MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the JSON shape of the `session` carrier (§2);
- the **field-registry mechanism** (§2.1) that lets other normative
  specifications claim `session` fields;
- the **closed set of fields claimed in this version** (§3), each
  cited to its owner specification;
- the **propagation behaviour** of fields across the Message
  derivations of OVOS-MSG-1 §5 (§4);
- **serialization** (§5) and **conformance** (§6).

It does **not** define:

- the **semantics** of any field — owned by the citing
  specification;
- session **lifecycle** — when a session begins or ends, how it
  expires, how it is created, how it is resumed;
- a session **store** — central indexing, persistence, sharing
  between processes;
- **authentication, authorization, encryption, multi-tenant
  routing** — layer-2 concerns built on top of the OVOS-MSG-1 §3
  substrate;
- any field not claimed by a normative specification under §2.1.

---

## 2. Shape

`session` is a JSON object.

Every field defined or claimed under this specification is
**omissible on the wire but never nullable**:

- A producer **MAY** omit any field. Omission means *"let the
  orchestrator decide"* — the consumer fills the field with its own
  deployment default at the point of consumption (§2.5).
- A producer **MUST NOT** emit any field as JSON `null`. Fields are
  either present with a value drawn from the value space defined by
  the owner specification, or omitted entirely.
- When a field is present, it carries normative meaning — the
  consumer **MUST** interpret it per the owner specification, not
  substitute its default.

A consumer that encounters an explicit `null` **MUST** treat it as a
malformed value: it **SHOULD** log the violation and **MUST** behave
as if the field were omitted (§2.5). A consumer **MUST NOT** reject
the Message solely because of a `null` field — fall back to the
omitted-field rule instead.

A session with only `session_id` is well-formed. A session with the
empty object `{}` is well-formed and is interpreted per §2.5.

### 2.5 Omission means "let the orchestrator decide"

Field omission is the **single mechanism** by which a producer defers
a session value to the orchestrator. An omitted field is interpreted
identically at every consumer that sees it: the consumer fills the
field with its own deployment default at the point of consumption.

This applies uniformly across the whole field set:

- An **omitted single field** means "let the orchestrator decide
  this one field." The remaining fields that are present carry
  normative meaning and are consumed per their owner specifications.
- An **omitted `session_id`** is filled by the consumer with the
  reserved value `"default"` (§3.1) — the session resolves to the
  device-local default.
- An **empty session** (`session: {}`) means "let the orchestrator
  decide every field." `session_id` is one of those fields, so an
  empty session resolves to `session_id: "default"` (§3.1) with
  every other field filled from deployment defaults.
- An **absent `session`** (no `session` key in `context`) is
  equivalent to an empty session — same resolution, including
  `session_id: "default"`.

The consumer's deployment defaults are the values it would apply when
no override is set: the deployment-configured `pipeline` ordering,
the deployment language, the deployment-configured transformer
chains, an empty `context`, and so on. This is a **read-side**
behaviour — every consumer arrives at the same effective session by
filling its own defaults.

A consumer **MUST NOT** treat an absent or empty `session`, or any
omitted field, as an unknown or untrusted origin. Absence and the
empty object are equivalent for every policy decision defined by
this specification; both resolve, at consumption, to the device-local
default session bearing `session_id: "default"` (§3.1).

A consumer **MAY** **materialize** an omitted field or an
empty / absent session at any point — that is, replace the omission
on a Message it emits with an explicit value drawn from its
deployment defaults. Materialization is governed by §4.1.

### 2.1 Field-registry mechanism

Other normative specifications **MAY** claim additional `session`
fields. A specification that claims a field **MUST**:

1. **Name** the field unambiguously: a short, lowercase, `snake_case`
   identifier (no `:`, no whitespace, no nested dotted paths).
2. **Fix** the field's wire type — one of: string, boolean, number,
   array, object — and document its full shape and permitted values.
3. **Declare** the field's **propagation rule**. The default is
   "propagates unchanged" (§4); a specification **MAY** declare a
   field *non-propagating* (stripped on `forward` / `reply` /
   `response`), in which case the carrier's behaviour overrides §4
   for that field only.
4. **Declare** the field's **scope**: *session-scoped* (set once,
   travels with the session) or *per-Message* (overwritten on every
   Message). Session-scoped is the default.
5. **Specify** the **deployment-default value** the consumer falls
   back to when the field is omitted (§2.5). The default **MAY** be
   "no behaviour" (the consumer skips the field-dependent action) or
   a concrete value drawn from deployment configuration. A consumer
   **MUST NOT** reject a Message because a claimed field is omitted;
   the default applies.
6. **Avoid collision** with any field already claimed by this
   specification (§3) or by another specification in force.

There is no central registry document beyond §3. The claiming
specification is itself the registry entry. A subsequent version of
this specification **SHOULD** update the §3 table to reflect
newly-claimed fields, but the wire contract a producer or consumer
follows is the union of §3 and every specification that claims a
field. A consumer is bound by the claim itself, not by §3's
enumeration of it; §3 is a convenience roster, not the source of
normativity for claimed fields.

### 2.2 Closed set

Every field with **normative meaning** on `session` is listed in §3
or is claimed by a specification that follows §2.1. A field that
appears in `session` but is claimed by no normative specification is
non-normative — carried for the convenience of producers and consumers
that recognize it, but no consumer is bound to interpret it, no
producer is bound to emit it, and a consumer that does not recognize
it treats it per §2.3.

### 2.3 Unknown-field tolerance

A consumer **MUST NOT** reject a Message because `session` carries a
key the consumer does not know. A consumer **MUST NOT** strip unknown
keys from a session it propagates (§4). A consumer **MAY** log unknown
keys for diagnostic purposes.

This rule is symmetric with OVOS-MSG-1 §2.3 for `context` and is what
makes the registry forward-compatible: a producer that adopts a
newly-claimed field does not break consumers that predate the claim.

---

## 3. Fields claimed in this version

This version of the specification recognizes the following fields.
The "Owner" column names the specification that defines the field's
semantic meaning, permitted values, propagation rule, and scope. This
specification fixes only the field name and the wire type;
everything else is owned by the cited specification.

| Field | Wire type | Owner |
|-------|-----------|-------|
| `session_id` | string | §3.1 (this spec) |
| `lang` | string (BCP-47) | §3.2 (this spec) |
| `stt_lang` | string (BCP-47) | §3.2 (this spec) |
| `request_lang` | string (BCP-47) | §3.2 (this spec) |
| `detected_lang` | string (BCP-47) | §3.2 (this spec) |
| `pipeline` | array of string | OVOS-PIPELINE-1 §5 |
| `context` | object | OVOS-CONTEXT-1 §2 |
| `audio_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `utterance_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `metadata_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `intent_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `dialog_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `tts_transformers` | array of string | OVOS-TRANSFORM-1 §5 |

Every field above is OPTIONAL on the wire. A producer that sets a
field **MUST** use the wire type listed and the value space defined
by the owner specification. A consumer that recognizes a field
**MUST** interpret it per the owner specification.

### 3.1 `session_id` semantics and the reserved `"default"` value

`session_id` is the **identity** of a session within a deployment. Two
Messages bearing the same `session_id` belong to the same session;
two Messages with distinct `session_id` values do not. A consumer
that maintains per-session state **MUST** key that state on
`session_id`.

`session_id` is an **opaque string** to this specification. A
consumer **MUST NOT** parse or ascribe structure to its value beyond
string equality, with one exception: the value **`"default"`** is
**reserved** and carries one specific meaning:

> *the Message originates from the device itself.*

It marks a Message as locally-originated rather than as belonging to
any remote or named participant.

`"default"` is also the value a consumer fills in whenever
`session_id` is omitted (§2.5). This means an absent `session`, an
empty `session: {}`, and an explicit `session_id: "default"` all
resolve to the same identifier at consumption: `"default"`. A
consumer **MUST NOT** treat the three forms differently for any
policy decision defined by this specification.

A producer that wants the Message treated as device-local **MAY**
either omit `session_id` (or `session` entirely) or set
`session_id: "default"` explicitly. The two are equivalent on the
wire.

A consumer that wants to apply different policy to device-local
Messages (audio routing, presence sensing, output locality) **MAY**
branch on `session_id == "default"`. No other policy hook is defined
by this specification on the value of `session_id`.

The reserved value is not a distinguished kind of session in the
schema; it is a normal session that carries the same field set as any
other (§3), distinguished only by its identifier.

### 3.2 Language signals

A session carries up to four BCP-47 language-tag fields, each
naming a different *kind* of language signal. All four are
session-scoped, all four are omissible per §2, and all four are
populated independently (typically by different stages of the
pipeline, by different components, or by an out-of-band caller).

Their **meanings** are normative; how a consumer **consolidates**
them into a single language for any given operation is not — that
choice is stage-dependent and implementation-specific. §3.2.5
suggests a default consolidation pattern as informative guidance.

#### 3.2.1 `lang`

`lang` — string — the **user's preferred language**, as a BCP-47
language tag. It declares which language the participant on the
external side of the bus boundary wants to communicate in. It is the
base signal: stable across the session, not derived from any one
utterance, and the natural fallback when no per-utterance signal is
available.

#### 3.2.2 `stt_lang`

`stt_lang` — string — the BCP-47 tag the speech-to-text stage
**actually transcribed in**. It records the language the audio was
decoded as, regardless of what was requested or expected. It is
typically populated by the component that produced the transcript;
once set, it travels with the session until overwritten by a later
stage that re-transcribes.

#### 3.2.3 `request_lang`

`request_lang` — string — the BCP-47 tag the **caller explicitly
requested** for downstream processing of this session. It is the
"please treat this session as language X" override: a remote client,
a UI selector, or a layer-2 router populates it to pin language
behaviour regardless of what the user is preferring (`lang`) or what
the audio was transcribed as (`stt_lang`).

#### 3.2.4 `detected_lang`

`detected_lang` — string — the BCP-47 tag a **language-detection
component** classified the most recent utterance as. It records the
opinion of a detector (acoustic, lexical, or hybrid) and may differ
from both `stt_lang` (which records what STT decoded the audio as,
which can fail when STT is fixed to a single language) and `lang`
(which records the user's stable preference).

#### 3.2.5 Consolidation (informative)

A consumer that needs **one** language for a particular operation
must consolidate the available signals into a single value. The
**right priority order is stage-dependent**: different stages of the
pipeline reasonably prioritize different signals. This specification
does not bind a single ordering; it lists each signal's meaning
(above) and leaves consolidation to the orchestrator and the consumer
performing the operation.

As **informative guidance** — not a normative rule — a sensible
default ordering for most stages is:

```
request_lang  →  stt_lang  →  detected_lang  →  lang  →  deployment default
```

with the per-payload `data.lang` (the content language of the
Message being processed) taking absolute priority for any operation
whose purpose is to act on that payload's content (TTS narrating
already-rendered text, translation of a specific string, format
conversion of a specific resource).

A consumer **MAY** choose any consolidation order that suits its
stage and need. A consumer **MUST NOT** assume any one signal is
present, **MUST NOT** assume one signal equals another, and **MUST
NOT** mutate any signal as a side effect of consolidating.

#### 3.2.6 `data.lang` (per-payload, not session-scoped)

The session-level fields above describe **session state**. The
language *of a particular Message's payload* is a per-payload concept
and is owned by the specification that defines the Message's topic.
By convention many topics carry a `data.lang` field describing the
language of the content in that Message (an utterance just
transcribed, a resource just registered, a dialog just rendered).

`data.lang` is **not** a session field and is not propagated by §4.
A consumer that needs the payload's content language reads
`data.lang` directly; it **MUST NOT** assume `data.lang` equals
`session.lang` or any other session-level signal.

### 3.3 Wire weight

Sessions carrying every per-component override populated may add
several hundred bytes to each Message. Because §4 propagates
`session` across every `forward` / `reply` / `response` derivation,
the override bytes ride along on every handler emission, on every
observer notification, on every cross-process hop. Producers **SHOULD
NOT** populate per-component override fields whose values match the
deployment default — set them only when the session genuinely diverges
from the default. Consumers **MUST** tolerate the resulting wire
weight; this specification places no maximum on session size.

---

## 4. Propagation

The Message-level propagation rule of OVOS-MSG-1 §4.3 — that
`session` rides unchanged across `forward`, `reply`, and `response`
derivations — applies unmodified to every field of §3.

For the avoidance of doubt:

- Every field in §3 propagates with the same rule unless its owner
  specification declares it non-propagating per §2.1 step 3.
- A consumer that derives a Message **MUST NOT** strip session
  fields it does not understand; it **MUST** preserve them so that a
  later consumer in the chain that does understand the field can
  read it (§2.3).
- A consumer that **does** modify a session field (because it owns
  the field's semantics and the modification is part of its
  contract) **MAY** do so. Any such mutation is governed by the
  field owner's specification, not this one.

### 4.1 Default materialization

OVOS-MSG-1 §4.3 permits an implementation to **materialize** a
default session on a derived Message when the source Message had no
`session`. A materialized default **MUST** set
`session_id: "default"`. A materialized default **MUST NOT** populate
the per-component override fields of §3 (`pipeline`, `context`, the
six `*_transformers`) — those fields have meaning only when explicitly
set by the session origin, and a materialized default would falsely
declare a divergence from deployment defaults that the origin never
asked for.

---

## 5. Serialization

A session is a JSON object embedded in `Message.context.session`. It
follows OVOS-MSG-1 §6 serialization rules:

- UTF-8 JSON per RFC 8259;
- no comments, no trailing commas;
- key order is **not significant**; producers and consumers **MUST
  NOT** rely on it;
- numbers **MUST** be finite (no NaN, no infinities);
- the `session` value is a single JSON object — not an array, not a
  string-encoded JSON blob.

A consumer that cannot parse `session` as a JSON object **MUST**
treat the Message as malformed per OVOS-MSG-1 §2 and §6.

---

## 6. Conformance

### A **producer** of session-carrying Messages **MUST**:

- populate `session` as a JSON object conforming to §2;
- give `session_id` a non-empty string value when set;
- when setting any field listed in §3, use the wire type fixed by §3
  and the value space fixed by the owner specification;
- propagate `session` unchanged across Message derivations per
  OVOS-MSG-1 §5 and §4 of this specification;
- not strip session fields it does not understand (§2.3, §4).

A producer **MUST NOT**:

- emit any session field with the JSON value `null` (§2); a field
  is either present with a value drawn from the owner specification's
  value space, or omitted entirely;
- populate a per-component override field (§3 — `pipeline`, `context`,
  the six `*_transformers`) with a value that matches the deployment
  default merely as a form of explicit confirmation. Omit the field
  and let the orchestrator's default apply (§2.5, §3.3).

### A **consumer** of session-carrying Messages **MUST**:

- treat an omitted field, an empty session object `{}`, and an
  absent `session` identically — all mean "let the orchestrator
  decide" and resolve to deployment defaults at consumption (§2.5);
- treat an explicit `null` as a malformed value: behave as if the
  field were omitted and **SHOULD** log the violation (§2);
- tolerate any field it does not recognize and propagate it
  unchanged on derived Messages (§2.3, §4);
- key per-session state on `session_id`;
- not reject a Message because of the presence, absence, or value
  of any single session field — invalid values for fields whose
  owner specification defines a fallback cause that fallback, never
  Message rejection.

A consumer **SHOULD**:

- log unknown session fields for diagnostic purposes.

### A specification that **claims a new session field** **MUST**:

- follow §2.1 in full — name, wire type, propagation, scope,
  absence rule, no collision;
- be self-contained: define everything the field needs in the
  claiming specification, not by reference to this one.

### Non-goals

The following are explicitly **outside** this specification and
**MUST NOT** be inferred from it: session lifecycle (creation,
expiration, end-of-session events), a session-store protocol,
central session indexing, session authentication and authorization,
session-resumption semantics, per-field encryption, multi-tenant
session isolation guarantees beyond the opaque `session_id` keying,
and any field not claimed under §2.1 by a normative specification.

---

## See also

- **OVOS-MSG-1** — defines `Message.context` as the carrier and the
  `forward` / `reply` / `response` derivations that propagate
  `session` unchanged.
- **OVOS-PIPELINE-1** — owns `session.pipeline`.
- **OVOS-CONTEXT-1** — owns `session.context`.
- **OVOS-TRANSFORM-1** — owns the six `session.*_transformers`
  fields.
