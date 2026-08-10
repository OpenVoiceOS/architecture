# Session Specification

**Spec ID:** OVOS-SESSION-1 · **Version:** 1 · **Status:** Draft

This document defines the **wire shape** of the `session` carrier —
the JSON object that travels inside `Message.context.session` — and
the rules consumers follow when reading and propagating it.

Its scope is narrow on purpose: the **shape on the wire** and **how
it may be consumed**. Lifecycle (when a session begins, ends,
expires, resumes), storage, authorization, and the semantics of
fields owned by other specifications are out of scope.

This specification is **prescriptive, not descriptive**. The closed
set of fields with normative meaning is the set claimed under the §2.2
registry mechanism; §3 enumerates that set as it stands at the
publication of this version. A field that no normative specification
claims is not a field of `session`; a consumer that encounters such a
field treats it per §2.4.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, **OPTIONAL**, and **RECOMMENDED** are used as in RFC 2119 and
RFC 8174 — they carry their normative meaning only when written in all
capitals.

---

## 1. Scope

This specification defines:

- the JSON shape of the `session` carrier (§2);
- the **field-registry mechanism** (§2.2) that lets other normative
  specifications claim `session` fields;
- the **closed set of fields claimed in this version** (§3), each
  cited to its owner specification;
- the **propagation behaviour** of fields across the Message
  derivations of OVOS-MSG-1 §5.1–§5.3, and the strengthening of
  OVOS-MSG-1 §4.1 that this specification applies to them (§4);
- **serialization** (§5) and **conformance** (§6).

It does **not** define:

- the **semantics** of any field owned by another specification —
  those are owned by the citing specification. The exception is the
  fields this specification owns itself: `session_id` (§3.1) and the
  six language signals (§3.2), whose semantics are defined here;
- session **lifecycle** — when a session begins or ends, how it
  expires, how it is created, how it is resumed (owned by
  OVOS-SESSION-2);
- a session **store** — central indexing, persistence, sharing
  between processes;
- **authentication, authorization, encryption, multi-tenant
  routing** — layer-2 concerns built on top of the OVOS-MSG-1 §3
  substrate;
- any field not claimed by a normative specification under §2.2.

---

## 2. Shape

`session` is a JSON object.

Every field defined or claimed under this specification is
**omissible on the wire but never nullable**:

- A producer **MAY** omit any field. Omission means *"let the
  orchestrator decide"* — the consumer fills the field with its own
  deployment default at the point of consumption (§2.1).
- A producer **MUST NOT** emit any field as JSON `null`. Fields are
  either present with a value drawn from the value space defined by
  the owner specification, or omitted entirely.
- When a field is present, it carries normative meaning — the
  consumer **MUST** interpret it per the owner specification, not
  substitute its default.

A consumer that encounters an explicit `null` **MUST** treat it as a
malformed value: it **SHOULD** log the violation and **MUST** behave
as if the field were omitted (§2.1). A consumer **MUST NOT** reject
the Message solely because of a `null` field — fall back to the
omitted-field rule instead.

The same rule governs a **wrong-typed** value. A field whose value is
not of the wire type fixed for it (§3, or the claiming specification
under §2.2) is malformed: the consumer **MUST** behave as if the field
were omitted (§2.1), **SHOULD** log the violation, and **MUST NOT**
reject the Message because of it. `null` is the special case of this
rule that a producer is additionally forbidden to emit.

A session with only `session_id` is well-formed. A session with the
empty object `{}` is well-formed and is interpreted per §2.1.

### 2.1 Omission means "let the orchestrator decide"

Field omission is the **single mechanism** by which a producer
defers a session value to the orchestrator. A producer **MUST**
defer by omitting the field; this specification provides no other
deferral surface (no `null`, no sentinel value, no separate
"unset" Message). An omitted field is interpreted identically at
every consumer that sees it: the consumer **MUST** fill the field
with its own deployment default at the point of consumption.

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

What "the consumer's own default" is depends on which of **two
resolution classes** the field belongs to. The owner specification
fixes the class when it claims the field (§2.2 item 3):

