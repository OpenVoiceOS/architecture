# Transformer Plugins Specification

**Spec ID:** OVOS-TRANSFORM-1 · **Version:** 2 · **Status:** Draft

This document defines **transformer plugins** as an architectural
pattern of voice operating systems: ordered black-box chains of
components, inserted at well-defined points in the utterance
lifecycle, that enrich, normalize, translate, or otherwise mutate
the artifacts flowing through the assistant. The spec identifies
**six injection points** that are the natural homes for this
kind of work in a voice operating system's utterance lifecycle
(§2), defines the per-type
contract for each (§3), and specifies the shared chain abstraction
— ordering, error handling, cancellation, registration — that any
orchestrator implementing chains follows (§4, §6, §7, §8).

An orchestrator **MAY** implement transformer chains at any subset
of the six injection points (none, some, or all). For each chain
it does implement, this spec defines what the chain looks like and
what it MUST do. The spec does **not** require any specific chain
to be implemented; it defines the design pattern and the contract,
not a feature list.

It builds on three companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope and
  the `session` carrier (§4) in which per-session transformer
  overrides live;
- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the per-utterance flow into which the six
  transformer chains insert (§2 of this spec extends
  OVOS-PIPELINE-1 §6);
- the *Intent Definition Specification* (OVOS-INTENT-3) — the
  `Match` shape an intent transformer (§3.4) consumes and emits.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
and **MAY** are used as in RFC 2119.

---

## 1. What a transformer is

A **transformer** is a black-box component that consumes one
artifact at a specific point in the utterance lifecycle,
optionally mutates it, and produces an artifact of the same shape
for the next stage to consume. What a transformer does internally
is unconstrained — anything from a regex substitution to a
language-model rewrite to an audio DSP filter qualifies, provided
its IO conforms to the contract of its type (§3).

A **transformer chain** is an ordered set of transformers of one
type that the orchestrator runs at an injection point. Unlike a
pipeline plugin (OVOS-PIPELINE-1 §3) — which *decides* whether to
claim an utterance — every transformer in a chain **always runs**
when its injection point is reached. There is no claim-or-decline,
no first-result-wins, no early exit (except utterance cancellation
per §8). Whatever the last transformer returns is what the
orchestrator passes to the next lifecycle stage.

Per OVOS-PIPELINE-1 §2, the orchestrator MAY be implemented as
multiple cooperating processes. The six transformer chains
partition naturally along the audio-boundary split named there:
the audio chain (§3.1) with the audio-input service; the
utterance / metadata / intent chains (§3.2–§3.4) with the
utterance-handling service; the dialog and TTS chains (§3.5,
§3.6) with the audio-output service. Under a split, no single
process holds a global view of loaded transformers — the
introspection surface (§6) is broadcast-query / scatter-response
specifically to accommodate this. A single-process implementation
is equally conformant; the wire shape is the same either way.

### 1.1 Transformer identity

A transformer is identified by a **`(type, transformer_id)` pair**.

- **`type`** is exactly one of the six values defined by §2:
  `audio`, `utterance`, `metadata`, `intent`, `dialog`, `tts`. The
  type fixes the injection point at which the transformer runs and
  the IO contract it conforms to (§3).
- **`transformer_id`** is an opaque deployment-unique string
  within its type. The orchestrator's loaded transformers are
  partitioned by type into per-type registries
  `transformer_id → transformer instance`. When the orchestrator
  is split across multiple processes, each process holds the
  slice of those registries relevant to the chains it implements;
  the union across processes is the full loaded set.

Constraints on `transformer_id` strings:

- Non-empty.
- Must match the topic-name syntax of OVOS-MSG-1 §2.1 (ASCII
  letters, digits, `.`, `_`, `-`; no whitespace).
- Must not contain `:` (the dispatch-topic separator of
  OVOS-PIPELINE-1 §7).
- Unique within its type's registry. A single deployment MAY load
  transformers with the same `transformer_id` across different
  types; the six type registries are independent.

A transformer **MAY** appear in a chain at most once for its
type; a chain is an ordered set of distinct `transformer_id`s
within a single type.

### 1.2 Scope

This specification defines the shared chain model (§1, §4, §7),
the six injection points in the utterance lifecycle and the
per-type IO contracts (§2, §3), the per-session override mechanism
(§5), the broadcast-query / scatter-response **introspection
surface** (§6), the **utterance cancellation** plugin contract
(§8), the **language disambiguation hierarchy** for
`Message.context` (§7.1), conformance (§9), and the non-goals
(§10).

It does **not** define:

- **What any individual transformer does internally** — transformers
  are black boxes; only the IO contract at the injection point is normative.
- **How transformers are loaded, discovered, configured, or
  instantiated** — deployment concerns.
- **Slot value typing schemas.** Intent transformers (§3.4) are
  the canonical home for system-type entity injection (dates,
  numbers, durations, etc.), but the *typed value formats*
  themselves are out of scope for this specification
  (OVOS-INTENT-1 §5.3).
- **Streaming / end-to-end pipeline shapes.** The §2 flow diagram
  describes the canonical staged flow most transformers depend on
  (mic → STT → text → intent → speak → TTS → playback);
  implementations that collapse stages (streaming STT, end-to-end
  speech-to-speech models) MAY omit hooks that have no
  corresponding artifact in their flow, provided the conformance
  rules of §9 are met for every chain they do implement.

For the design rationale behind each injection point and why
transformer chains are the right architectural primitive for
cross-cutting concerns, see
[appendix/rationale.md §4.7](appendix/rationale.md).

### 1.3 Transformer self-identification

This specification claims six `Message.context` keys, one per
transformer type:

| Type (§2)   | Context key                  |
|-------------|------------------------------|
| audio       | `audio_transformer_ids`      |
| utterance   | `utterance_transformer_ids`  |
| metadata    | `metadata_transformer_ids`   |
| intent      | `intent_transformer_ids`     |
| dialog      | `dialog_transformer_ids`     |
| tts         | `tts_transformer_ids`        |

Each key, when present, holds an **ordered list of `transformer_id`
strings** (§1.1) belonging to the corresponding type's registry.
The list records the chain of transformers of that type that
touched the Message, in order of touch. The **last element** is
the current-attribution transformer; the full list records chain
provenance. The plural key name signals the list shape; the
singular `<type>_transformer_id` naming is **not** used by this
specification.

**Stamp rule.** On every Message a transformer places on the
bus by **authorial action** — a fresh emission, or
`Message.reply(...)` / `Message.response(...)` derivation it
performs and emits (OVOS-MSG-1 §5) — and on every Message it
**modifies in place** within its execution window before the
Message proceeds, the transformer **MUST** ensure that its own
`transformer_id` is the **last element** of the corresponding
`<type>_transformer_ids` list.

