# Changelog

Each entry records a change to a specification in this repository. Each
specification carries a `Version` field equal to its V0/V1/V2 compatibility
class (VERSIONING.md): `1` for a formalization compatible with the pre-spec
status quo, `2` once it is not backwards compatible. Entries are grouped under
the spec's current class. Every pull request that alters normative content adds
an entry here.

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

## OVOS-CONVERSE-1 — Active Handlers and Interactive Response

### 2

- The imperative continuous-dialog surface. Two session fields:
  `converse_handlers` (§2.1), a recency-ordered list of active handlers,
  and `response_mode` (§2.2), the single entry holding the next utterance
  exclusively. The converse plugin as a pure matcher (§3) that checks
  `response_mode` first, then polls `converse_handlers` via sequential
  unicast ping/pong (`<skill_id>.converse.ping` / `.pong`, §4). Response
  mode (§5): single-shot delivery via the reserved `response` intent_name.
  Activation lifecycle (§6), stop integration (§7), and the bus surface
  (§8). Dispatch on the reserved intent_names `converse` and `response`
  (OVOS-PIPELINE-1 §7.3) follows ordinary §7 routing.
