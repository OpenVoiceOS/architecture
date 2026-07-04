# Changelog

Each entry records a change to a specification in this repository. Each
specification carries a `Version` field equal to its V0/V1/V2 compatibility
class (VERSIONING.md): `1` for a formalization compatible with the pre-spec
status quo, `2` once it is not backwards compatible. Entries are grouped under
the spec's current class. Every pull request that alters normative content adds
an entry here.

## OVOS-AUDIO-1 — Audio Output Service

### 2

- The audio output service: the rendering pipeline (dialog-transformer
  chain, TTS synthesis, TTS-transformer chain, playback queue), the
  sequential playback queue shared by speech (`ovos.utterance.speak`) and
  sound effects (`ovos.audio.queue` / `ovos.audio.play_sound`), the
  remote-client rendering mode (`ovos.utterance.speak.b64` →
  `ovos.audio.speech`), output lifecycle signals
  (`ovos.audio.output.started` / `.ended`), the speaking-status query
  (`ovos.audio.is_speaking`), stop integration (`ovos.audio.stop`,
  `ovos.stop`), and the `listen`-triggered `ovos.mic.listen` follow-up.
## OVOS-PERSONA-1 — Persona Pipeline Plugin

### 2

- Initial draft. Defines persona as a scoped match+handler layer with
  its own pipeline position, `persona_id` session field, summon and
  dismiss bus messages, match contract (MAY return `None` for pass-
  through), handler contract, no-persona mode, multiple-persona
  coexistence rules, `fallback_pipeline_id` for escalation, dialog-
  transformer compatibility, capability-discovery via
  `ovos.persona.capabilities`, and conformance roles (Persona Plugin,
  Orchestrator, Skill).
## OVOS-CONTEXT-1 — Intent Context

### 2

- Initial draft. Defines `session.intent_context` as a flat map of
  key → entry carried inside the SESSION-1 session carrier. Covers
  context entries (key, value, TTL, owner `skill_id`), private vs.
  shared scopes (`:` discriminator in the key), decay (TTL counts down
  per utterance; entries without TTL are session-persistent), three
  mutation pathways (skill bus events; engine auto-population on match;
  orchestrator sweep on session close), the `requires_context` and
  `excludes_context` intent declaration fields, interaction with the
  match result (context values MAY be injected into `Match.slots`),
  and conformance roles (Orchestrator, Pipeline Plugin, Skill).
  Non-goals: trust enforcement and replay prevention are explicitly
  out of scope.
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
- §5.5 — slot consistency applies to `.dialog` only; a `.intent` file MAY
  declare different slot sets across its templates, and the engine extracts
  the slots of the best-matching template. §6.2 — the engine verifies a
  consistent slot set for `.dialog` and accepts differing slot sets for
  `.intent`.

### 1

- Initial draft.

## OVOS-INTENT-2 — Locale Resource Formats

### 2

**Breaking change.** Version 2 reinterprets brace handling in `.prompt` and
`.dialog` files. A `.prompt` authored against version 1 changes meaning under
version 2: its `{{ … }}` sequences become substitution points, and its
`<!-- … -->` sequences are no longer stripped.

- §4.4 (`.prompt`) — the substitution variable is now the **double-brace**
  `{{name}}` form **only**. A single `{name}`, a lone `{` or `}`, and literal
  JSON/markup pass through unchanged, which is why prompts (full of literal
  single braces) require double braces. The previous author-comment exception
  is **removed**: a `.prompt` has no comment handling and `<!-- … -->` is
  literal text. `{{name}}` substitution is now the *only* special handling a
  `.prompt` receives, dropping the prior fenced-code-block rule (a single-brace
  `{name}` is already literal everywhere).
- §4.2 (`.dialog`) — now recognizes **both** named-slot forms, `{name}` and the
  equivalent `{{name}}`, per OVOS-INTENT-1 §3.4, treating them identically. The
  prior "single-brace only; no `{{ }}` form" restriction is dropped.
- §4.1 (`.intent`) — clarified to match OVOS-INTENT-1 §5.5: lines MAY declare
  different sets of named slots, the intent's slot set is their union, and the
  engine extracts only the slots of the matched template. The prior "every line
  MUST declare the same set of slots; use a separate file" rule is dropped for
  `.intent` (it still holds for `.dialog`, §4.2).

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

- The intent definition model: keyword and template intents, identity,
  captured slots, suppression, and the handler contract.
- §5.1 — a template intent's `.intent` templates MAY declare different slot
  sets; the engine extracts the slots of the best-matching template
  (OVOS-INTENT-1 §5.5).
- §5.3 — `required_slots`: an optional list of slot names the engine MUST
  extract for a match to be valid; a required slot must be declared by at
  least one template, otherwise the definition is malformed. §5.4 (Example)
  and §5.5 (the `.blacklist` suppression) renumber accordingly.
