---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 4. Design rationale, per specification

Short notes on *why* the specifications make the choices they
do — the reasoning, not the requirement. Cross-reference into
the normative sections.

### 4.1 Intent grammar and resources (INTENT-1, -2, -3)

- **ASR-normalized input, no escaping** (INTENT-1 §2). The
  grammar targets voice input. By contract, text reaching an
  engine is already lowercased, punctuation-stripped,
  single-spaced. Bracket metacharacters therefore cannot
  occur as literal input, so no escape mechanism is needed.
  A simplification *bought* by scoping the grammar to voice.
- **Templates are training data** (INTENT-1 §4). Enumerating
  every phrasing is futile for natural speech. A template
  describes the *shape* of the training data; the engine
  generalizes. This is why expansion is defined precisely
  but matching is not.
- **An intent is not an event** (INTENT-3 §1). Necessary for
  an open skill ecosystem — see §2.2.
- **Two non-interoperable methods** (INTENT-3 §2). Keyword
  and template intents describe a command in fundamentally
  different shapes. Rather than forcing one model, the spec
  keeps both and makes engines declare which they accept.
  The cost is that a developer must choose per intent and
  know which engines an installation runs.
- **Slot typing is deferred** (INTENT-1 §5.3). Interpreting
  a slot value as a number or date is inseparable from how
  ASR output is normalized — and normalization is specified
  separately. Specifying typing first would be incoherent, so
  a slot value is an opaque sequence of words.
- **`.blacklist` vs `excluded`** (INTENT-3 §4.2, §5.4). The
  template grammar is purely generative — it cannot express
  "not this". Template intents therefore need a separate
  `.blacklist` artifact for suppression. Keyword intents
  express the same idea natively with the `excluded`
  constraint role. The asymmetry follows from the grammar,
  not from inconsistency.
- **No regular expressions** (INTENT-3 §4.4). Free-form
  structured text is a slot — use a template intent and the
  slot extractor. Regexes are also notoriously hard to
  localize, which conflicts with the per-language model.
- **Inline vocabulary references reuse `.voc`** (INTENT-1
  §3.7). A reusable template fragment and a keyword
  vocabulary are the same thing — a named, slot-free phrase
  set — so `<name>` resolves to a `.voc` rather than
  introducing a new file role. The change is one grammar
  token plus an expander step.

### 4.2 Bus message envelope (MSG-1)

- **One spec, not two.** Envelope + routing + derivations
  are tightly coupled — every routing key lives in
  `context`, every derivation manipulates routing, and all
  of them formalize *existing* OVOS code. Splitting them
  was tried; the split did not survive the derivations
  (which can only meaningfully be defined where the routing
  keys are), so they were merged into a single bus-message
  spec. The session carrier, by contrast, did split out
  cleanly into OVOS-SESSION-1.
- **`context` is extensible by design.** Only the keys
  other systems already key behaviour off (`source`,
  `destination`, `session`) are given normative meaning.
  Everything else — GUI routing, tracing, security — is
  layered by other specs without touching the envelope.
- **`source`/`destination` are informational, not
  authorization** (MSG-1 §3.3). The bus is not a security
  boundary. Layer-2 systems (HiveMind) build authentication
  and routing enforcement on top of the pair without OVOS
  itself learning about peers.
- **The boundary is user ↔ assistant, not core ↔ handler.**
  The `(source, destination)` pair marks who is currently
  talking to whom across one boundary only: the external
  participant on one side, the assistant — core and every
  skill handler — on the other. The flip happens **once**
  per conversational turn (§3.1.1), not on every internal
  hop.
- **No central correlation, no central state** (MSG-1 §5.4,
  §3.1.2 above). The bus is fully asynchronous. Components
  that need correlation or state own it themselves, keyed
  on `session.session_id`. Multi-turn conversation, intent
  context, cross-skill state, and similar concerns are
  deferred to other specifications.
- **One correlation key, in the envelope, for everybody**
  (PIPELINE-1 §9.1.1). `utterance_id` is the exception that proves the
  bullet above: it is a *value* every derivation copies, not an
  index anybody keeps. It was lifted into MSG-1 rather than left
  to each poll family because the alternative was three
  incompatible keys — a common-query `query_id`, a fallback
  round tag, a converse round tag — each correlating the same
  thing, none of them usable by a bridge or a monitor that sees
  all three. Putting it beside `session` makes it propagate under
  a rule the derivations already enforce, and makes "these two
  Messages belong to the same lifecycle" one string comparison
  anywhere on the bus.