- **Override fields** — fields by which the session origin overrides
  a deployment-configured behaviour: `pipeline`, the six
  `*_transformers` chains, the `blacklisted_*` denylists, `lang` and
  the other language signals, `fallback_handlers`
  (OVOS-FALLBACK-1 §4). An omitted
  override field resolves to the value the consumer would apply when
  no override is set: the deployment-configured `pipeline` ordering,
  the deployment language, the deployment-configured transformer
  chains, and so on.
- **State-record fields** — fields that record state accumulated
  within the session rather than a preference over deployment
  configuration: `active_handlers` (OVOS-PIPELINE-1 §7.1),
  `converse_handlers` and `response_mode` (OVOS-CONVERSE-1 §2.1,
  §2.2), `intent_context` (OVOS-CONTEXT-1 §2). There is no deployment
  default to fall back to: an omitted state-record field resolves to
  the **empty value the owner specification names** — `[]` for the
  recency lists, `{}` for `intent_context`, "no holder" for
  `response_mode`. A consumer **MUST NOT** substitute deployment
  configuration for an omitted state record.

Either way this is a **read-side** behaviour — every consumer arrives
at the same effective session by filling its own defaults.

A consumer **MUST NOT** treat an absent or empty `session`, or any
omitted field, as an unknown or untrusted origin. Absence and the
empty object are equivalent for every policy decision defined by
this specification; both resolve, at consumption, to the device-local
default session bearing `session_id: "default"` (§3.1).

A consumer **MAY** **materialize** an omitted field or an
empty / absent session at any point — that is, replace the omission
on a Message it emits with an explicit value drawn from its
deployment defaults. Materialization is governed by §4.1.

### 2.2 Field-registry mechanism

Other normative specifications **MAY** claim additional `session`
fields. A specification that claims a field **MUST**:

1. **Name** the field unambiguously: a short, lowercase, `snake_case`
   identifier (no `:`, no whitespace, no nested dotted paths).
2. **Fix** the field's wire type — one of: string, boolean, number,
   array, object — and document its full shape and permitted values.
   JSON has one numeric type; a specification that needs an integer
   claims the type as **number** and states the integer constraint in
   its own value space (a consumer treats a non-integral value for
   such a field as malformed per §2). An **array** claim states the
   element type; an **object** claim states every key it defines, each
   key's own type, and whether unlisted keys are permitted.
3. **Specify** the **deployment-default value** the consumer falls
   back to when the field is omitted (§2.1). The default **MAY** be
   "no behaviour" (the consumer skips the field-dependent action) or
   a concrete value drawn from deployment configuration. A consumer
   **MUST NOT** reject a Message because a claimed field is omitted;
   the default applies.
4. **Avoid collision** with any field already claimed by this
   specification (§3) or by another specification in force.

There is no central registry document beyond §3. The claiming
specification is itself the registry entry. A subsequent version of
this specification **SHOULD** update the §3 table to reflect
newly-claimed fields, but the wire contract a producer or consumer
follows is the union of §3 and every specification that claims a
field. A consumer is bound by the claim itself, not by §3's
enumeration of it; §3 is a convenience roster, not the source of
normativity for claimed fields.

### 2.3 Closed set

Every field with **normative meaning** on `session` is listed in §3
or is claimed by a specification that follows §2.2. A field that
appears in `session` but is claimed by no normative specification is
non-normative — carried for the convenience of producers and consumers
that recognize it, but no consumer is bound to interpret it, no
producer is bound to emit it, and a consumer that does not recognize
it treats it per §2.4.

### 2.4 Unknown-field tolerance

A consumer **MUST NOT** reject a Message because `session` carries a
key the consumer does not know. A consumer **MUST NOT** strip unknown
keys from a session it propagates (§4). A consumer **MAY** log unknown
keys for diagnostic purposes.

This rule is symmetric with OVOS-MSG-1 §2.3 for `context` and is what
makes the registry forward-compatible: a producer that adopts a
newly-claimed field does not break consumers that predate the claim.