- §7.1 — a handler MAY rely on `required_slots` being populated; all other
  slots remain optional and must be defended against.

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

### 2

- Bus contract for declaring intents and entities, the wire companion to
  OVOS-INTENT-3. Registration topics (`ovos.intent.register.keyword` /
  `.template`, `ovos.entity.register`), deregistration / enable / disable,
  and orchestrator-owned manifest introspection (`ovos.intent.list` /
  `.describe`). Keyword registration carries inline `required` / `optional`
  / `one_of` / `excluded` vocabulary descriptors. A single intent MAY be
  registered under both keyword and template methods. Registrations are
  fire-and-forget broadcasts: no `.response` acknowledgement; manifest
  presence is the only success signal. Consuming plugins log
  malformed-payload rejections at WARN with full identifiers and the
  rejecting topic. File paths never cross the bus — locale files are
  expanded inline before emission.
- §6 — `required_slots`: an optional array on the
  `ovos.intent.register.template` payload listing slot names the engine
  MUST extract for a match to be valid (OVOS-INTENT-3 §5.3). §6.2 —
  templates MAY declare different slot sets (OVOS-INTENT-1 §5.5). §6.3 —
  a `required_slots` entry naming a slot no template declares is malformed.
- §11 — session-scoped registration. The registration key is the quintuple
  `(session_id, skill_id, intent_name, lang, method)`, read from
  `context.session.session_id`; `session_id == "default"` is the global
  scope every session inherits (§11.2). Deregistration MAY narrow to one
  `session_id` (§8.4); session teardown removes its session-scoped
  registrations (§11.3). A plugin that does not implement session scoping
  treats every registration as global (§11.4). §11.5 — the orchestrator
  routes a session-scoped intent only to utterances whose session matches
  the registration's `session_id`.
- §8.1 — intent replacement is keyed by the quintuple, entity replacement
  by `(session_id, skill_id, entity_name, lang)`. §12 — the orchestrator
  keys the manifest by the quintuple and serves session-aware
  `ovos.intent.list` queries.

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

## OVOS-OCP-1 — OVOS Common Playback: the Virtual Media Player

### 1

- Initial draft. Formalizes the Virtual Media Player: one logical,
  session-scoped media player that every media command targets, behind
  which any number of playback backends, remote devices, or external OS
  players may serve a track. Defines the player and media state model;
  the bus surface that requests playback and transport control and
  reports state; the distinction between playback requests (start
  something) and control requests (act on what is already playing); and
  the MPRIS bridge by which the virtual player is exported to the host OS
  and by which externally-initiated, standards-compliant players become
  controllable by voice. Media discovery and ranking, URI-to-bytes
  playback, stream-extraction formats, now-playing rendering, and the
  NLU that classifies an utterance as media are out of scope — provider,
  backend, GUI, and pipeline concerns respectively.
## OVOS-GUI-1 — GUI Display Subsystem

### 1

- Initial draft. Formalizes the GUI display subsystem that decouples
  voice applications from rendering technology: applications declare
  *what* to display by naming a template from a closed, curated
  vocabulary of `SYSTEM_*` templates and supplying that template's
  session data; interchangeable render backends (adapters) decide
  *how*. Defines the closed template vocabulary (§3) — state/feedback,
  content primitives, media, domain cards, and interactive companions
  — each with its normative session-data keys; the wire protocol (§4)
  — `gui.value.set` / `gui.page.show`, the reserved `__from` / `__idle`
  data keys, and the per-session namespace lifecycle; addressing (§5)
  — routing by `session_id` alone, with shared/multi-room screens
  expressed by clients sharing a `session_id` and no separate
  site/location dimension; the adapter contract (§6) — entry-point
  discovery, unconditional multi-modal fan-out to every installed
  adapter, exception-safe non-blocking handlers/hooks keyed by
  `session_id`, and capability-degradation guidance; the
  interaction path (§7) for media transport and `confirm` / `select`
  responses carrying the originating `session_id`; and conformance
  (§8) for producers, the GUI service, and adapters. Raw-passthrough
  templates (`SYSTEM_html` / `SYSTEM_url`) are documented as
  discouraged escape hatches. Builds on OVOS-MSG-1 (envelope),
  OVOS-SESSION-1 (the `session_id` routing key and reserved
  `"default"`), and OVOS-SESSION-2 (lifecycle / client authority).

## OVOS-STOP-1 — Stop Pipeline Plugin

### 2