- **Topic naming conventions** (MSG-1 v2 §2.1.1). The
  conventions other specs in the family follow are
  codified as SHOULD-rules: dot-separated hierarchy;
  stable ecosystem-identifying root; verb-tense pattern for
  the trailing segment; request/terminal pairs sharing a root
  verb (`handle` ↔ `handled`); `.response` suffix for
  response derivations. Above them sits one MUST: `:` belongs
  to the `<skill_id>:<intent_name>` dispatch shape and nothing
  else, so every other topic is a static string and no runtime
  identifier mints a topic. That is what makes a topic
  allowlist, a bridge filter, or a conformance harness writable
  against the specs instead of against a running deployment;
  the price is paid by the poll families, which moved their
  addressing into the payload (divergences §5.4).

### 4.3 Session carrier (SESSION-1)

- **Why a separate session spec.** `Message.context.session`
  is a load-bearing carrier claimed by multiple specs
  (PIPELINE-1, CONTEXT-1, TRANSFORM-1) — without a single
  owner, its wire contract drifts. SESSION-1 consolidates
  the wire shape and fixes a **registry mechanism** so
  future specs claim fields without amending SESSION-1
  itself.
- **Prescriptive, not descriptive.** Only the fields
  normatively claimed by other specs are recognized.
  Implementations carrying extra per-session state
  (the OVOS Session, for example, has `persona_id`,
  `system_unit`, `time_format`, `date_format`, `location`,
  `is_speaking`, `is_recording`, …) are non-normative under
  v1 — they ride through as opaque pass-through and remain
  available for per-domain specs to claim.
- **Omission means "let the orchestrator decide".** Single
  deferral mechanism: omitted single field, empty
  `session: {}`, absent `session`, explicit
  `session_id: "default"` — all equivalent on the wire,
  all resolve at consumption to deployment defaults filled
  by each consumer. No `null`, no sentinels.
- **Language signals.** Six BCP-47 fields with normative
  meanings but stage-dependent consolidation: `lang` (user
  preference, base), `secondary_langs` (additional
  understood languages, constrains lang-detect predictions
  and fallback selection), `output_lang` (renderer's
  preferred output language; simplifies the
  bidirectional-translation transformer to a fallback role),
  `stt_lang` / `request_lang` / `detected_lang`
  (per-utterance signals from STT, emitter, and lang-detect
  respectively). `request_lang` is an emitter-reported hint
  (per-wakeword language assignment in multi-wakeword
  setups), not an override.

### 4.4 Intent registration broadcast (INTENT-4)

- **Registrations are broadcast.** Skills emit registration
  messages on the bus; plugins that care about a particular
  registration kind subscribe to the corresponding topic.
  There is no central routing party; INTENT-4 gives this
  model normative topic names in the `ovos.intent.*`
  namespace — see §5.7 for the mapping from the bus topics
  other engines use.
- **No "no plugin claimed" error.** Following from the
  broadcast model: a registration that no plugin consumes
  is silently dropped. The producer gets no signal — the
  introspection topics (`ovos.intent.list` /
  `ovos.intent.describe`) are the supported way to verify
  what the orchestrator's passive index recorded.
- **The orchestrator passively indexes; it does not
  gate.** The introspection topics serve from a passive
  registration index the orchestrator builds purely by
  listening to broadcasts; it is not a routing authority.
  The index reflects what skills *declared*, not what
  plugins actually match against — observability-only.
- **Skill self-identification on every emission**
  (INTENT-4 §3.1). Every Message a skill emits or
  modifies in place carries `Message.context["skill_id"]`.
  Enforcement is structural on the dispatch path: the
  orchestrator stamps `context.skill_id` from the
  `<skill_id>:<intent_name>` dispatch topic prefix
  (PIPELINE-1 §7.1), and skill emissions via
  `forward`/`reply` inherit automatically.

### 4.5 Pipeline and lifecycle (PIPELINE-1)

- **PIPELINE-1 formalizes an existing plugin model**
  (§3.2). The orchestrator loads plugins by id through
  `OVOSPipelineFactory` and iterates `Session.pipeline`;
  PIPELINE-1 tightens the contract rather than introducing
  the abstraction.
- **Orchestrator and plugin contracts live in one spec**,
  since the orchestrator's job *is* iterating plugins and
  translating their matches into bus events. Splitting
  them would leave neither coherent.
- **Plugin contract is minimal.** `match(utterances, lang,
  session) → Match | None`. Side-effect-free during
  `match`; everything else (state, registrations,
  language-model calls, response generation) is
  plugin-internal black box. The smaller the contract, the
  wider the set of plugins it accommodates.