`Message.forward(...)` (OVOS-MSG-1 §5.1) preserves `context`
unchanged and is propagation, not authorial assertion. A
transformer that `.forward`s a Message it did not modify **MUST
NOT** append its own `transformer_id` for that derivation — the
inherited list rides through untouched. If the transformer
modifies the Message in place and then `.forward`s the modified
Message, the modify-in-place clause applies and the stamp
obligation fires.

Operationally, on every touch the transformer **appends** its
own `transformer_id` to the list (creating the list if absent or
empty). The append fires once per execution window. The six
`<type>_transformer_ids` keys coexist on a single Message with
each other and with the component-identity keys claimed by other
specifications — `context["skill_id"]` (OVOS-INTENT-4 §3.1) and
`context["pipeline_id"]` (OVOS-PIPELINE-1 §3.1), both **single
strings**. Attribution consumers that need to pick a single
emitter **SHOULD** take the identity most specific by lifecycle
position among the keys present, reading the last element of the
list-valued keys.

`<type>_transformer_ids` is the transformer chain's
**self-attribution**. It is distinct from any
`data["transformer_id"]` (singular) a topic's payload schema may
carry as the **subject** of the Message — for example, the
`transformer_id` payload field in
`ovos.transformer.{type}.list` responses (§6) identifies the
transformer the entry describes, not who emitted the response.

#### Orchestrator-side enforcement

The orchestrator (or any component that loads transformers)
**SHOULD** intercept / decorate the transformer's emit pathway and
its return-value handling at load time so non-compliant
transformer code cannot emit a Message or hand back a modified
Message whose `<type>_transformer_ids` list does not end with the
transformer's own id. The orchestrator's own bus emissions on
behalf of a transformer — the `cancel_by` stamping of §8.1, for
example — are made by the orchestrator from its own runtime
knowledge of which transformer caused the event; those emissions
carry the orchestrator's own attribution discipline, not the
transformer's.

A consumer that needs to attribute a transformer's action
**MUST** read the corresponding `<type>_transformer_ids` list
directly (typically the last element for current attribution, the
full list for chain provenance); it **MUST NOT** infer the
transformer from `source`,
from `data` fields, or from the topic name.

---

## 2. Injection points in the utterance lifecycle

This specification identifies **six injection points** in the
utterance lifecycle of OVOS-PIPELINE-1 §6 where transformer chains
are the right architectural primitive. Each injection point exists
because the lifecycle, at that exact moment, holds an artifact in
a state that makes a particular class of work possible there and
nowhere else. §3 covers each in detail; this section is the
catalogue.

The six injection points, in lifecycle order:

```
mic audio
  │
  ├─ audio-transformer chain           (§3.1)
  │
STT → text
  │
  ├─ utterance-transformer chain       (§3.2)
  │
  ├─ metadata-transformer chain        (§3.3)
  │
intent-context decay                   (OVOS-CONTEXT-1 §4)
  │
match round                            (OVOS-PIPELINE-1 §6)
  │
  ├─ intent-transformer chain          (§3.4)
  │
dispatch + handler trio                (OVOS-PIPELINE-1 §7, §8)
  │
skill emits speak()
  │
  ├─ dialog-transformer chain          (§3.5)
  │
TTS → wav file
  │
  ├─ tts-transformer chain             (§3.6)
  │
playback
```

An orchestrator **MAY** implement transformer chains at any subset
of these injection points (none, some, or all). Each chain it
implements MUST conform to the per-type contract of the matching
§3 subsection; each chain it does not implement is simply a no-op
at that point in the lifecycle. Implementations whose architecture
omits an upstream artifact entirely (a streaming STT that produces
no discrete "STT → text" boundary, an end-to-end speech-to-speech
model that bypasses intermediate text) **MAY** likewise omit the
chains for artifacts they don't materialise.

Each implemented chain is run to completion before the next stage
of the lifecycle proceeds. A chain whose transformers all raise
still produces the input unchanged (§7) and the lifecycle
continues. A chain or stage MAY be aborted early by **utterance
cancellation** (§8) — the only sanctioned way to short-circuit the
lifecycle before its natural terminal events; cancellation
preserves OVOS-PIPELINE-1 §9.5's universal `ovos.utterance.handled`
invariant.

---

## 3. Per-type contracts

For each of the six injection points, this section defines the
chain's input artifact, what the chain MAY/MUST change, and any
type-specific conformance rules. Design rationale for each injection
point — why each is the only point in the lifecycle where its class
of work is possible — is in
[appendix/rationale.md §4.7](appendix/rationale.md).

### 3.0 `lang` parameter — common contract across artifact-bearing chains

Four of the six per-type contracts (§3.1 audio, §3.2 utterance,
§3.5 dialog, §3.6 TTS) operate on an artifact whose **content
language** can be authoritative. The orchestrator threads this
language through the chain as a parameter named **`lang`**,
alongside the artifact and `Message.context`. The parameter is
**bidirectional** — it appears in both the input and the output
of each transformer call, so a transformer that mutates the
artifact's language can mutate `lang` in lockstep.

- **Source at chain start.** The orchestrator sources the initial
  `lang` from `Message.data.lang` of the Message whose artifact
  the chain is processing. `data.lang` is owned by the topic's
  spec; its presence is an authoritative declaration that the
  artifact is in that language.
- **Optional, no orchestrator-side synthesis.** `lang` is
  **OPTIONAL**. The orchestrator **MUST** pass it through when
  `Message.data.lang` is present, and **MUST NOT** synthesize a
  value when it is absent — in particular, it **MUST NOT** fall
  back to `session.lang`, to any per-utterance signal field
  (`stt_lang`, `request_lang`, `detected_lang`), or to a
  deployment default. An absent `lang` parameter is a faithful
  signal that the content language is not authoritatively known.
- **Consumer-side resolution.** A transformer that needs a
  language and receives `lang: None` **MAY** consult
  `Message.context.session` to read the user-preference signal
  (`session.lang`) or any per-utterance signal field, or fall
  back to its own default — the choice is the transformer's.
- **Output `lang` — transformer mutation.** Each transformer
  call returns a `lang` value alongside the modified artifact and
  context. The returned `lang` MAY differ from the input `lang`:
  pass-through (unchanged), set/detect (new value replacing
  `None`), translate (destination language), or clear (`None`).
- **Threading across the chain.** The orchestrator threads the
  output `(artifact, lang)` of each transformer into the input of
  the next.
- **Writeback to `data.lang`.** After the chain finishes, the
  orchestrator **MUST** reflect the final output `lang` into the
  artifact-bearing Message's `data.lang`: set `data.lang` to the
  final value when non-`None`; unset `data.lang` when the final
  value is `None` and the field was present on entry.
- **Metadata (§3.3) and intent (§3.4) transformers do not
  receive `lang` as a parameter.** Intent transformers receive a
  `Match` whose `Match.lang` (OVOS-PIPELINE-1 §4.1) already names
  the language; metadata transformers operate on `Message.context`
  only and read whichever language signal their policy calls for.