- The stop pipeline plugin: matches utterances expressing the intent to
  interrupt the assistant and either cascades a stop across the
  recency-ordered active handlers or broadcasts a global stop. The
  ping/pong discovery exchange (`ovos.stop.ping` / `.pong`, §4) that
  selects the most recently activated handler able to stop, per-handler
  dispatch on the reserved intent_name `stop` (`<skill_id>:stop`,
  OVOS-PIPELINE-1 §7.3), the global-stop path (`global_stop` →
  `ovos.stop` broadcast), and the session-scoping obligations over
  `session.active_handlers`.

## OVOS-COMMON-QUERY-1 — Common Query Pipeline Plugin

### 2

- Initial draft. Specifies the common query pipeline plugin: a
  scatter-gather contest that answers factual questions by
  broadcasting the utterance, collecting competing answers from
  skills, ranking them, and speaking the best. Reserves the
  `common_query` intent_name (PIPELINE-1 §7.3). The full contest runs
  in the plugin's blocking `match` (a deliberate, documented exception
  to PIPELINE-1 §4.4 latency discipline, since the answer is the claim
  decision): a fast `ovos.common_query.ping`/`pong` poll filters
  skills down to plausible answerers using only cheap local checks,
  then `<skill_id>:common_query` requests full answers (where network
  and DB I/O are expected) collected on `<skill_id>.common_query.response`.
  Filtering and selection (minimum confidence, denylist, fast-win,
  optional reranker) run against the live session; if no answer
  survives, `match` returns `None` so the pipeline reaches fallback.
  A surviving answer is carried in `Match.slots.answer` and spoken by
  the plugin's trivial handler — skills never speak. Defines an
  optional question gate (SHOULD, for latency) and an early-start
  optimisation subscribing to `ovos.utterance.handle` to overlap the
  contest with upstream pipeline stages, caching only raw responses
  keyed by `(session_id, utterance)`. All poll/answer messages carry
  the `utterance` as correlation key and derive via MSG-1 `reply`,
  with the session in `context.session`. Tunable defaults and
  confidence-range guidance are collected in appendices.
## OVOS-FALLBACK-1 — Fallback Pipeline Plugin

### 2

- The fallback pipeline plugin: the final stage(s) that handle utterances
  no earlier stage claimed by querying registered fallback skills in
  priority order and dispatching to the first willing one. Skill
  registration (`ovos.fallback.register` / `.deregister`) with a priority
  hint, session-scoped per OVOS-INTENT-4 §11. The `session.fallback_handlers`
  preference list (§4), pool construction (§5), the sequential unicast
  ping/pong match contract (`<skill_id>.fallback.ping` / `.pong`, §6),
  dispatch on the reserved intent_name `fallback` (§7, OVOS-PIPELINE-1 §7.3),
  and pipeline positioning with multi-stage priority ranges (§8).
- Consistency audit: §6.1 the per-skill wait MUST be bounded by a ceiling
  (unbounded waits stall the utterance), with the ceiling itself a
  RECOMMENDED default of 0.5 s matching the analogous OVOS-CONVERSE-1 /
  OVOS-STOP-1 polls; an absent or malformed pong (missing or non-boolean
  `can_handle`, mismatched `skill_id`) MUST be treated as
  `can_handle: false`, uniform with the companion polls' silence rules;
  §6.1 broadcast-poll optimisation defined as observably equivalent to
  the sequential cycle (selection stays keyed on pool order, never
  response-arrival order); §3.3 priority tiers remain a recommended
  convention mapped by band membership, not numeric rescale; §3.3
  catch-all ordering stated as SHOULD; §6.3 `utterance` defined as the
  first element of the candidate list (OVOS-PIPELINE-1 §4.1); cross-spec
  note that the poll boolean's field name (`can_handle`) is
  protocol-local; citations corrected (`blacklisted_skills` →
  OVOS-PIPELINE-1 §5.3, session registry → OVOS-SESSION-1 §2.2,
  `ovos.intent.unmatched` → PIPELINE-1 §9.3) and OVOS-SESSION-1 cited by
  its canonical title.
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
## OVOS-PIPELINE-1 — Utterance Lifecycle and Pipeline

### 2

- §6.2 — the orchestrator treats a `Match` as declined when any slot
  listed in the intent's `required_slots` (OVOS-INTENT-3 §5.3) is absent,
  behind the engine's own enforcement during `match`.
- §7.3 — reserve the intent_names `fallback` (OVOS-FALLBACK-1 §6.3) and
  `common_query` (OVOS-COMMON-QUERY-1 §3).
- §9.6 — the OPTIONAL `listen` field on `ovos.utterance.speak`: when
  `true`, the output stage re-opens the user input channel after the
  response is delivered.

### 1

- The utterance lifecycle, the pipeline-plugin `match` contract, the
  session fields owned by this specification (§5), dispatch (§7), the
  handler-lifecycle trio (§8), and the utterance-layer topics — the
  entry point `ovos.utterance.handle` (§9.1) and the response exit point
  `ovos.utterance.speak` (§9.6).
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
