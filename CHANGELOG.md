# Changelog

Each entry records a change to a specification in this repository. Each
specification carries a `Version` field equal to its V0/V1/V2 compatibility
class (VERSIONING.md): `1` for a formalization compatible with the pre-spec
status quo, `2` once it is not backwards compatible. Entries are grouped under
the spec's current class. Every pull request that alters normative content adds
an entry here.

## OVOS-TRANSFORM-1 — Transformer Plugins

### 1

- Initial draft. Defines six transformer chains at six injection
  points in the OVOS-PIPELINE-1 §6 utterance lifecycle, in lifecycle
  order: audio (raw audio before STT, §3.1), utterance (post-STT text
  normalization before intent matching, §3.2), metadata (session
  enrichment after the utterance text, before the match round, §3.3),
  intent (match-result adjustment after the match round, before
  dispatch, §3.4), dialog (response-text transformation after a skill
  emits `speak()`, before TTS, §3.5), and tts (synthesized-audio
  transformation after TTS, before playback, §3.6). An orchestrator
  MAY implement any subset of the six points; an unimplemented chain
  is a no-op. Chains are ordered; the output of one transformer is the
  input to the next. Per-session ordering and denylists via the
  `<type>_transformers` / `blacklisted_<type>_transformers` session
  fields (§5). Defines session mutation discipline: transformers MAY
  mutate session fields they own (SESSION-1 §2.1) but MUST NOT mutate
  fields owned by other specs; and utterance cancellation (§8) as the
  only sanctioned early short-circuit of the lifecycle, preserving the
  `ovos.utterance.handled` invariant. Conformance roles: Audio,
  Utterance, Metadata, Intent, Dialog, and TTS Transformer, plus
  Orchestrator.

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
- §3.4, §3 — a named slot MAY be written either `{name}` or `{{name}}`; the two
  forms are exactly equivalent, with a conformant tool folding `{{name}}` to
  `{name}`. The slot-name charset and whitespace rules apply to both. The
  double-brace spelling is a slot, not a brace-escaping form; the grammar still
  provides no escape.

### 1

- Initial draft.

## OVOS-INTENT-2 — Locale Resource Formats

### 1

- The locale folder layout and the plain-text resource file formats
  (`.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist`).
- §1, §4.3 — the `.voc` role is a named set of localized phrasings,
  consumed as a keyword vocabulary and/or referenced inline via `<name>`
  (OVOS-INTENT-1 §3.7), and may itself contain such references.
- §4.4 — the `.prompt` resource role: a whole-file verbatim string
  delivered to a language model. Not a template grammar file: no
  expansion, no line filtering, every character literal. Author-only
  HTML-style comments (`<!-- … -->`) are stripped before delivery; a
  malformed comment (unmatched `<!--`) MUST be reported. Optional
  `{name}` substitution fills only names the caller provides; unfilled
  slots remain literal text, and slots inside fenced code blocks are
  never substituted. Follows the §2.1 locale-override precedence.

## OVOS-INTENT-3 — Intent Definition

### 1

- Initial draft.

## OVOS-MSG-1 — Bus Message

### 1

- Initial draft. Formalizes existing OVOS bus behaviour as a single
  specification covering: the on-the-wire JSON envelope (`type` /
  `data` / `context`); the routing keys `source` and `destination`
  that mark the OVOS / handler-code boundary (the attachment point
  layer-2 systems like HiveMind build on top of); the `session`
  carrier with two normative internal fields — `session_id` (where
  `"default"` is reserved for "the Message originates from the device
  itself", already used by `ovos-audio` to decide whether to play TTS
  locally; an absent `session` is treated as equivalent to
  `session_id: "default"`, and `forward`/`reply`/`response` MAY
  materialize the default during derivation) and `lang` (the user's
  preferred language, distinct from per-payload `data.lang`
  describing the message's data language, usually but not necessarily
  matching); the `forward` / `reply` / `response` Message
  derivations; the topic+session correlation model for `.response`
  matching; and UTF-8 JSON serialization rules. No new fields are
  introduced; every key and derivation defined already exists in
  current OVOS code paths (`ovos-bus-client.Message` for the
  envelope, `Message.reply` for source/destination swap,
  `context["session"]` for the session carrier, `ovos-audio` for the
  `session_id == "default"` policy hook). Encryption, transport,
  authentication, authorization, retry, delivery and ordering
  guarantees, session lifecycle, and the internal shape of `session`
  beyond `session_id` and `lang` are explicitly out of scope.