### 3.1 Audio transformers

**Injection point.** Pre-STT. Operate on raw audio chunks from the
microphone or any other audio source feeding the assistant.

**Input.** A binary audio chunk, the optional `lang` parameter
(§3.0), and a metadata object carrying at minimum the audio's
sample rate, sample width, and channel count; the metadata
object is otherwise extensible.

**Output.** A binary audio chunk, the (possibly mutated) `lang`
value per §3.0, and an updated metadata object.

**Permitted mutations.** A transformer MAY rewrite the audio
buffer (noise reduction, gain control, format conversion) and MAY
add or modify metadata keys (detected language, loudness, voice
activity score). When the transformer changes the audio's physical
format (sample rate, sample width, channel count), it **MUST**
update the corresponding metadata fields to match; conversely it
**SHOULD NOT** modify those physical-format metadata fields without
having actually changed the audio.

### 3.2 Utterance transformers

**Injection point.** Post-STT, pre-intent. Operate on the candidate
transcription list.

**Input.** A non-empty list of candidate utterance strings, the
optional `lang` parameter (§3.0), and the full **`Message.context`**
object (OVOS-MSG-1 §2.3) for the in-flight utterance — same
surface §3.3 describes, including the `session` carrier and
everything other transformers and other specifications have
written into it.

By convention `utterances[0]` is the **primary candidate** — the
canonical STT transcription, or the result of whatever upstream
chain step elected one. Later indices are alternative candidates
(STT n-best alternatives, paraphrases added by an earlier
transformer, normalized variants). Plugins that operate on a
single text MAY target `utterances[0]` only; plugins that produce
alternatives extend the list. Downstream matchers MAY try any
candidate.

**Output.** A possibly modified list of utterance strings, the
(possibly mutated) `lang` value per §3.0, and a possibly mutated
`Message.context`.

**Permitted mutations.** A transformer MAY rewrite, expand, or
contract the candidate list (add a paraphrase, drop an invalid
transcription). Mutation MAY be performed in place on the input
list or by returning a new list; both are conformant. It MAY also
mutate `Message.context` per the same permissive rules of §3.3 —
utterance transformers legitimately need to write metadata they
derived from the text (detected language, confidence rescoring),
and may mutate session-internal fields when the result of their
work warrants it (e.g. a translation transformer that normalizes
`session.lang` to the internal language after translating). The
§3.3 coordination guidance on companion-spec reserved keys
applies here equally.

> **Empty-list semantics.** A transformer MAY return an empty list.
Two distinct outcomes share this shape: (1) **no plausible
transcription** — empty list without the §8.1 cancellation signal;
downstream stages treat it as silence and the lifecycle terminates
with `ovos.intent.unmatched` followed by `ovos.utterance.handled`
per OVOS-PIPELINE-1 §9; (2) **cancellation** — empty list returned
together with `canceled: true` and `cancel_reason` per §8.1; the
orchestrator terminates via the §8.2 path, emitting
`ovos.utterance.cancelled` followed by `ovos.utterance.handled`.
A transformer that wants the cancellation outcome **MUST** set the
§8.1 keys; returning an empty list alone is the no-transcription
case.

### 3.3 Metadata transformers

**Injection point.** Post-utterance, pre-intent. The metadata-
transformer chain operates directly on the **`Message.context`
object** (OVOS-MSG-1 §2.3) for the in-flight utterance — including
the session carrier (OVOS-MSG-1 §4), accumulated context from prior
transformers, and any other context keys other specifications have
populated. A metadata transformer's defining trait is that its only
input *and* its only output is `Message.context`; it has no
artifact-specific input the way audio (§3.1), utterance (§3.2),
intent (§3.4), dialog (§3.5), or TTS (§3.6) transformers do.

**Input.** The full `Message.context` object (OVOS-MSG-1 §2.3) for
the in-flight utterance: routing keys (§3 of MSG-1), the `session`
carrier (§4 of MSG-1, which itself carries `session.intent_context`
(OVOS-CONTEXT-1 §2), `session.pipeline` (OVOS-PIPELINE-1 §5), the
six per-session transformer overrides (§5 of this spec), and any
other normative or non-normative internal session fields), plus
any top-level metadata keys earlier transformers or other
specifications have written.

**Output.** A `Message.context` object — in practice the input
mutated in place, or a returned replacement of the same shape.

**Permitted mutations.** A metadata transformer **MAY** mutate
`Message.context` however it sees fit. That is its purview, by
design: the chain exists to give a deployer a single in-process
place to manipulate per-message context unrestricted. This
includes:

- adding, updating, or removing top-level keys in `Message.context`;
- mutating session-internal fields directly: writing entries to
  `session.intent_context` (OVOS-CONTEXT-1 §2), reordering or replacing
  `session.pipeline` (OVOS-PIPELINE-1 §5), mutating the
  active-handler list `session.active_handlers` or the response-mode
  holder `session.response_mode` (OVOS-CONVERSE-1 §3.3 explicitly
  cites the metadata-transformer hook as the recommended position
  for such mutations, and §5.3 there fixes the cancellation
  semantics when a transformer mutation removes or replaces the
  current response-mode holder), changing `session.lang`
  (OVOS-MSG-1 §4.2), overriding the six per-session transformer
  chains (§5 of this spec) for this utterance, or any other field
  on `session`;
- adjusting routing keys `source` / `destination` (OVOS-MSG-1 §3).
  Routing-key mutation is a load-bearing change that affects every
  downstream `forward`/`reply`/`response` derivation and is the
  attachment point layer-2 substrates build on (OVOS-MSG-1 §3.4).
  A metadata transformer **SHOULD NOT** mutate `source` or
  `destination` unless the transformer's deliberate role is
  re-routing this lifecycle (e.g. an authorization-rewrite
  transformer); a transformer that mutates routing keys **MUST**
  understand the OVOS-MSG-1 §5 derivation consequences for every
  emission downstream of this stage.

The spec **does not police** what a metadata transformer mutates.
A deployer who loaded a particular metadata transformer has
implicitly authorized whatever it does to `Message.context`. A
consumer trying to attribute an unexpected context key to its
source uses the introspection surface of §6 (the set of loaded
metadata transformers) and the chain order — these together name
the universe of candidates deterministically.

> **Informative — mutations with cross-spec consequences.**
> Mutating certain reserved keys has effects that spec readers
> should be aware of even though they are not prohibited:
>
> - Mutating `session.intent_context` directly is the sanctioned
>   transformer pathway of OVOS-CONTEXT-1 §5.2; the key-shape
>   rules of OVOS-CONTEXT-1 §3 apply to every entry written.
> - Mutating `session.pipeline` (OVOS-PIPELINE-1 §5) changes which
>   pipeline plugins are consulted for this utterance — a powerful
>   per-utterance routing primitive that is also easy to misuse.
> - Mutating session-level language signals (OVOS-SESSION-1 §3.2)
>   changes how subsequent stages localize.
> - Mutating `source` / `destination` (OVOS-MSG-1 §3) changes
>   routing for downstream Message derivations
>   (`forward`/`reply`/`response`).

