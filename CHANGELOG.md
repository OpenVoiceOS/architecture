# Changelog

Each entry records a versioned change to a specification in this repository.
Each specification is versioned independently, starting at version 1. Every
pull request that alters normative content bumps the affected spec's `Version`
field and adds an entry here.

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

### 2

- §1, §4.3 — the `.voc` role description is broadened to "a named set of
  localized phrasings": a `.voc` is consumed as a keyword vocabulary and/or
  referenced inline via `<name>` (OVOS-INTENT-1 §3.7), and may itself contain
  such references.

### 1

- Initial draft.

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

## OVOS-INTENT-4 — Intent and Entity Registration and Dispatch

### 1

- Initial draft. Defines the full intent lifecycle on the bus:
  registration (`ovos.intent.register.keyword`,
  `ovos.intent.register.template`, `ovos.entity.register`),
  deregistration (`ovos.intent.deregister`, `ovos.entity.deregister`,
  `ovos.skill.deregister`), enable/disable (`ovos.intent.enable`,
  `ovos.intent.disable`), introspection (`ovos.intent.list`,
  `ovos.intent.describe`), the match-result notification
  (`ovos.intent.matched`), dispatch on the qualified-name topic
  `<skill_id>:<intent_name>` (INTENT-3 §3), and the handler-lifecycle
  trio `ovos.intent.handler.start` / `.complete` / `.error` that the
  skill emits to report handler outcome (mirroring the start/complete/
  error pattern `ovos-workshop` already implements, renamed into the
  `ovos.intent.*` namespace for uniformity). No separate dispatch
  `.response` is defined; handler outcome is observed via the trio.
  Adopts a host-mediated model: the host is the sole bus consumer of
  skill-originated registration topics and delegates matching to its
  engines through a host-internal interface; skills subscribe to their
  own qualified-name dispatch topics. Aligned to OVOS-INTENT-3's
  keyword/template duality; replaces the legacy Mycroft-era
  registration topics (`register_vocab`, `register_intent`,
  `padatious:register_intent`, `padatious:register_entity`,
  `detach_intent`, `detach_skill`) and renames the legacy
  `mycroft.skill.handler.{start,complete,error}` trio; **keeps** the
  existing `<skill_id>:<intent_name>` dispatch topic as the prescribed
  v1 form so existing skill handler subscriptions continue to work
  unchanged. See Appendix A for the full mapping.