### 2.5 Malformed carrier

The tolerance rules above (§2, §2.4) govern a **malformed field** — a
`null` or wrong-typed value on an otherwise well-formed `session`
object. They resolve field-by-field: the offending field is treated as
omitted and the rest of the session is consumed normally. They presume
a `session` that **is** a JSON object.

A **malformed carrier** is different: the `session` value is present in
`context` but is **not a JSON object** at all (a string, number,
array, or boolean). There is no field set to interpret, no
`session_id` to key conversation state on — the carrier itself is
unusable, so the field-by-field rules cannot apply.

On a malformed carrier a consumer:

- **MUST NOT** crash, and **MUST NOT** let the error tear down its
  transport. A malformed carrier is a **per-message** producer fault,
  never a transport fault; a consumer that drops its bus connection
  over one bad message can be held offline indefinitely by a single
  misbehaving producer.
- **SHOULD** reject (drop) the offending Message and **SHOULD** log the
  violation.
- **MUST NOT** substitute the default session and process the Message
  as though the carrier were valid. Absence resolves to the default
  (§2.1); a malformed carrier does **not** — fabricating a session
  identity for a message whose producer supplied a broken one would
  route that message into the wrong session, which is worse than
  dropping it.

An explicit `null` for the whole `session` value is **absence**, not a
malformed carrier: it resolves to the default session per §2.1, the
same as an omitted `session` key.

---

## 3. Fields claimed in this version

This version of the specification recognizes the following fields.
The "Owner" column names the specification that defines the field's
semantic meaning and permitted values. For a field owned by another
specification this specification fixes only the field name and the
wire type; everything else is owned by the cited specification. For
the fields whose owner column names a section of *this* document
(§3.1, §3.2) the semantics below are normative here.

All fields propagate unchanged on derivation (§4, over the
OVOS-MSG-1 §5.1–§5.3 derivations); all fields are session-scoped —
they travel with the session rather than with one Message payload.

Session scope is not uniform persistence. The **per-utterance
observation** fields — `stt_lang`, `request_lang`, `detected_lang`
(§3.2.4–§3.2.6) — record what a stage observed about the **current**
utterance and are overwritten by the next stage that makes the same
observation. They ride on the session as the current reading, not as
an accumulated history; a consumer **MUST NOT** read a surviving
value as a description of an older utterance.