### 3.4 Intent transformers

**Injection point.** Post-match, pre-handler-dispatch. Operate on
the `Match` object that a pipeline plugin produced
(OVOS-PIPELINE-1 §4.1) before the orchestrator emits the dispatch
Message (OVOS-PIPELINE-1 §7). Two things happen in this window —
**engine-side session mutation** per OVOS-CONTEXT-1 §5.1 and the
intent-transformer chain of this section — and the **engine-side
mutation MUST happen first**. The orchestrator accepts the match,
allows the matching engine to write any context entries it intends
to per CONTEXT-1 §5.1, and only then runs the intent-transformer
chain over the resulting `Match`. This ordering lets an intent
transformer read context the matching engine just wrote (for
example, to enrich a slot based on a freshly-promoted entry).

**Input.** The `Match` produced by the pipeline plugin that
claimed the utterance — `skill_id`, `intent_name`, `slots`,
`utterance` (OVOS-PIPELINE-1 §4.1) — together with the post-engine-
mutation `session.intent_context` snapshot.

**Output.** A `Match` of the same shape, possibly with an enriched
`slots` map.

**Permitted mutations.** A transformer MAY add entries to
`Match.slots` and MAY overwrite existing entries it itself
produced earlier in the chain. It **SHOULD NOT** delete or
overwrite slot entries produced by the matching engine or by an
earlier transformer in the chain, unless deletion is the
transformer's deployer-configured purpose (PII redaction,
content filtering, profanity censoring). It **MUST NOT** change
`Match.skill_id` or `Match.intent_name` — those identify the
dispatch topic (OVOS-PIPELINE-1 §7), and changing them would route
the handler elsewhere than the engine that matched intended.

**Orchestrator enforcement of identity invariants.** If a
transformer returns a `Match` whose `skill_id` or `intent_name`
differs from its input, the orchestrator **MUST** treat the return
as a shape violation per §7 — discard the transformer's output and
proceed with the prior step's `Match` unchanged. This is the
orchestrator-side safety net for the MUST NOT above.

### 3.5 Dialog transformers

**Injection point.** Post-skill, pre-TTS. Operate on the rendered
dialog string a skill emitted (typically via a `speak` event),
before it becomes synthesized audio.

**Input.** The dialog string, the optional `lang` parameter
(§3.0), and the full **`Message.context`** object (OVOS-MSG-1 §2.3)
carrying the session and any per-message
context written by earlier lifecycle stages. Same surface §3.3
describes.

**Output.** A possibly modified dialog string, the (possibly
mutated) `lang` value per §3.0, and a possibly mutated
`Message.context`.

**Permitted mutations.** A transformer MAY rewrite the dialog
string entirely (translation, persona, simplification, length cap).
It MAY also mutate `Message.context` per the same permissive rules
of §3.3 — common cases include setting a `voice_id` hint for a
downstream TTS transformer, restoring `session.lang` to the user's
preferred language after a temporary mid-lifecycle override, or
writing the rewriter's choices into context for downstream
observability. The §3.3 coordination guidance on companion-spec
reserved keys applies here equally.

### 3.6 TTS transformers

**Injection point.** Post-TTS, pre-playback. Operate on the
synthesized audio file the TTS engine produced, before the playback
subsystem consumes it.

**Input.** A path or handle to the synthesized audio, the
optional `lang` parameter (§3.0), and the full
**`Message.context`** object (OVOS-MSG-1 §2.3). Same surface
§3.3 describes.

**Output.** A path or handle to the (possibly replaced)
synthesized audio, the (possibly mutated) `lang` value per §3.0,
and a possibly mutated `Message.context`.

**Permitted mutations.** A transformer MAY replace the audio with
a transformed version (pitch shift, reverb, EQ, tempo, format
conversion, watermarking, insertion of jingles or earcons). It
**SHOULD NOT** silently re-synthesize the speech in a different
language or with different content — translation and rewriting are
dialog-transformer (§3.5) concerns, performed against the text
before TTS; performing them again on the synthesized audio
defeats the staging. The transformer MAY also mutate
`Message.context` per the same permissive rules of §3.3 — for
example writing playback metadata (final audio format, duration,
applied effects) for observability.

---

## 4. Chain ordering

A chain runs in **ascending priority order**: a transformer with
`priority = 1` runs before one with `priority = 50` runs before one
with `priority = 100`. Lower number = earlier in the chain. This
matches the natural "stages count up" reading.

Each transformer plugin declares an integer `priority`. The
default is `50` — the middle of the band — so plugins with no
opinion sit between explicitly-early and explicitly-late
transformers.

Two ordering mechanisms are defined; deployers choose:

- **Priority-based (default).** The orchestrator sorts the loaded
  set ascending by `priority` and runs the resulting chain. Ties
  are broken in a stable but unspecified order — chain authors who
  care about relative ordering between two transformers **SHOULD**
  give them distinct priorities.
- **Explicit deployer order.** Deployer configuration supplies an
  ordered list of `transformer_id`s for the chain. The orchestrator
  runs them in that order, ignoring declared priorities. Explicit
  order wins over priority. Transformers loaded but absent from the
  explicit list are not run at this hook.

The orchestrator **MUST** support both mechanisms and **MUST**
apply explicit order when configured.

Priorities are meaningful only under ascending order. A priority
assignment authored for the inverse convention — descending order,
where the lowest number runs *last* and, because each transformer's
output overwrites its predecessor's, effectively "wins" — is the
exact inverse of a conformant one and **MUST** be renumbered; run
unrenumbered, such a chain executes backwards.

---

## 5. Per-session overrides

This specification claims **twelve session fields** under
OVOS-SESSION-1 §2.2: six **preference** fields naming a per-type
chain ordering (§5.1) and six **policy** fields naming a per-type
denylist (§5.2). The composition rule of §5.3 layers them.

All six preference fields propagate unchanged per OVOS-MSG-1 §4.1
and are session-scoped; in the absence of a field, the
deployer-configured default chain for that type is used.

### 5.1 Per-type chain ordering — `<type>_transformers`

Six session fields, one per injection point, expressing the
**session origin's preferred chain** for that type:

| Field | Chain | Wire type | Deployment default (absence) |
|-------|-------|-----------|------------------------------|
| `session.audio_transformers` | §3.1 | array of string (`transformer_id`) | the deployer-configured audio chain for this orchestrator process |
| `session.utterance_transformers` | §3.2 | array of string (`transformer_id`) | the deployer-configured utterance chain |
| `session.metadata_transformers` | §3.3 | array of string (`transformer_id`) | the deployer-configured metadata chain |
| `session.intent_transformers` | §3.4 | array of string (`transformer_id`) | the deployer-configured intent chain |
| `session.dialog_transformers` | §3.5 | array of string (`transformer_id`) | the deployer-configured dialog chain |
| `session.tts_transformers` | §3.6 | array of string (`transformer_id`) | the deployer-configured TTS chain |