- **`lang` parameter is propagation-only.** The
  orchestrator passes `lang` through from
  `Message.data.lang`; it **MUST NOT** synthesize a value
  from `session.lang` or any per-utterance signal field
  when `data.lang` is absent. Absence is a faithful
  "unknown" signal; consumer-side fallback policy is the
  consumer's.
- **First-match-wins is the arbitration model, not the
  absence of one** (PIPELINE-1 §6.2). Evaluation order is the
  policy surface: an earlier plugin gets first refusal, which is
  the only way to guarantee that a stateful interceptor —
  converse with an open `response_mode`, an active persona, a
  media plugin holding paused media to *resume* — claims an
  utterance before the general engines. Heterogeneous plugins
  share no calibrated score space, and a state-derived certainty
  is not a quantity a text score can outbid, so cross-plugin
  ranking is both unworkable and would defeat interception;
  selective plugins are expected to be conservative and trust
  their position.
- **Tier conventions are out of scope.** The
  high / medium / low suffix is implementation strategy: each
  tier is a separate entry in `Session.pipeline`, referencing
  a match configuration rather than a distinct actor
  (PIPELINE-1 §3). The convention is compatible with
  PIPELINE-1.
- **Skills and plugins are equivalent handler owners.**
  The dispatch topic `<skill_id>:<intent_name>` is uniform:
  for a pure-matcher plugin the `skill_id` is the matched
  skill's id; for a plugin that bundles its own handler
  (e.g. a language-model persona) `skill_id == pipeline_id`.
  The two identifiers are one namespace, so both are addressed
  the same way.
- **Universal `ovos.utterance.handled` end-marker on every
  terminal path.** One reserved invariant lets observers
  count turns, route fallbacks, and know "the assistant
  is idle now" without per-stage knowledge.
- **Three-stage composition** (PIPELINE-1 §5.5) —
  preference (from `session.pipeline` or default-session
  pipeline) → availability (drop unloaded plugins) →
  policy (drop denylisted). Mirrors TRANSFORM-1 §5.3
  exactly. The same shape supports the
  client-requests/layer-2-enforces split (§3.1).

### 4.6 Intent context (CONTEXT-1)

- **Lifts intent context out of Adapt.** The Adapt-specific
  `add_context` / `remove_context` mechanism, and the
  `mycroft.skill.set_cross_context` /
  `remove_cross_context` fan-out for cross-skill use, are
  Adapt-only at the matcher level — Padatious and other
  engines ignore them. CONTEXT-1 generalizes the mechanism
  into a session-bound, decaying flat key/value store
  consumed by every intent engine uniformly via
  `requires_context` and `excludes_context` declarations.
- **Two explicit scopes encoded in the key shape.**
  `private` (orchestrator auto-prefixes with
  `<skill_id>:`) and `shared` (flat, cross-skill). OVOS
  models the same distinction informally
  (`MycroftSkill.set_context` auto-prefixes with
  `alphanumeric_skill_id`; `set_cross_skill_context` fans
  out via a bus event); CONTEXT-1 names the scopes
  explicitly and routes both through one bus surface.
- **Why private is the default.** A skill that calls
  `ovos.context.set` without specifying `scope` gets a
  private entry. This optimises for the safer case: a
  cross-skill leak from an accidentally-shared entry is
  harder to debug than a cross-skill miss from an
  accidentally-private entry. The Adapt `set_context`
  pattern is effectively skill-private, which the private
  default matches. Cross-skill coordination is a conscious
  decision that deserves an explicit `scope: "shared"`.