| Field | Wire type | Owner |
|-------|-----------|-------|
| `session_id` | string | §3.1 (this spec) |
| `lang` | string (BCP-47) | §3.2 (this spec) |
| `secondary_langs` | array of string (BCP-47) | §3.2 (this spec) |
| `output_lang` | string (BCP-47) | §3.2 (this spec) |
| `stt_lang` | string (BCP-47) | §3.2 (this spec) |
| `request_lang` | string (BCP-47) | §3.2 (this spec) |
| `detected_lang` | string (BCP-47) | §3.2 (this spec) |
| `pipeline` | array of string | OVOS-PIPELINE-1 §5 |
| `intent_context` | object | OVOS-CONTEXT-1 §2 |
| `active_handlers` | array of object `{skill_id, activated_at}` | OVOS-PIPELINE-1 §7.1 |
| `converse_handlers` | array of object `{skill_id, activated_at}` | OVOS-CONVERSE-1 §2.1 |
| `response_mode` | object `{skill_id, expires_at}` | OVOS-CONVERSE-1 §2.2 |
| `fallback_handlers` | array of string | OVOS-FALLBACK-1 §4 |
| `persona_id` | string | OVOS-PERSONA-1 §3 |
| `audio_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `utterance_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `metadata_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `intent_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `dialog_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `tts_transformers` | array of string | OVOS-TRANSFORM-1 §5 |
| `blacklisted_skills` | array of string | OVOS-PIPELINE-1 §5 |
| `blacklisted_intents` | array of string | OVOS-PIPELINE-1 §5 |
| `blacklisted_pipelines` | array of string | OVOS-PIPELINE-1 §5 |
| `blacklisted_audio_transformers` | array of string | OVOS-TRANSFORM-1 §5.2 |
| `blacklisted_utterance_transformers` | array of string | OVOS-TRANSFORM-1 §5.2 |
| `blacklisted_metadata_transformers` | array of string | OVOS-TRANSFORM-1 §5.2 |
| `blacklisted_intent_transformers` | array of string | OVOS-TRANSFORM-1 §5.2 |
| `blacklisted_dialog_transformers` | array of string | OVOS-TRANSFORM-1 §5.2 |
| `blacklisted_tts_transformers` | array of string | OVOS-TRANSFORM-1 §5.2 |
| `site_id` | string | OVOS-BRIDGE-1 §3.3 |

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

> *interact with the device-local session.*

A Message bearing `session_id: "default"` is processed as part of
the device's default session — the persistent, locally-held session
described in OVOS-SESSION-2 §5. This is the normal path for
messages that originate from the device itself, but it is equally
valid for **remote clients that wish to interact with the local
device** (remote-control commands, home-automation "speak" requests,
media injection from a layer-2 framework). Using `"default"` from a
remote client is deliberate impersonation of the device-local
session; whether that is authorized is a **layer-2 concern** outside
this specification.

A layer-2 authentication system **MAY** gate access to the default
session behind an elevated-privilege flag (an "admin" grant or
equivalent); SESSION-1 places no requirement on it.

`"default"` is also the value a consumer fills in whenever
`session_id` is omitted (§2.1). This means an absent `session`, an
empty `session: {}`, and an explicit `session_id: "default"` all
resolve to the same identifier at consumption: `"default"`. A
consumer **MUST NOT** treat the three forms differently for any
policy decision defined by this specification.

A producer that wants to interact with the device-local session
**MAY** either omit `session_id` (or `session` entirely) or set
`session_id: "default"` explicitly. The two are equivalent on the
wire.

A consumer that wants to apply different policy to the default
session (audio routing, presence sensing, output locality) **MAY**
branch on `session_id == "default"`. No other policy hook is defined
by this specification on the value of `session_id`.

*Informative:* the reserved value is not a distinguished kind of
session in the schema; it is a normal session that carries the same
field set as any other (§3), distinguished only by its identifier.

### 3.2 Language signals

A session carries up to six BCP-47 language-tag fields, each
naming a different *kind* of language signal — one purpose per
field, no overlaps:

| Field | The one thing it records |
|-------|--------------------------|
| `lang` | the user's stable input-side language preference (§3.2.1) |
| `secondary_langs` | the ordered fallback pool of additional languages the user accepts (§3.2.2) |
| `output_lang` | the language the user wants responses rendered in, when it differs from the input side (§3.2.3) |
| `stt_lang` | the language the speech-to-text stage assumed for the audio (§3.2.4) |
| `request_lang` | the emitter's per-utterance hint of the expected language (§3.2.5) |
| `detected_lang` | a language detector's classification of the most recent utterance (§3.2.6) |

All six are session-scoped, all six are omissible per §2, and all
six are populated independently (typically by different stages of
the pipeline, by different components, or by an out-of-band
caller). Preference (`lang`, `secondary_langs`, `output_lang`) is
declared by the session origin and stable; observation (`stt_lang`,
`request_lang`, `detected_lang`) is written per utterance by the
stage that made it and may disagree with the preferences and with
each other — disagreement is signal, not error.

Their **meanings** are normative; how a consumer **consolidates**
them into a single language for any given operation is not — that
choice is stage-dependent and implementation-specific. §3.2.7
suggests a default consolidation pattern as informative guidance.

#### 3.2.1 `lang`

`lang` — string — the **user's preferred language**, as a BCP-47
language tag. It declares which language the participant on the
external side of the bus boundary wants to communicate in. It is the
base signal: stable across the session, not derived from any one
utterance, and the natural fallback when no per-utterance signal is
available.

#### 3.2.2 `secondary_langs`

`secondary_langs` — array of string — additional BCP-47 tags the
participant **also speaks or understands**, ordered by preference
(most-preferred first). It is the broader language set the session
operates inside; `lang` is the primary, `secondary_langs` is the
fallback pool.

`secondary_langs` **MUST NOT** contain `lang` at the time of
emission (it is *additional* languages, not a list including the
primary). It **MUST NOT** contain duplicates. An empty array and an omitted field are
equivalent and mean "no additional languages declared".

Typical uses by consumers:

- **Constraining a language detector** — a detector reading
  `lang` + `secondary_langs` produces predictions only from that
  candidate set, instead of from the detector's full label space.
  A detected language outside the set is either coerced to the
  nearest in-set member or reported as unknown, at the detector's
  discretion.
- **Fallback selection** — a stage that cannot serve `lang`
  (missing TTS voice, missing intent locale, missing translation
  pair) **MAY** walk `secondary_langs` in order and pick the first
  it can serve, instead of falling all the way to a deployment
  default.
- **Gating outputs** — a stage that renders text **MAY** decline
  to render in a language that is neither `lang` nor in
  `secondary_langs`, to avoid producing content the participant
  will not understand.

`secondary_langs` is a hint, not an authorization boundary: a
consumer **MAY** ignore it. Per §3.2.7, no consolidation order is
prescribed.

#### 3.2.3 `output_lang`

`output_lang` — string — the BCP-47 tag the participant wants the
**assistant's responses rendered in**, independently of the input
language. It is an output-side preference: a user who speaks German
but always wants English replies sets `output_lang: "en-US"`.

When `output_lang` is **omitted**, the assistant replies in whatever
language naturally falls out of input-side signals (consumer's
choice per §3.2.7 — typically `lang`, `stt_lang`, or the per-payload
content language). This is the status quo: input language and output
language are the same.

When `output_lang` is **set**, a stage that renders text not yet
produced (dialog selection, prompt selection, response composition,
GUI text) **SHOULD** render in `output_lang` if it has the resources
to do so (a localized dialog, a TTS voice, a prompt in that
language). When the stage cannot render in `output_lang`, it **MAY**
fall back to `secondary_langs` (§3.2.2) and then to the input-side
language; alternatively a deployment **MAY** insert a translation
transformer that rewrites the rendered text into `output_lang`
post-hoc — `output_lang` does not prescribe how the goal is met, only
that it is the goal.

`output_lang` is not consulted by TTS voice selection directly: TTS
narrates already-produced text and keys on the payload `data.lang`
of the text being spoken (§3.2.8). `output_lang` influences which
language the upstream renderer produced, which determines `data.lang`,
which TTS then voices. The cascade is intentional: a single
preference field controls the language of every output stage.

A consumer that **cannot** render in `output_lang` and has no fallback
strategy **MUST NOT** silently render in another language without
recording the divergence: it **MUST** set the rendered Message's
`data.lang` (§3.2.8) to the language it actually rendered in, so that
downstream TTS voices the text correctly. Setting `data.lang` is the
only means this specification gives for recording the divergence, so
the obligation above is discharged by that field and no other.

#### 3.2.4 `stt_lang`

`stt_lang` — string — the BCP-47 tag the speech-to-text stage was
**configured to assume** for the audio (the model's input language).
It is written by the audio input service before or at the point of
STT invocation. In a straightforward transcription, `stt_lang`
matches `data.lang` (the transcript's output language). In a
speech-translation model, they diverge: `stt_lang` is the audio's
spoken language; `data.lang` is the language the transcript was
produced in. Downstream stages that need the audio's source language
read `stt_lang`; stages that need the transcript's language read
`data.lang` or `session.lang`. Once set, `stt_lang` travels with
the session until overwritten by a later transcription stage.

#### 3.2.5 `request_lang`

`request_lang` — string — the BCP-47 tag the **emitter reported**
for this utterance at the point it was emitted. It is a hint about
what language the emitter expects the content to be in — not an
authoritative claim and not an override.

Typical sources of `request_lang`:

- a **multi-wakeword** setup where each wake word is associated
  with a language: the wakeword that triggered the capture
  determines the reported hint (the user pressed an "English wake
  word" so the emitter reports `en-US`). The detection itself is
  observable as `ovos.listener.wakeword` (OVOS-AUDIO-IN-1 §6.5),
  whose optional `lang` field carries the same binding;
- a UI lang selector the user toggled before speaking;
- a layer-2 router that knows the per-peer expected language.

The hint is **not authoritative**. The user may speak a different
language than the emitter expected (wake-word trigger does not
constrain what the user actually says next), and downstream stages
**MUST NOT** treat `request_lang` as a guarantee. The actual decoded
language is recorded by `stt_lang` (§3.2.4); a language-detection
component's opinion is recorded by `detected_lang` (§3.2.6);
disagreement between the three is normal.

A consumer **MAY** use `request_lang` as a prior — for example to
bias an STT model toward the reported language, or to break ties
when other signals are missing — but **MUST NOT** reject or override
contradictory `stt_lang` / `detected_lang` values purely on the
strength of `request_lang`.

#### 3.2.6 `detected_lang`

`detected_lang` — string — the BCP-47 tag a **language-detection
component** classified the most recent utterance as. It records the
opinion of a detector (acoustic, lexical, or hybrid) and may differ
from both `stt_lang` (which records what STT decoded the audio as,
which can fail when STT is fixed to a single language) and `lang`
(which records the user's stable preference).

#### 3.2.7 Consolidation (informative)

A consumer that needs **one** language for a particular operation
must consolidate the available signals into a single value. The
**right priority order is stage-dependent**: different stages of the
pipeline reasonably prioritize different signals. This specification
does not bind a single ordering; it lists each signal's meaning
(above) and leaves consolidation to the orchestrator and the consumer
performing the operation.

As **informative guidance** — not a normative rule — each pipeline
stage naturally prioritizes different signals:

- **STT configuration** — bias toward `request_lang` (the emitter's
  hint about what language is coming) before falling back to `lang`.
- **Language detection** — produce `detected_lang` from the audio or
  transcript; use `lang` + `secondary_langs` to constrain the
  candidate set.
- **Intent matching and dialog selection** — prefer `stt_lang` or
  `detected_lang` (the language the utterance was actually in), then
  `lang`.
- **Response rendering (dialog, prompt)** — prefer `output_lang`
  when set; fall back to `lang`.
- **TTS voice selection** — key on the per-payload `data.lang` of
  the text being spoken (§3.2.8); ignore `request_lang` entirely.

`data.lang` takes absolute priority for any operation whose purpose
is to act on a specific payload's content — it records the language
already present in the payload, which the operation must match.

A consumer **MAY** choose any consolidation order that suits its
stage and need. A consumer **MUST NOT** assume any one signal is
present, **MUST NOT** assume one signal equals another, and **MUST
NOT** mutate any signal as a side effect of consolidating.

#### 3.2.8 `data.lang` (per-payload, not session-scoped)

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

### 3.3 `site_id`

`site_id` is an opaque group identifier. Its full normative
definition — assignment rules, bridge behaviour, and consumer
constraints — is owned by **OVOS-BRIDGE-1 §3.3**. This section is
a registry pointer only; it states no rule of its own.

Consumers of `site_id` within the orchestrator pipeline (audio
routing, output-locality policy) **MAY** use it to scope decisions
to a physical or logical group. The consumer constraints are
OVOS-BRIDGE-1's, restated here verbatim for the reader's
convenience and normative only there:

> Once `site_id` is present on an inbound message after bridge
> processing, downstream components **MUST NOT** overwrite it.

> Consumers **MUST NOT** parse or ascribe structure to `site_id`
> beyond string equality.

Note the precondition in the first sentence: the ban applies **after
bridge processing**, which is precisely what leaves OVOS-BRIDGE-1
§3.3 step 1 free to override a client-supplied value.

### 3.4 Wire weight

Because §4 propagates `session` across every `forward` / `reply` /
`response` derivation, every populated override rides along on every
handler emission, on every observer notification, and on every
cross-process hop. This section defines the **canonical wire-weight
rule** consumed by every other field-claiming specification in the
registry.

**Omit-when-wire-equivalent-to-omission.** A producer **SHOULD**
omit any field whose value is wire-equivalent to omission. Three
canonical cases:

1. **`session_id == "default"`.** Per §3.1, an omitted `session_id`,
   an absent `session`, an empty `session: {}`, and an explicit
   `session_id: "default"` are all wire-equivalent. A producer that
   intends device-local origin **SHOULD** omit `session` entirely
   rather than emit the reserved string.
2. **A per-component override field whose value matches the
   deployment default.** Producers **SHOULD NOT** populate
   `pipeline`, `intent_context`, the six `*_transformers` lists,
   `blacklisted_skills`, `blacklisted_intents`,
   `blacklisted_pipelines`, or `site_id` with a value the consumer
   would compute as the deployment default anyway. Set them only
   when the session genuinely diverges from the default.
3. **An empty array on a list-valued override field.** For every
   list-valued override field claimed by §3 (and by other specs
   via the §2.2 registry), an empty array (`[]`) is
   wire-equivalent to omission: both resolve to the
   deployment default at consumption (§2.1). A producer
   **SHOULD** omit the field rather than emit `[]`. This includes
   the three denylists (`blacklisted_*`), the six
   `*_transformers` chains, and the `pipeline` ordering.

The rule is **SHOULD**, not **MUST**: a producer that emits a
redundant default-valued field is non-optimal but conformant. A
consumer **MUST** tolerate the resulting wire weight; this
specification places no maximum on session size.

Other specifications claiming session fields via §2.2 inherit
this rule for the fields they claim — they need not restate it.

## 4. Propagation

The Message-level propagation rule of OVOS-MSG-1 §4.1 — that
`session` rides unchanged across the `forward`, `reply`, and
`response` derivations of OVOS-MSG-1 §5.1–§5.3 — applies to every
field of §3, with one deliberate **strengthening**.

OVOS-MSG-1 §4.1 states propagation as a **SHOULD** for consumers, so
that a producer with no session awareness is still conformant at the
envelope level. Within the field set §3 claims, this specification
raises it to a **MUST**: a consumer that derives a Message from one
carrying a `session` **MUST** propagate that `session` unchanged. The
registry only works if a field survives every hop between the
component that sets it and the component that reads it, and a
consumer that silently drops the carrier breaks fields it has never
heard of. Nothing else about OVOS-MSG-1 §4.1 is modified; a producer
originating a Message is still free to emit no `session` at all.

For the avoidance of doubt:

- Every field in §3 propagates unchanged — no field is non-propagating.
- A consumer that derives a Message **MUST NOT** strip session
  fields it does not understand; it **MUST** preserve them so that a
  later consumer in the chain that does understand the field can
  read it (§2.4).
- A consumer that **does** modify a session field (because it owns
  the field's semantics and the modification is part of its
  contract) **MAY** do so. Such mutations are permitted only at the
  boundaries defined by OVOS-SESSION-2 §2.6 (transformer, pipeline,
  and handler boundaries); the mutation's semantics are governed by
  the field owner's specification, not this one.

### 4.1 Default materialization

The derivations of OVOS-MSG-1 §5.1–§5.3 permit an implementation to
**materialize** a default session on a derived Message when the source
Message had no `session`; this specification narrows that permission
for the field set §3 claims.

A materialized default **MUST** set `session_id: "default"`. A
materialized default **MUST NOT** populate any field whose
deployment default is a deployment-configured or "no behaviour"
value — those fields carry meaning only when explicitly set by the
session origin, and materializing them would falsely declare a
divergence from deployment defaults that the origin never requested.
Fields whose default is a fixed normative value (`session_id:
"default"`) MUST be set. Fields outside the §3 closed set remain
governed by OVOS-MSG-1 §4.1 and §5.1–§5.3 alone.

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
treat the Message as malformed. Where the failure is that `session`
parses but is not an object, §2.5 governs — including its guarantee
that the consumer **MUST NOT** substitute the default session for the
broken carrier. OVOS-MSG-1 §2 and §6 govern only the case where the
envelope itself fails to parse.

---

## 6. Conformance

### A **producer** of session-carrying Messages **MUST**:

- populate `session` as a JSON object conforming to §2;
- give `session_id` a non-empty string value when set;
- when setting any field listed in §3, use the wire type fixed by §3
  and the value space fixed by the owner specification;
- propagate `session` unchanged across the Message derivations of
  OVOS-MSG-1 §5.1–§5.3, per §4 of this specification (which raises
  OVOS-MSG-1 §4.1's SHOULD to a MUST), except when acting as
  the owner of a session field and mutating it at a permitted
  boundary (OVOS-SESSION-2 §2.6);
- not strip session fields it does not understand (§2.4, §4).

A producer **MUST NOT**:

- emit any session field with the JSON value `null` (§2); a field
  is either present with a value drawn from the owner specification's
  value space, or omitted entirely.

A producer **SHOULD NOT**:

- populate a per-component override field (§3 — `pipeline`,
  `intent_context`, the six `*_transformers`, `blacklisted_skills`,
  `blacklisted_intents`, `blacklisted_pipelines`, `site_id`) with a
  value that matches the deployment default merely as a form of
  explicit confirmation. Omit the field and let the orchestrator's
  default apply (§2.1, §3.4). Producers that cannot determine the
  deployment default are non-optimal but conformant.

### A **consumer** of session-carrying Messages **MUST**:

- treat an omitted field, an empty session object `{}`, and an
  absent `session` identically — all mean "let the orchestrator
  decide" and resolve to deployment defaults at consumption (§2.1);
- treat an explicit `null`, or a value of the wrong wire type, as a
  malformed value: behave as if the field were omitted and **SHOULD**
  log the violation (§2);
- tolerate any field it does not recognize and propagate it
  unchanged on derived Messages (§2.4, §4);
- key per-session state on `session_id`;
- not reject a Message because of the presence, absence, or value
  of any single session field — invalid values for fields whose
  owner specification defines a fallback cause that fallback, never
  Message rejection;
- survive a malformed `session` **carrier** — a `session` that is
  present but not a JSON object — without crashing or tearing down its
  transport, and **MUST NOT** substitute the default session for it
  (§2.5).

A consumer **SHOULD**:

- log unknown session fields for diagnostic purposes;
- drop (reject) a Message carrying a malformed `session` carrier and
  log the violation (§2.5).

### A specification that **claims a new session field** **MUST**:

- follow §2.2 in full — name, wire type, deployment-default, no collision;
- be self-contained: define everything the field needs in the
  claiming specification, not by reference to this one.

### Non-goals

The following are explicitly **outside** this specification and
**MUST NOT** be inferred from it: session lifecycle (creation,
expiration, end-of-session events) and session-resumption
semantics (both owned by OVOS-SESSION-2); session-store
protocols, central session indexing, session authentication and
authorization, per-field encryption, multi-tenant session
isolation guarantees beyond the opaque `session_id` keying, and
any field not claimed under §2.2 by a normative specification.

---

## See also

- **OVOS-MSG-1** — defines `Message.context` as the carrier and the
  `forward` / `reply` / `response` derivations that propagate
  `session` unchanged.
- **OVOS-PIPELINE-1** — owns `session.pipeline` and
  `session.active_handlers`.
- **OVOS-CONTEXT-1** — owns `session.intent_context`.
- **OVOS-CONVERSE-1** — owns `session.converse_handlers` and
  `session.response_mode`.
- **OVOS-TRANSFORM-1** — owns the six `session.*_transformers`
  fields.
- **OVOS-FALLBACK-1** — owns `session.fallback_handlers`.
- **OVOS-PERSONA-1** — owns `session.persona_id`.
- **OVOS-BRIDGE-1** — owns `session.site_id`.