Each field is OPTIONAL on the wire. An omitted, empty, or absent
field resolves at consumption to the deployment default for that
hook per OVOS-SESSION-1 §2.2. An empty array (`[]`) is wire-
equivalent to omission for every field in the table above. Per
the canonical wire-weight rule of OVOS-SESSION-1 §3.4, a producer
**SHOULD** omit any of these fields whose value matches the
deployment default — including the empty-array case where the
deployment default is to run no transformers of that type — rather
than emit a redundant value.

The fields are a **preference channel**: any session origin
(local, remote, layer-2-attached, programmatic) MAY populate them
to request a specific chain ordering. The orchestrator narrows the
request by what is loaded and what policy permits, per §5.3.

Different sessions may carry different chains. This is how a
deployment provides differentiated behaviour per participant — for
example, a remote-peer session may request restricted chains
tailored to its participant. Whether the preference is honoured is
a policy decision (§5.3).

The plugin **instances** stay process-wide. Per-session chains are
per-session *orderings over the loaded set*, not per-session
instantiation.

### 5.2 Per-type denylists — `blacklisted_<type>_transformers`

Six session fields, one per injection point, expressing the
**policy channel** for transformer selection:

| Field | Chain |
|-------|-------|
| `session.blacklisted_audio_transformers` | §3.1 |
| `session.blacklisted_utterance_transformers` | §3.2 |
| `session.blacklisted_metadata_transformers` | §3.3 |
| `session.blacklisted_intent_transformers` | §3.4 |
| `session.blacklisted_dialog_transformers` | §3.5 |
| `session.blacklisted_tts_transformers` | §3.6 |

Each field is an unordered array of `transformer_id` strings of
the corresponding type's registry. Wire type, propagation, and
absence semantics match the chain-ordering fields of §5.1: array
of string, propagates unchanged, OPTIONAL on the wire, `[]`
wire-equivalent to omission, SHOULD-omit per OVOS-SESSION-1 §3.4
when no transformer is to be denied.

A transformer whose `transformer_id` is listed in the corresponding
`blacklisted_<type>_transformers` for this session **MUST NOT** be
invoked by the orchestrator for that injection point on that
session — **even if the same `transformer_id` is requested in the
corresponding `<type>_transformers` chain-ordering field of §5.1**.
Policy overrides preference (§5.3).

Filtering is **orchestrator-only** — a single-tier rule. When the
orchestrator composes the effective chain for the injection point
(per §5.3), it skips any denied `transformer_id` as if it were not
loaded. No `transform` call is made; no bus event is emitted for
the skip. The filtering is observable only as a non-invocation. The
two-tier shape used by PIPELINE-1 §5.3 / §5.4 for skill / intent
denylists has no analogue here because transformers do not return
match candidates — the orchestrator drives the chain directly.

Unknown `transformer_id`s in the denylist are harmless and
**MUST NOT** cause the utterance to abort — they simply match
nothing.

### 5.3 Composition: preference, availability, policy

For each of the six injection points, the orchestrator composes
the **effective chain** for an utterance in a fixed three-stage
order, mirroring OVOS-PIPELINE-1 §5.5:

1. **Preference.** Start from the corresponding
   `<type>_transformers` field if set and non-empty; otherwise
   start from the deployer-configured default chain for that
   injection point (§4).
2. **Availability.** Drop any `transformer_id` that does not
   correspond to a transformer loaded for this type. Unknown
   identifiers do not abort the utterance and do not trigger
   fallback to the deployer default — the remaining known
   identifiers are the effective ordered set.
3. **Policy.** Drop any `transformer_id` listed in the
   corresponding `blacklisted_<type>_transformers`, even if it
   was explicitly requested in step 1. Policy overrides
   preference.

The result is the ordered list of transformers the orchestrator
invokes at that injection point for this utterance.

The effective chain for **each** injection point is composed
**once per utterance lifecycle**, from the session as committed at
utterance start. Mid-lifecycle mutations of the
`<type>_transformers` or `blacklisted_<type>_transformers` fields —
by a transformer, a match-phase `updated_session`, or a handler —
take effect on the **next** utterance, not the current one.
Composing each chain from a live session would let an earlier chain
rewrite which later chains run for the same utterance, making the
lifecycle's transformer surface depend on mutation timing rather
than on the state the utterance arrived with.

If every requested `transformer_id` is dropped by availability or
policy, the effective chain is empty for that injection point and
the orchestrator simply runs no transformers at that stage — the
artifact passes through unmodified to the next lifecycle stage.
This is consistent with §9's null-implementation conformance:
running zero transformers at a chain is always valid.

The intended separation of concerns mirrors PIPELINE-1 §5.6:

- **Any session origin** MAY populate `<type>_transformers` to
  request a preferred chain. No authorization implied.
- **Only policy** — the denylists of §5.2, typically populated by
  the orchestrator owner or by a layer-2 substrate that owns the
  session — can refuse a transformer the preference layer asked
  for. The two channels are layered, not alternatives.

This is the same authorization surface OVOS-PIPELINE-1 §5.6
describes for pipeline plugins, extended to the transformer
chains: a layer-2 substrate that grants per-peer permissions
populates the relevant denylists from the peer's grant, and the
orchestrator's §5.3 composition enforces the policy without any
per-hop re-authorization.

---

## 6. Introspection — broadcast queries, scatter responses

The orchestrator's loaded transformers may be split across
multiple cooperating orchestrator processes (§1) — typically along
the audio-input / utterance-handling / audio-output boundary. No
single process holds the global picture. Introspection therefore
follows a **broadcast-query / scatter-response** pattern: the
requester emits a query; every orchestrator process that has
loaded transformers of the queried type responds with its own
local slice; the requester aggregates if it wants a global
picture. Deployments that run the orchestrator as a single
process answer fully from one reply.

Six per-type query/response topic pairs, one per chain type:

| Topic | Reply | Scope |
|-------|-------|-------|
| `ovos.transformer.audio.list` | `ovos.transformer.audio.list.response` | Audio chain (§3.1) |
| `ovos.transformer.utterance.list` | `ovos.transformer.utterance.list.response` | Utterance chain (§3.2) |
| `ovos.transformer.metadata.list` | `ovos.transformer.metadata.list.response` | Metadata chain (§3.3) |
| `ovos.transformer.intent.list` | `ovos.transformer.intent.list.response` | Intent chain (§3.4) |
| `ovos.transformer.dialog.list` | `ovos.transformer.dialog.list.response` | Dialog chain (§3.5) |
| `ovos.transformer.tts.list` | `ovos.transformer.tts.list.response` | TTS chain (§3.6) |