- **Prior art for the negative gate.** Three in-tree
  intent engines under `/plugins-pipeline/` —
  [jurebes](https://github.com/OpenJarbas/jurebes),
  [nebulento](https://github.com/OpenJarbas/nebulento),
  and [palavreado](https://github.com/OpenJarbas/palavreado)
  — independently implement `exclude_context` as a
  first-class negative gate. CONTEXT-1's `excludes_context`
  adopts the same primitive at the spec level, addressing
  patterns ("fire once", "modal suppression") that
  positive gating alone cannot express.
- **Engine-side mutation as a sanctioned non-bus
  pathway.** The Adapt pipeline plugin auto-injects matched
  entities into context *inside* `match()`, which conflicts
  with PIPELINE-1 §4.2's side-effect-free `match` rule.
  CONTEXT-1 §5.3 carves an explicit window between
  match-accept and dispatch-emit for engine-side session
  mutation, with the orchestrator (not the bus) carrying
  the write. This both legitimizes the established
  practice and resolves the PIPELINE-1 contradiction.
- **Eight-level lifecycle-position owner precedence**
  (CONTEXT-1 §5.2). When a Message carries multiple
  component-identity keys (skill_id, pipeline_id, the six
  `<type>_transformer_ids`) from a derivation chain that
  crossed component boundaries, the orchestrator picks the
  owner by lifecycle position: the latest stage to run is
  the most specific.

### 4.7 Transformer plugins (TRANSFORM-1)

- **Spec'd as an architectural pattern, not a feature
  list.** An orchestrator MAY implement chains at any
  subset of six injection points (audio, utterance,
  metadata, intent, dialog, TTS); a null-implementation is
  conformant. For each chain it does implement, the
  per-type contract binds. Each injection point's
  existence is justified by what the lifecycle holds at
  that exact moment — what's possible there that isn't
  possible elsewhere.
- **Intent transformers as the system-typing home.**
  INTENT-1 §5.3 defers slot value typing pending a text
  normalization specification. TRANSFORM-1 §3.4 is the
  spec'd injection home for typing: a deployer ships
  date / number / duration parsing once, and every skill
  receives typed values in `Match.slots` regardless of
  which engine matched. The OVOS analogue of ASK's
  `AMAZON.DATE` and Dialogflow's `@sys.date-time`, but as
  an injected enrichment rather than a built-in engine
  feature.
- **Concrete in-tree plugins as prior art.** Nine plugins
  live under `/plugins-transformer/`, covering five
  of the six injection points: utterance transformers
  (`ovos-utterance-normalizer`,
  `ovos-utterance-corrections-plugin`,
  `ovos-transcription-validator-plugin`,
  `ovos-utterance-plugin-cancel`,
  `ovos-bidirectional-translation-plugin`); dialog
  transformers (`ovos-dialog-normalizer-plugin`,
  `ovos-bidirectional-translation-plugin`,
  `ovos-dialog-transformer-openai-plugin`); audio
  transformers
  (`ovos-audio-transformer-plugin-speechbrain-langdetect`,
  `ovos-audio-transformer-plugin-ggwave`,
  `ovos-audio-transformer-redis-publish`); intent
  transformers (`ovos-keyword-template-matcher`,
  `ovos-ahocorasick-ner-plugin`). The
  `bidirectional-translation` plugin exercises the
  cross-chain coordination via `Message.context` that
  TRANSFORM-1 §7 formalizes.
- **Ascending priority.** TRANSFORM-1 §4 specifies
  ascending priority (lower = earlier, default 50). Where
  prior plugins sort transformer chains **descending**
  (`ovos_core/transformers.py:53,117,205`, `reverse=True`),
  the spec adopts the **ascending** convention used by
  fallback skills (`fallback_service.py:49`, default 101 =
  run last) and the natural "stages count up" reading.
- **Cancellation aligned with prior plugin convention.**
  Two utterance transformers
  (`ovos-utterance-plugin-cancel`,
  `ovos-transcription-validator-plugin`) signal the
  lifecycle should abort by returning empty utterance
  lists with `{canceled: true, cancel_word: <reason>}`
  context keys. TRANSFORM-1 §8 adopts this convention,
  naming the field `cancel_reason` for the structured
  concept it encodes and adding orchestrator-stamped
  `cancel_by: <transformer_id>`. The spec's
  `ovos.utterance.cancelled` terminal event sits alongside
  `ovos.intent.unmatched`, keeping cancellation and
  failure observably distinct on the bus.
- **`lang` parameter is bidirectional** (TRANSFORM-1 §3.0).
  Four of the six per-type contracts (audio, utterance,
  dialog, TTS) take `lang` as input and return it as
  output. A bidirectional-translation transformer that
  takes Spanish in and produces English out returns the
  destination language; the orchestrator writes the
  chain's final `lang` back into `Message.data.lang` for
  downstream stages. Language-detector and clearing cases
  fall out of the same channel.
- **Per-type self-identification keys, list-valued.**
  TRANSFORM-1 §1.3 claims six `Message.context` keys
  (one per transformer type) rather than a single generic
  key. Role matters: a Message may have been touched by
  multiple types in sequence, and a multi-type plugin
  (e.g., both utterance and dialog) would be ambiguous
  in a single-key model. Keys are lists because
  transformers chain — the full per-type chain is
  preserved in order.
- **Per-type denylists complete the policy surface.**
  TRANSFORM-1 §5.2 claims six
  `blacklisted_<type>_transformers` session fields,
  paralleling the six `<type>_transformers` chain-ordering
  fields of §5.1 and the
  `pipeline` / `blacklisted_pipelines` pair of PIPELINE-1
  §5. Three-stage composition (preference → availability
  → policy) in §5.3 mirrors PIPELINE-1 §5.5 exactly.
- **The per-type "explosion" is deliberate.** Twelve flat
  session fields (six chain-orderings + six denylists) plus
  six `Message.context` attribution keys. A prefix-encoded
  single namespace would require prefix parsing at every
  lookup; the per-type partition matches the existing
  registry and chain-ordering structure. Under
  SESSION-1 §3.4's SHOULD-omit rule the common case carries
  zero of these on the wire.
- **Language signals live in SESSION-1.** Language signals
  (`stt_lang`, `request_lang`, `detected_lang`, alongside
  `lang`, `secondary_langs`, `output_lang`) are
  session-scoped fields with normative meanings but a
  non-binding consolidation order — the right priority is
  stage-dependent. TRANSFORM-1 §7.1 names which
  transformer types are natural producers of which
  signals; consolidation is the consumer's decision per
  SESSION-1 §3.2.7.
- **Why each injection point is the only point.**
  Each of the six transformer chains exists at the *only*
  lifecycle stage where its input artifact is available and
  its class of mutation is possible:
  - **Audio (§3.1)** — the only stage where unprocessed
    audio exists. STT is information-lossy by design; it
    preserves *what was said* and discards almost everything
    about *how it was said*: prosody, acoustic language cues,
    speaker characteristics, ambient context, sub-vocal
    signals. Any concern that depends on the audio signal
    itself — voice activity, acoustic language detection,
    speaker identification, acoustic-event detection, noise
    reduction for downstream STT accuracy — has exactly one
    place to live.
  - **Utterance (§3.2)** — the only stage where the user's
    utterance exists as text but no semantic interpretation
    has been committed to yet. Once intent matching runs,
    the utterance is bound to a specific intent's
    slot-and-vocabulary shape; any cross-cutting text
    manipulation after that point would have to be
    intent-aware. Mutations here therefore ripple uniformly
    through every downstream stage and every intent engine —
    normalize contractions once and every engine sees the
    normalized form; translate Spanish to English once and
    every English-trained engine becomes reachable.
  - **Metadata (§3.3)** — the only stage where the joint
    audio-plus-text signal is fully available, intent
    matching has not yet committed, and the full
    `Message.context` is in flight and mutable. Audio
    transformers had no text and no session; utterance
    transformers primarily mutate the utterance list; intent
    transformers operate after match. Here a metadata
    transformer can derive cross-cutting signals from the
    joint audio+text material and make them available *once*
    to every downstream stage, by writing wherever in
    `Message.context` the consumers will look.
  - **Intent (§3.4)** — the only stage that holds *both* the
    resolved intent identity and the user's free-text capture
    values. Before match, the intent is unknown — there's
    nothing to enrich. After dispatch, the handler has
    already been called — too late to add typed equivalents
    or contextual fallbacks. The capture map is the universal
    interface every engine produces (OVOS-INTENT-3 §7), so
    enrichment here is engine-agnostic.
  - **Dialog (§3.5)** — the only stage where the assistant's
    response exists as *final text* — the skill has committed
    to what to say but TTS has not committed to how it
    sounds. Mutations here are language-aware, persona-aware,
    and content-policy-aware in ways no later stage can be:
    once the text is synthesized into audio, the
    modifications available are audio-domain only.
  - **TTS (§3.6)** — the only stage where the final response
    exists as *synthesized audio bytes* — speech text has
    been rendered to a waveform, but the waveform hasn't been
    played yet. Audio-domain modifications belong here for
    the same reason audio transformers belong pre-STT: this
    is where the acoustic dimension exists and is mutable.

- **Canonical use cases, per injection point.**
  - **Audio §3.1:** voice activity detection; audio language
    detection (writing detected language into metadata for
    downstream STT and intent stages to read); acoustic noise
    reduction; format/sample-rate normalization.
  - **Utterance §3.2:** text normalization (contractions,
    casing, common typo correction); STT transcription
    validation — dropping garbled candidates;
    cancellation/stop-word detection; source-language
    translation into the matching language; code-switching
    cleanup.
  - **Metadata §3.3:** caller/speaker identification written
    to a top-level context key; mood/urgency/formality
    classification from the joint signal; per-utterance
    language override (combining audio-language detection with
    utterance-language hint, writing the resolved language to
    `session.lang`); per-utterance pipeline switch (detecting a
    sensitive-query signal and swapping `session.pipeline`);
    system context injection (writing entries to
    `session.intent_context` for downstream pipeline plugins
    and skills to read as gates, without round-tripping
    through CONTEXT-1 §5 bus events).
  - **Intent §3.4:** system entity injection — the canonical
    use. Parse free-text capture values into typed system
    entities (dates, numbers, durations, named locations,
    ordinals) and add typed equivalents under
    conventionally-named keys for skill handlers to consume
    uniformly. This is OVOS-INTENT-1 §5.3's deferred value
    typing; this chain is the agreed home for applying that
    normalization globally so individual skills do not each
    implement it. Also: named-entity recognition over capture
    values; per-skill enrichment a deployer wants applied
    without each skill re-implementing it.
  - **Dialog §3.5:** translation to the user's preferred
    language when it differs from the rendering language;
    persona/tone rewriting; content moderation (profanity
    filtering, sensitive-topic rephrasing); length
    normalization for voice responses.
  - **TTS §3.6:** voice effects (character voices, pitch
    shifting, post-processing EQ); cross-fade or jingle
    injection for branded assistants; format conversion for
    downstream playback constraints.

- **Where LLMs fit, per injection point.**
  - **Audio §3.1:** language identification is the typical
    model-backed audio transformer; full LLMs do not run at
    this stage in any practical deployment.
  - **Utterance §3.2:** a natural injection point for
    language models — a small local model validating STT
    plausibility, a translation model producing a candidate
    string in the assistant's primary language, a paraphrase
    model adding alternative candidates so a downstream intent
    engine has more material to match against.
  - **Metadata §3.3:** a small classifier (LLM-backed or
    otherwise) inferring conversational metadata from the
    utterance and feeding the result into `Message.context` —
    useful when several pipeline plugins or skills want to
    read the same derived signal without each computing it
    themselves. Also: an LLM that reads the utterance text
    and decides per-utterance which `session.pipeline`
    configuration to apply.
  - **Intent §3.4:** the strongest match in the stack. A
    small LLM can extract structured entities (dates,
    durations, quantities) from free-text capture values and
    inject the typed forms into `Match.captures` — once,
    centrally — so every skill receives the same typed payload
    regardless of which engine matched.
  - **Dialog §3.5:** the most prominent LLM application —
    response rewriting under a persona prompt. A `tone` or
    `persona` directive on a dialog transformer routes the
    skill's plain response through an LLM with a system
    prompt, yielding the user-facing voice the assistant wants
    to present. Translation models also live here for runtime
    localization of skill-rendered text.
  - **TTS §3.6:** not applicable in any practical sense; this
    stage operates on audio bytes only.

- **Cross-cutting concerns are the architectural value.**
  Transformer chains are how a voice OS layers cross-cutting
  concerns — translation, normalization, entity tagging,
  persona rewriting, audio filtering — onto the lifecycle
  without each skill or pipeline plugin having to reinvent
  them. The architectural value is *uniformity*: a
  cross-cutting concern applied via a transformer chain
  affects every utterance / response / artifact that flows
  through that injection point, with no skill-side opt-in or
  coordination required.

- **Cancellation in-spec use cases.** An utterance transformer
  (§3.2) recognises a stop / cancel / never-mind cue in the
  user's speech and wants the lifecycle to terminate without
  reaching intent matching. A metadata or intent transformer
  detects a condition under which the utterance should not be
  acted on (a profanity filter rejecting unsafe input, a
  sensitive-context guard halting in a parental-control mode,
  a transcription-validator dropping garbage transcriptions).
  A dialog or TTS transformer determines the response itself
  should not be spoken (policy block, late content filter).

- **Introspection surface: no aggregate query.** There is
  deliberately no "give me everything" query; that would
  imply a single responder with a global view, which this
  specification does not assume exists. A consumer that wants
  all six types issues six queries.

- **Typical introspection consumers.** Developer tooling
  surfacing the loaded set; monitoring services tracking chain
  composition; integration tests asserting on chain order
  under specific session policies.

### 4.8 Bus bridge (BRIDGE-1)

- **Normative minimalism is design intent.** The bridge could have
  been an auth spec, a topology spec, a wire-protocol spec. By
  limiting normative weight to source stamping, session preservation,
  and destination-based relay (3 MUSTs in §6), the spec correctly
  identifies that session fields do the heavy lifting — the bridge
  just carries them. Everything else (policy injection, topology
  patterns) emerges from composing existing lifecycle,
  state-ownership, and session-field specifications at the bus
  boundary.
- **Destination-based routing provides client isolation.**
  Session-id-only routing lets one participant claim another's
  `session_id` and receive messages intended for the other (§3.2).
  Destination-based routing fixes this because the orchestrator uses
  `.reply()` to set `destination` to the original `source`. Two
  participants sharing the same `session_id` (especially
  `"default"`) cannot impersonate each other — their `source` values
  differ, and each receives only messages whose `destination`
  matches its own identifier. The OVOS-MSG-1 §5 derivation chain
  preserves routing metadata through every emission, so the bridge
  never sees a message without sufficient routing information.
- **Session_id matching is a MAY convenience.** The OVOS-MSG-1 §5
  derivation chain preserves `destination` across every `forward` /
  `reply` / `response` hop, so all bus messages that carry
  conversation progress carry a `destination` the bridge can route
  on. `session_id` matching exists for the narrow case of topic-level
  subscriptions a bridge explicitly opts into, not for correctness.
- **Layer-2 systems inject policy via session fields, not bridge
  protocol.** A layer-2 system (MSG-1 §3.4) mutates the session at
  the bridge boundary before the message enters the bus. No
  bridge-specific protocol is needed — the session fields do the
  work. This is the same pattern as SESSION-2 §2.4's
  SHOULD-project pathway, but with the bridge as the enforcement
  point rather than the component itself.
- **The hardened minimum topic set is a deployer choice.** A bridge
  MAY subscribe to everything (default, simpler, compatible with
  future lifecycle additions) or restrict to the utterance-lifecycle
  topic set (hardened, reduced attack surface). The trade-off is by
  design, not prescribed.
- **`site_id` enables bridge-side physical grouping without
  requiring skills to understand topology.** The bridge is the
  natural point to assign `site_id` because it is the only
  component that has visibility into both the physical deployment
  and the internal bus. A concrete example: a bridge that receives
  Wi-Fi or Bluetooth scan data from participants can resolve that
  signal to a physical location and stamp the appropriate `site_id`
  before injecting the message; a bridge that integrates with a
  home-automation system can use the canonical area name from that
  system (e.g. `"living_room"`, `"kitchen"`) directly as the
  `site_id`, giving skills a stable identifier that is already
  meaningful in the user's home model. In either case, participants
  need not know their own location, and skills need not understand
  network topology.
  The `site_id` then travels with the session, giving any downstream
  pipeline plugin or skill a stable, location-derived grouping key
  for context (e.g. applying room-specific TTS voices, routing
  media to the nearest speaker, or gating location-aware intents).
  This is why the spec mandates `site_id` for group routing rather
  than enumerating participants: the bridge encapsulates the mapping
  from physical signal to logical group, and everything downstream
  consumes an opaque string.

### 4.9 Audio output service (AUDIO-1)

**Sentence segmentation as a latency-reduction technique (AUDIO-1 §3.2).**
When a TTS engine synthesises a long utterance as a single unit, the
user must wait for the entire synthesis to complete before hearing
anything. An implementation can reduce perceived latency by splitting
the utterance at sentence boundaries, synthesising each sentence
independently, and enqueuing each segment as soon as it is ready —
so the first sentence begins playing while later sentences are still
being synthesised.

This is an internal implementation strategy: no other bus participant
observes whether the TTS engine segments or not. The visible contract
is unchanged — `ovos.audio.output.started` fires when the first
audio begins, `ovos.audio.output.ended` fires when the last audio
completes. The `listen` flag is honoured after all audio for the
originating utterance has played, regardless of how many internal
segments were used.

### 4.10 Stop pipeline plugin (STOP-1)

The most common reader question on first encountering STOP-1 is
*why a pipeline plugin and not a skill*. Stop sounds like an
ordinary intent: a user utterance ("stop", "cancel") matched and
handled. A skill that registers a `stop` intent and implements a
`stop` handler looks like the obvious shape. STOP-1 deliberately
lifts stop into the pipeline layer instead, and the reasons are
load-bearing — a skill cannot implement the cascade defined in
STOP-1 §4 even in principle.

**Pre-emption requires evaluation-layer ordering control, not
handler-layer dispatch.** Stop's defining property is that it
pre-empts every other matching stage — active converse polls,
response-mode delivery, ordinary intent matching. Pipeline
plugins are evaluated in declared order with first-match-wins;
STOP-1 §7 positions the stop plugin first so it gets the first
opportunity to claim every utterance. A skill's intent handler
runs only *after* intent matching has already selected it, by
which point converse and intent matchers have already had their
say. The escape-hatch property lives at the pipeline-iteration
layer, not the handler layer; a skill is at the wrong layer to
own it.

**The cascade target is decided before dispatch.** STOP-1 §4.1
consults `session.active_handlers`, performs the ping-pong
filter, picks the most recently activated responder by
`activated_at`, and emits a Match whose `skill_id`
is the chosen target. The orchestrator then dispatches
`<target>:stop` directly using its ordinary routing rules. A
skill matching stop utterances would itself become the dispatch
target, and would then have to re-emit synthetic dispatches at
other skills — bypassing the orchestrator's routing model and
losing the standard handler-lifecycle trio for the actual stop.
Match-phase target selection is what reduces the cascade to a
single clean PIPELINE-1 dispatch instead of two-step orchestration.

**`Match.updated_session` carries the post-stop session state.**
STOP-1 §6.2 requires the stopped handler to be removed from
`active_handlers` via `Match.updated_session` so the cleared
state propagates through the rest of the utterance lifecycle.
Skills have no `Match` to mutate; their handlers receive the
dispatch session and may mutate it from within the handler
boundary, but cannot communicate session changes that apply
*to the dispatch itself*.

**The reserved-name authority lives at the spec / pipeline
layer.** STOP-1 §2 reserves `stop` across every OVOS-INTENT-4
registration in the deployment, enforced by the orchestrator's
malformed-payload treatment of competing registrations. The
authority to define what `stop` means globally — and to police
skill-level attempts to claim the name — cannot live inside any
single skill that itself uses the name.

**Confidence-tier interleaving is a pipeline-ordering concern.**
STOP-1 §7 describes `stop_high` / `stop_medium` / `stop_low`
interleaved with other pipeline plugins of comparable confidence.
A skill has no analogous handle on inter-stage ordering; intent
confidence is consumed *by* intent matchers, not by the outer
pipeline that decides which matcher runs first.

The two layers cooperate by design. A skill MAY — and per STOP-1
§9 SHOULD — provide its own stop *handler*: every skill that
participates in the cascade implements a stop intent handler
subscribed to `<own_skill_id>:stop`. The pipeline plugin matches
and selects; the skill stops. Stop is one of the few cases in
the spec set where the pipeline / skill split is not
substitutable.

### 4.11 Common query pipeline plugin (COMMON-QUERY-1)

Common query answers factual questions by holding a timed contest
among skills — broadcast the question, collect competing answers,
rank them, speak the best. Four of its design choices are
unusual enough to be worth recording, because each one trades
against an instinct a reader brings to the spec.

**`match` blocks, and that is deliberate.** PIPELINE-1 §4.4 tells
plugins to return from `match` quickly and defer expensive work to
the handler, because match-phase latency is response latency. Common
query openly violates that discipline, and it has to: the answer
*is* the claim decision. The plugin cannot return a `Match` and
collect afterwards, because whether it claims at all depends on
whether any skill produced an answer above threshold. Routing and
processing are the same act here, so both happen in `match`. This is
the one place the spec set says "yes, this matcher blocks for
seconds" — and it pays for that admission with the early-start
optimisation and explicit pipeline positioning, rather than
pretending the cost away.

**Returning `None` on no-answer is what keeps fallback alive.** The
earlier, discarded design had the plugin claim the utterance, then
discover during the handler that no skill could answer, and speak a
dead-end "I don't know." That permanently starves fallback: once a
plugin claims, first-match-wins means no later stage runs. Moving the
whole contest into `match` lets the plugin make an honest claim — it
returns a `Match` only when it actually has an answer, and `None`
otherwise — so a failed contest flows naturally to fallback. The
correctness of the whole pipeline tail depends on the contest
finishing before the claim is made.

**Ping/pong is a cheap filter gating an expensive operation, not
ceremony.** It would be simpler to broadcast the question once and
let skills answer directly. The two-phase poll earns its place
because the full-answer request invites real I/O — a knowledge skill
will hit Wikipedia, Wolfram, or a database. Without the cheap local
pong filter, every such skill performs that I/O for every question
that passes the gate, including ones far outside its domain. The
~500ms poll window buys the right to *not* hammer every backend on
every utterance. (Mycroft's original CommonQuerySkill was also
two-phase, but for a different reason — message-bus timeout
management; see comparisons §2.6.)

**Early start hides latency without shrinking the contest.** Because
`match` blocks, the plugin MAY begin the contest the instant the
utterance arrives (`ovos.utterance.handle`), running it in parallel
with the upstream stop/converse/intent stages that get first refusal
anyway. The subtle requirement is that the early-start cache holds
only *raw* responses — never a selected answer — and all filtering
and selection run at `match` time against the *live* session. That
keeps the optimisation transparent: an upstream stage that
blacklists a skill or changes session state still takes full effect,
because the denylist and confidence filters never saw the stale
snapshot.

The question gate (COMMON-QUERY-1 §4) is the other half of the
latency story: a cheap up-front classifier that rejects weather
requests, music commands, timers, and plain statements before any
broadcast. It is a SHOULD, not a MUST — the confidence filter
guarantees correctness without it — but on mixed traffic it is the
single largest latency win available, since it skips the entire
contest for utterances no knowledge skill would answer anyway.