## OVOS-SESSION-2 — Session Lifecycle and State Ownership

### 1

- The state-ownership model (stateless bus, stateless orchestrator for
  named sessions, orchestrator-owned default session), the mutation
  boundaries, `ovos.session.sync` (§2.7), client-side merge rules,
  resumption semantics, and conformance.
- §2.4 — a handler that emits no Message cannot propagate in-place
  session mutations: the orchestrator-emitted handler-lifecycle trio
  does not reflect handler-side changes, so a handler whose mutations
  must appear in terminal events emits at least one Message
  (`ovos.utterance.speak` or `ovos.session.sync`).
## OVOS-SESSION-1 — Session Carrier Wire Shape

### 1

- The `context.session` carrier wire shape: the `session_id` and `lang`
  core fields, the language field family (§3.2), the §2.1 field-registry
  mechanism by which other specifications claim OPTIONAL session fields,
  and the propagation and wire-weight rules.
- §2.1 — registered fields and their owning specifications:
  `converse_handlers` (OVOS-CONVERSE-1 §2.1), `fallback_handlers`
  (OVOS-FALLBACK-1 §4), and `persona_id` (OVOS-PERSONA-1 §3).
- §3.3 — `site_id` is defined by OVOS-BRIDGE-1 §3.3; this section states
  the consumer constraints that apply within the orchestrator pipeline.
- See also — each field's owning specification, including
  `session.active_handlers` (OVOS-PIPELINE-1 §7.1) and
  `session.converse_handlers` (OVOS-CONVERSE-1 §2.1).

## OVOS-INTENT-4 — Intent and Entity Registration Bus Contract

### 1

- Initial draft. Bus contract for declaring intents and entities, the
  wire companion to OVOS-INTENT-3. Defines registration topics
  (`ovos.intent.register.keyword` / `.template`, `ovos.entity.register`),
  deregistration / enable / disable, and orchestrator-owned manifest
  introspection (`ovos.intent.list` / `.describe`). Atomic keyword
  registration with inline `required` / `optional` / `one_of` /
  `excluded` vocabulary descriptors. Structured identity via the
  `(skill_id, intent_name, lang)` triple plus a `method` axis for
  manifest indexing — a single intent MAY be registered under both
  keyword and template methods as two training-data representations.
  Fire-and-forget broadcast model: no `.response` acknowledgements;
  manifest presence is the only success signal. Consuming plugins MUST
  log malformed-payload rejections at WARN with full identifiers and
  the rejecting topic. File paths never cross the bus — INTENT-2 locale
  files are a producer-side authoring convenience expanded inline by
  the skill loader before emission.

## OVOS-AUDIO-IN-1 — Audio Input Service

### 2

- §6 (new) — listening lifecycle signals. The audio input service
  emits `ovos.listener.record.started` / `ovos.listener.record.ended` around
  voice-command capture, accepts `ovos.listener.sleep` to enter sleep mode
  and suspend capture, and emits `ovos.listener.awoken` on the sleep→awake
  transition. These replace the legacy `recognizer_loop:record_begin`
  / `recognizer_loop:record_end` / `recognizer_loop:sleep` /
  `mycroft.awoken` topics. All carry no payload; the session is
  identified by `context.session.session_id`.
- §6.5 — bus surface table for the listener role, including the
  consumer-side `ovos.mic.listen` row (defined in OVOS-AUDIO-1 §4.4).
- See-also — cross-references OVOS-AUDIO-1 §4.4 as the defining spec
  for `ovos.mic.listen`.

## OVOS-BRIDGE-1 — Bus Bridge and Opaque Relay

### 2

- The bus bridge: a participant that terminates an external channel and
  relays Messages between the internal bus and remote participants. §3 —
  the normative core: inbound identity stamping (`source`), outbound
  routing by `destination` / `session_id` / `site_id`, `site_id`
  assignment, and the relaying vs managing session-preservation modes.
  §4 — emergent patterns over MSG-1 + SESSION-1/2 + PIPELINE-1 +
  TRANSFORM-1 + CONTEXT-1 + INTENT-4 at a bus boundary: policy injection,
  multi-deployment topologies, out-of-utterance `ovos.session.sync`, and
  satellite skill registration. §5 ordering guidance; §6 conformance.
- §3.3 — `site_id` assignment is owned here; OVOS-SESSION-1 §3.3 carries
  the registry pointer and the orchestrator-pipeline consumer constraints.