There is deliberately no aggregate "give me everything" query; a
consumer that wants all six types issues six queries.

Each query takes no payload. Each `.response` (OVOS-MSG-1 §5.3
reply convention) carries one orchestrator process's own slice:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `loaded` | array of strings | yes | The `transformer_id`s this responding process has loaded for this type. |
| `priorities` | object (string→integer) | yes | The declared priority of every `transformer_id` in `loaded`. Priorities are intrinsic to the plugin and always returned. |

A `.response` carries **only the responder's local view**. It
does **not** report a global chain order — chain composition is
the §4 priority order plus the §5 per-session override applied
across the union of responses, and any aggregating consumer (a
developer tool, a monitoring service) is responsible for
combining the slices.

**Response aggregation.** A requester that wants the full picture
collects responses arriving on the corresponding `.response` topic
within an implementation-defined window. The bus is async; there
is no completeness signal. A requester that needs guaranteed
completeness must keep its own roster of expected responders
(via service-discovery means out of scope here) and time out
non-responders.

**Pull-query is the source of truth.** Each orchestrator process
**MUST** subscribe to the relevant `ovos.transformer.{type}.list`
topics — one per chain it implements — and respond with its
local slice. A consumer that needs accurate state **MUST** query
and **MUST NOT** assume any prior announcement reached it — load
ordering between producers and consumers on the bus is not
guaranteed (a consumer that starts after a producer's announcement
fired has missed it; the bus is async and has no catch-up channel
for missed broadcasts).

**Optional load-time announcements.** On load, an orchestrator
process **MAY** volunteer a one-shot announcement on the
corresponding `.response` topic, with the same shape it would
return to a pull query. This is a convenience for consumers that
happen to be listening already (a monitoring service subscribed
before the orchestrator process came online). Announcements are
**not normative** and consumers **MUST NOT** rely on receiving
them. Processes that do not announce are fully conformant;
consumers that ignore announcements and only act on query
responses are equally so.

A process that comes online answers subsequent queries; one that
goes offline simply disappears from subsequent aggregations.

---

## 7. Error handling

A transformer that raises is treated as if it returned its input
unchanged. The orchestrator **MUST** catch the exception,
**SHOULD** log it, and **MUST** proceed to the next transformer in
the chain. A single transformer's bug **MUST NOT** abort the
utterance — same posture as OVOS-PIPELINE-1 §6.2 for pipeline
plugin exceptions. Logging is **SHOULD** rather than **MUST**
because logging policy is a deployment concern (embedded targets,
regulated environments) and the catch-and-proceed behaviour is the
load-bearing contract.

A transformer that returns an output of the wrong shape — wrong
type, missing required field, list shrunk to empty for a non-empty
input — is treated the same as a raised exception: the orchestrator
**SHOULD** log and **MUST** proceed with the prior transformer's
output as if this transformer had returned its input unchanged.

Timeouts and per-transformer execution limits are
**implementation-defined**. Deployers concerned about a slow
transformer blocking the lifecycle **SHOULD** configure timeouts
at the orchestrator level; this specification does not prescribe a
default.

**Concurrency.** A transformer instance is process-wide and **MAY**
be invoked concurrently by the orchestrator for utterances in
different sessions. Transformers **MUST** be re-entrant: any
per-utterance state lives in the artifact and context passed
through `transform`, not in the transformer instance. Implementations
that need per-instance state (loaded models, caches, opened sockets)
**MUST** guard it for concurrent access.

**No rollback on partial chain failure.** Side effects a transformer
performs through other bus events (intent context mutations per
OVOS-CONTEXT-1 §5, telemetry emissions, external HTTP calls)
**MUST NOT** be rolled back by the orchestrator if a later
transformer in the chain raises or signals cancellation (§8). The
chain is a best-effort enrichment pipeline, not a transaction. A
transformer that needs all-or-nothing semantics must implement them
internally (e.g. stage its mutations and apply them only at chain
end via a final commit step).

**Mid-lifecycle session mutations propagate via `Message.context`.**
When a transformer mutates the `session` carrier inside
`Message.context` (`session.lang`, `session.pipeline`,
`session.intent_context`, etc., per §3.2 / §3.3 / §3.5 permissions), the
mutated session rides forward as part of `Message.context` to
every downstream stage that reads it. Downstream consumers
**MUST** read live session values from the in-flight
`Message.context` rather than caching session state from an
earlier observation; this is what makes mid-lifecycle session
mutation work uniformly across transformer chains, intent
matching, dispatch (OVOS-PIPELINE-1 §7), and skill handlers.

**Cross-transformer coordination via context keys.** Transformers
that need to coordinate (a bidirectional translator's input half
signalling its output half; a metadata transformer writing a hint
a later intent transformer will consume) communicate through
top-level keys in `Message.context`. To avoid collisions between
unrelated plugins, transformers **SHOULD** namespace their
ad-hoc coordination keys with their `transformer_id` (or a
related stable identifier) as a prefix —
e.g. `<transformer_id>.output_lang` rather than
bare `output_lang`. The spec defines no central registry for
context-key names; namespacing is the discipline that makes the
absence of a registry safe.

### 7.1 Language signals produced by transformers

Several injection points are natural producers of session-level
language signals defined by OVOS-SESSION-1 §3.2:

- **§3.1 audio transformers** are the natural source for
  `session.detected_lang` derived from acoustic features. An audio
  language detector writes `session.detected_lang` after running.
- **§3.2 utterance transformers** **MAY** refine
  `session.detected_lang` from text characteristics (script,
  function-word density). They **MAY** also overwrite
  `session.lang` directly per §3.2's mutation permissions if a
  confident classification warrants persisting the change beyond
  this utterance.
- **§3.3 metadata transformers** are the catch-all for any further
  language-classification refinement; the chain runs after
  utterance transformers so it sees the cumulative signal.

How a downstream consumer **consolidates** the available language
signals into a single value for any given operation is not
prescribed by this specification — see OVOS-SESSION-1 §3.2.7 for
the informative default ordering. Transformers that produce
signals **MUST NOT** assume any particular consolidation policy on
the part of consumers; they populate the appropriate session field
and leave consumption to the operation that needs it.

---

## 8. Utterance cancellation

The lifecycle MAY be aborted early — before reaching its natural
terminal events — by a transformer in any of the six chains
signalling **utterance cancellation**. Cancellation is the only
sanctioned short-circuit defined by this specification.

Cancellation is **always signalled by a transformer plugin**. There
is no bus event a third party can send to request it; the
orchestrator owns the cancellation machinery and exposes the
signal only as a plugin contract. A deployment that wants
out-of-band cancellation (a hardware stop button, a caller-side
abort signal, a barge-in from another channel) ships an
appropriately scoped transformer that watches for the trigger and
sets the cancellation signal from inside the chain — keeping the
trigger surface a deployment concern and the contract a plugin
concern.

### 8.1 The cancellation signal — `canceled` / `cancel_reason`

A transformer **MAY** signal cancellation by setting two reserved
keys in the context object it returns:

```json
"canceled": true,
"cancel_reason": "<short string describing why>"
```

Both keys **MUST** be present together when cancellation is being
signalled. `canceled` is the boolean flag the orchestrator
recognises; `cancel_reason` is a short string identifying the
cancellation reason. A context with `canceled: true` but no
`cancel_reason`, or with `cancel_reason` set but `canceled` absent
or false, is treated as a §7 shape violation; the orchestrator
**SHOULD** log and **MUST** proceed as if the transformer returned
its input unchanged.

**[Informative] `cancel_reason` vocabulary.** Downstream consumers
of `ovos.utterance.cancelled` — analytics, audit, transcript
viewers, end-user diagnostics — benefit when the reason field
draws from a stable shared vocabulary rather than free-form
strings. This specification mints the following reserved values;
a transformer **SHOULD** use one of them when its reason fits:

| Value | Meaning |
|-------|---------|
| `stop_word` | A stop / cancel keyword was detected in the utterance. |
| `transcription_invalid` | STT output was deemed unusable (garbage, low confidence, validation failure). |
| `policy_block` | A content / safety / authorization policy refused the utterance or response. |
| `parental_control` | A parental-control or restricted-mode guard refused. |
| `other` | Universal fallback for reasons that don't fit a reserved value. |

A transformer with a more specific reason than any of the above
**MAY** emit a free-form string; deployers are encouraged to
coordinate vocabulary across their loaded transformers. A
transformer that doesn't want to think about vocabulary
**SHOULD** use `other`. The orchestrator **MUST NOT** rewrite
or normalize `cancel_reason`; it propagates whatever value the
transformer set.

A transformer MAY additionally set other top-level context keys
carrying plugin-specific cancellation metadata (the matched cue,
a confidence score, a sentinel identifying the cancellation
source) — those are not part of this specification and
transformers SHOULD namespace them per §7's coordination guidance.

The orchestrator **MUST** stamp a third key automatically when it
observes a cancellation signal:

```json
"cancel_by": "<emitting transformer_id>"
```

Stamped from the transformer that produced the signal (the
orchestrator knows which one), **not** from any value the
transformer included in the payload. The orchestrator-side stamp
guarantees that a transformer cannot impersonate another
transformer's cancellation.

When `canceled: true` is observed alongside an empty utterance
list (§3.2) or any other artifact, the cancellation flag is the
signal — the empty list is a convention, not the trigger.

On observing the signal:

1. The orchestrator **MUST** stop running the current chain — no
   further transformers in this chain are invoked.
2. It **MUST** skip every subsequent injection-point chain in §2
   that has not yet started, including any chain belonging to a
   downstream stage the orchestrator implements.
3. It **MUST** terminate the lifecycle per §8.2.

The orchestrator **MUST NOT** strip or modify the `canceled` /
`cancel_reason` / `cancel_by` keys between transformers — a later
observer of the cancelled utterance's Messages (debugger,
analytics) sees that it was cancelled, why, and by whom.

### 8.2 Terminal events on cancellation

On cancellation, the orchestrator **MUST** terminate the
lifecycle with:

```
ovos.utterance.cancelled    (new; defined here)
ovos.utterance.handled      (OVOS-PIPELINE-1 §9.5)
```

emitted in that order. `ovos.utterance.cancelled` carries the
`cancel_reason` and orchestrator-stamped `cancel_by` from the §8.1
signal that triggered the cancellation. `ovos.utterance.handled`
preserves the universal end-marker invariant of OVOS-PIPELINE-1
§9.5.

The orchestrator **MUST NOT** emit `ovos.intent.unmatched`
(OVOS-PIPELINE-1 §9.3) on the cancellation path — failure and
cancellation are distinct outcomes; an observer that wants to
count "user gave up" or "policy blocked it" separately from
"matcher found nothing" needs them distinguishable on the bus.

The orchestrator **MUST NOT** dispatch any handler whose match
preceded the cancellation in the same dispatch sequence. An intent
transformer (§3.4) runs after the orchestrator accepted the match
but before dispatch (OVOS-PIPELINE-1 §6); an intent transformer
that cancels preempts the dispatch entirely.

Side effects performed by earlier transformers in the same
lifecycle (intent context mutations per OVOS-CONTEXT-1 §5,
telemetry emissions, external HTTP calls) are **not** rolled back
by cancellation — consistent with §7's no-rollback rule. The
cancellation aborts what hasn't run yet; it does not unwind what
has.

---

## 9. Conformance

An orchestrator **MAY** implement transformer chains at any subset
of the six injection points of §2 (including none). The conformance
rules below apply per chain — for each chain the orchestrator
implements, all of the corresponding obligations bind; for chains
the orchestrator does not implement, no obligations arise.

**An orchestrator that implements one or more transformer chains**
**MUST**, for each chain it implements:

- run the chain to completion at its injection point before the
  next stage of the lifecycle proceeds (§1, §2);
- order the chain by §4 — ascending priority by default, or the
  explicit deployer-configured order when one is present;
- apply per-session chain overrides (§5) when the session carries
  a non-empty corresponding `session.*_transformers` field,
  falling back to the deployer-configured chain otherwise;
- catch transformer exceptions and shape-violations, log them,
  and proceed with the prior transformer's output (§7);
- inspect the context object after every transformer for the
  `canceled` flag (§8.1) and terminate the lifecycle per §8.2 when
  set, skipping every subsequent chain in §2 of this spec that
  has not yet started; **MUST** stamp `cancel_by` from the
  emitting transformer's `transformer_id` on observing the signal;
- on any cancellation, emit `ovos.utterance.cancelled` followed
  by `ovos.utterance.handled` (§8.2), carrying `cancel_reason` and
  the stamped `cancel_by`, and **MUST NOT** emit
  `ovos.intent.unmatched` on the cancellation path; **MUST NOT**
  strip the `canceled` / `cancel_reason` / `cancel_by` keys from
  `Message.context` on the terminal events or downstream
  derivations; **MUST NOT** dispatch a Match that was reached
  before cancellation.

When the orchestrator is implemented as a single process, the
introspection obligations of §6 are met by that process. When the
orchestrator is split (§1) across cooperating processes — typically
along the audio-input / utterance-handling / audio-output boundary
— **each process** that implements one or more chains MUST meet the
per-process introspection obligations below for the chains it
implements. The composition of all such per-process responses is
the orchestrator's full view.

Additionally, an orchestrator that implements the **intent
transformer chain** (§3.4) **MUST** enforce the §3.4 identity
invariants on transformer output, treating `skill_id` /
`intent_name` changes as §7 shape violations.

An orchestrator that implements **none** of the six chains is a
conformant null-implementation of this specification — it has no
obligations under §9 and exposes none of the artefacts (per-type
queries, override fields, cancellation handling) that depend on
implemented chains. Such an orchestrator simply does not offer
transformer extensibility at the points this specification
covers.

**Each orchestrator process** that implements one or more chains
**MUST**:

- subscribe to the relevant `ovos.transformer.{type}.list` query
  topics — one per chain it implements — and respond on the
  corresponding `.response` topic (§6) with **its own local
  slice** of loaded `transformer_id`s and their declared
  priorities — never invent entries for transformers it has not
  loaded.

**Each orchestrator process** **MAY**:

- volunteer a one-shot load-time announcement on the corresponding
  `.response` topic (§6) with the same shape it would return to a
  pull query. Announcements are not normative; consumers MUST NOT
  rely on receiving them.

**Consumers of the introspection surface** **MUST**:

- query `ovos.transformer.{type}.list` (one per chain type they care
  about) when they need accurate state; **MUST NOT** assume any
  prior announcement reached them (load ordering between producer
  and consumer is not guaranteed — §6).

**A transformer** (the plugin itself) **MUST**:

- conform to its type's IO contract (§3): consume the input shape,
  produce the output shape, observe the type's MAY/MUST NOT rules
  on permitted mutations;
- be re-entrant — the host may invoke it concurrently for
  utterances in different sessions, and any per-instance state
  must be guarded for concurrent access (§7);
- declare an integer `priority` (§4); the value `50` is the
  conventional middle-of-the-band default;
- when signalling cancellation (§8.1), set both `canceled: true`
  and `cancel_reason: <reason>` in the returned context; the
  orchestrator will stamp `cancel_by` from the emitting
  transformer's `transformer_id`.

**A transformer** **MAY**:

- read and mutate `session.intent_context` (OVOS-CONTEXT-1 §2)
  directly on the session object it holds in hand. The direct-
  mutation pathway is normatively permitted for any transformer
  type by OVOS-CONTEXT-1 §5.2 — the orchestrator is the carrier
  of writes, not the bus. When mutating, the transformer **MUST**
  use the key-shape rules of OVOS-CONTEXT-1 §3 and §5.2 (private
  entries prefixed `<skill_id>:`, where `<skill_id>` for a
  transformer is its own `transformer_id` or, when the transformer
  is writing on behalf of a specific skill, that skill's
  `skill_id`). Mutations made via the `ovos.session.sync` pathway
  (OVOS-CONTEXT-1 §5.3) are also permitted; the choice between
  the in-place pathway and the sync pathway is the transformer's;
- access the bus for side-effects unrelated to the transformer's
  IO (logging, telemetry, cross-session signals) — but **SHOULD
  NOT** make the transformer's output depend on bus responses
  fetched synchronously inside `transform`, as this serializes the
  lifecycle on the bus's responsiveness. Every such bus emission
  **MUST** ensure the appropriate `<type>_transformer_ids` list
  in `Message.context` ends with the transformer's own id per
  §1.3.

**An observer** that sees `Message.context` carrying `canceled:
true` or `cancel_reason`:

- **MUST NOT** attempt to cancel the utterance by emitting bus
  events — cancellation is a transformer-plugin contract only
  (§8);
- **MAY** read `cancel_reason` and `cancel_by` for audit,
  analytics, or observational purposes.

---

## 10. Non-goals

- **Slot value typing schemas.** Intent transformers (§3.4) are
  where typed system entities are injected, but the typed value
  formats themselves (date encoding, number representation,
  duration units) are out of scope for this specification
  (OVOS-INTENT-1 §5.3). This spec defines the injection pathway
  only, not what gets injected.
- **Behavioural contracts for any specific transformer type beyond
  the IO shape and the canonical use-case list.** Whether an
  utterance transformer normalizes contractions, translates,
  validates STT — that is per-plugin behaviour, not spec-level
  contract. This spec covers only the *frame* every transformer
  runs in.
- **Cross-transformer coordination protocols.** Transformers do not
  see each other's prior outputs except through the artifact they
  pass forward. There is no shared scratch space, no
  transformer-to-transformer messaging, no inheritance hierarchy.
  Coordination, when it is needed, happens through the artifact
  (the utterance list, the context object, the `Match`).
- **Loading, discovery, instantiation, configuration management.**
  Deployment concerns; out of scope.
- **Mandating any specific chain be implemented.** This spec
  defines the architectural pattern and the per-chain contract;
  it does not require any orchestrator to implement any
  particular chain. A null-implementation that runs no chains is
  conformant (§9). Which chains a given orchestrator implements is
  a deployment decision.
- **Out-of-band cancellation channels.** Cancellation is exclusively
  a transformer-plugin contract (§8); the orchestrator owns the
  cancellation machinery and exposes the trigger only via the §8.1
  context flag. Deployments that want hardware buttons, peer
  signals, or barge-in to cancel an in-flight utterance ship a
  thin transformer that watches for the trigger and sets the
  cancellation signal from within the chain. The bus has no
  third-party cancel topic.
- **Hot reload of transformer chains.** Whether and how an
  orchestrator can swap a transformer chain at runtime is an
  implementation concern.
- **Timeouts and execution limits per transformer.** Recommended
  for production deployments (§7) but not specified.
- **Wire-level invocation messages across orchestrator processes.**
  When the orchestrator is split across cooperating processes
  (§1), one process may invoke a transformer loaded by another
  process. This specification defines the introspection surface
  (§6) and the IO contracts (§3) any invocation MUST satisfy, but
  does not prescribe a specific `transformer.{type}.invoke`
  request / response topic shape. A single-process orchestrator
  needs no such surface; a split orchestrator requires one, and
  deployments adopt whatever request / response convention fits
  their substrate.

---

## See also

- *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the per-utterance flow into which §2 of this
  spec inserts the six transformer hooks; the `Match` shape §3.4
  consumes.
- *Bus Message Specification* (OVOS-MSG-1) — the `session` carrier
  (§4), the shared identifier-component rule (§2.1.1) bounding
  `transformer_id`, and the `.response` reply convention (§5.3) the
  §6 query events follow.
- *Session Specification* (OVOS-SESSION-1) — the wire shape of
  `session`, the registry mechanism under which this specification
  claims the six per-session transformer-override fields (§5), and
  the deployment-default fallback rule for omitted fields.
- *Intent Context Specification* (OVOS-CONTEXT-1) — the
  context-mutation pathways transformers may use. Both the
  `ovos.session.sync` pathway (§5.3) and the in-place
  transformer pathway (§5.2) are available; the choice is the
  transformer's per the conformance rules of §9 of this spec.
- *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  and `Match` model that §3.4 operates on; §7 slot-map shape.
- *Sentence Template Grammar Specification* (OVOS-INTENT-1) — §5.3
  deferred slot value typing, for which §3.4 of this spec is the
  agreed injection home.
