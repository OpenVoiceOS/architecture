---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

> **⚠️ AI-generated draft — not yet fully reviewed.** This content
> was produced by a large language model (Claude Code) and
> has not yet been fully reviewed for accuracy, completeness, or
> consistency with the specifications. The normative specifications
> themselves are human-reviewed; this appendix is supplementary
> context. Readers should verify claims before relying on them.

## 5. Where the specs differ from current OVOS code

These specifications are *prescriptive*. Some of what they
prescribe matches what runs in OVOS today verbatim; some is a
deliberate cleanup the implementations are expected to grow
into. This section catalogues every known divergence so
implementers know what to migrate and reviewers know what to
expect.

### 5.1 Already aligned

Formalizations of behaviour that exists in current OVOS code
and needs no implementation change:

- The Message envelope (`type` / `data` / `context`) — matches
  `ovos-bus-client.Message`.
- `source`, `destination` semantics including the
  `Message.reply` swap — matches `ovos-bus-client/message.py`.
- `context.session` as a serialized Session object — matches
  `ovos-bus-client/client/client.py`'s
  `message.context["session"] = sess.serialize()`.
- `session.session_id == "default"` for device-local origin —
  matches `ovos-audio/utils.py`'s `require_default_session`
  decorator.
- `session.lang` as the user's preferred language — matches
  the Session class's `lang` attribute.
- `forward` / `reply` / `response` derivation semantics —
  matches `ovos-bus-client.Message.{forward,reply,response}`.
- The `.response` suffix convention — pervasive across OVOS
  topics today.
- `ovos.utterance.cancelled` and `ovos.utterance.handled`
  (PIPELINE-1) — match current topic names verbatim.
- Per-utterance first-match-wins iteration (PIPELINE-1) —
  matches `ovos-core/intent_services/service.py`'s
  `handle_utterance` / `get_pipeline`.
- Per-session pipeline configuration (PIPELINE-1) — matches
  `Session.pipeline`.
- The `<skill_id>:<intent_name>` dispatch topic shape
  (PIPELINE-1) — matches current OVOS practice; skills
  already subscribe to these topics.

### 5.2 Prescriptive renames

| Spec | Current | Prescribed | Notes |
|------|---------|------------|-------|
| INTENT-3 v1.1 | "host" | "orchestrator" | Editorial; conformance unchanged. |
| PIPELINE-1 | `mycroft.skill.handler.start` / `.complete` / `.error` | `ovos.intent.handler.start` / `.complete` / `.error` | Renamed into the `ovos.intent.*` namespace for uniformity. Breaks every existing handler-lifecycle observer; the migration cost is real. |
| PIPELINE-1 | `recognizer_loop:utterance` | `ovos.utterance.handle` | See §5.4 entry. Migration touches `ovos-dinkum-listener`, `ovos-simple-listener`, `ovos-audio`, and `ovos-core/intent_services/service.py`. |
| PIPELINE-1 | `complete_intent_failure` | `ovos.intent.unmatched` | Follows `ovos.intent.*` namespace; pairs with `ovos.intent.matched`. |

### 5.2.1 Topics to remove from ovos-core

The following topics exist in current ovos-core but are **not
defined by any spec** and should be removed or replaced:

- **`ovos.session.sync` / `ovos.session.update_default`** —
  emitted by `SessionManager` to broadcast the current default
  session to interested components. SESSION-2 §6.4 acknowledges
  that an orchestrator MAY emit default-session state on a
  deployer-defined topic but assigns no normative name. These
  ad-hoc topics should be retired: any component that needs the
  default-session state can subscribe to `ovos.utterance.handled`
  (PIPELINE-1 §9.5) and read the session it carries, or listen
  to any other assistant-emitted Message on the default session.
  A named sync topic adds an implicit state-broadcast contract
  that the specs deliberately avoid; clients are expected to
  track session from Message flow, not from dedicated sync
  broadcasts.

### 5.3 Prescriptive shape changes

- **Keyword intent registration is atomic** (INTENT-4 §5).
  Today a keyword intent is built up via multiple
  `register_vocab` messages followed by a `register_intent`
  with an Adapt `IntentBuilder.__dict__` payload. INTENT-4
  collapses this into a single message with structured
  `{required, optional, one_of, excluded}` arrays of
  vocabulary descriptors. Every skill's keyword-intent path
  needs to be rewritten in the workshop layer.
- **Template intent registration uses structured identity**
  (INTENT-4 §6). Today `padatious:register_intent` carries
  `{name, samples, file_name, lang, blacklisted_words}`; the
  prescribed shape uses the structured `(skill_id,
  intent_name, lang)` triple plus `samples|file` and
  `blacklist|blacklist_file`.
- **Dispatch payload is minimal** (PIPELINE-1 §7.1). Today
  dispatch carries `skill_id` and `intent_name` in the
  payload. PIPELINE-1 drops both from the payload — they
  are already in the topic (`<skill_id>:<intent_name>`);
  a consumer that needs them splits the topic. The
  prescribed payload is `{lang, utterance, slots}`.
  For plugin-bundled handlers (`pipeline_id == skill_id`),
  the same uniform dispatch applies.
- **Handler-lifecycle payload updated** (PIPELINE-1 §8.2).
  Today the trio payload is `{name: <handler_func_name>}`.
  Prescribed: `{skill_id, intent_name, optional exception}`.

### 5.4 Architectural divergences

- **The orchestrator maintains a passive registration index**
  (INTENT-4 §10). Today there is no central index — each
  plugin knows what it consumed; nothing aggregates that
  view. INTENT-4 prescribes the orchestrator subscribe to
  all registration topics in parallel with plugins and serve
  `ovos.intent.list` / `ovos.intent.describe` from the
  passive view. This is a new orchestrator responsibility,
  not a change to existing behaviour.
- **The match contract is the single obligation** (PIPELINE-1
  §4.2). The plugin's `match` operation has one MUST: return
  a `Match` or `null`. Bus emissions during `match` are
  allowed — converse plugins, LLM-backed matchers, and
  agent-backed shapes are all conformant. Session mutation
  during `match` goes via `Match.updated_session` so
  declined matches' mutations never escape.
- **`Match.updated_session` as the match-phase session channel**
  (PIPELINE-1 §4.1, §4.2). Promotes the existing ovos-core
  code pattern
  `sess = match.updated_session or SessionManager.get(message)`
  to a normative Match field. The plugin that produces a
  claiming match composes any session mutations it needs
  (decrementing a response-mode counter, pre-promoting an
  active-handler to the head, setting intent_context
  alongside the match) into a fresh snapshot returned in
  `Match.updated_session`. The orchestrator uses that
  snapshot for the dispatch and every downstream stage; a
  declined-match (plugin returns `null`) drops the snapshot
  at the plugin boundary. This is what makes match-phase
  mutation safe under §6.2 first-match-wins iteration.
- **`ovos.utterance.handled` on every terminal path**
  (PIPELINE-1 §9.5). Current `ovos-workshop`'s
  `_on_event_error` does not emit it on the handler-error
  path (`ovos.py:1478-1497`). PIPELINE-1 §8 places trio
  emission on the orchestrator-wrapper around the handler,
  not on the handler itself — workshop is the wrapper in
  current OVOS, and the spec contract requires the wrapper
  to emit `ovos.utterance.handled` unconditionally.
- **Handler-trio is orchestrator-owned** (PIPELINE-1 §8).
  The orchestrator that invokes the handler wraps the call
  and emits `ovos.intent.handler.start` / `.complete` /
  `.error` around it. Third-party handler code carries **no
  normative obligation** to participate in trio emission.
  Skill authors are not protocol authors; the wrapper
  observes start / return / exception around an opaque
  callable.
- **Per-pipeline_id intent introspection** (PIPELINE-1 §10).
  Pull-query / scatter-response surface keyed on
  `pipeline_id`, giving consumers visibility into *which
  intents a particular pipeline plugin's matcher has
  compiled*, distinct from the orchestrator's manifest of
  declared intents (INTENT-4 §10). No current OVOS analogue.
- **CONTEXT-1 scope and ownership encoded in the key shape**
  (CONTEXT-1 §2, §3). A bare key `Person` is shared; a
  prefixed key `music.skill:Person` is private to
  `music.skill`. The `:` is load-bearing — mirroring the
  `<skill_id>:<intent_name>` dispatch topic. Drops separate
  `scope` and `origin` fields on stored entries (both were
  redundant with the key shape). `requires_context` and
  `excludes_context` declarations take an OPTIONAL
  `scope: private|shared` discriminator (default `private`)
  to express which lookup the gate uses; bare-string
  declarations default to private to prevent shared-leak.
- **Skill self-identification on every emission** (INTENT-4
  §3.1). Current OVOS skills set `context.skill_id` on some
  emissions but not uniformly. Enforcement is structural on
  the dispatch path: the orchestrator stamps
  `context.skill_id` from the `<skill_id>:<intent_name>`
  dispatch topic prefix, and skill emissions via
  `forward`/`reply` inherit automatically. Loader-side
  interception covers off-dispatch emissions.
- **Entry-point topic renamed `ovos.utterance.handle`**
  (PIPELINE-1 §9.1). `recognizer_loop:utterance` fails
  MSG-1 §2.1.2 naming conventions: `:` as a segment
  separator, an implementation-role prefix, and no pairing
  with the terminal `ovos.utterance.handled`. Migration cost
  is real — every audio-input service and intent-service
  handler is affected. A transitional deployment MAY
  subscribe to both names during migration.

### 5.5 New topics with no direct precedent

- **`ovos.intent.matched`** (PIPELINE-1 §9.2). The
  positive-match broadcast notification. No current equivalent.
- **`ovos.intent.unmatched`** (PIPELINE-1 §9.4). Renamed from
  `complete_intent_failure`; follows the `ovos.intent.*`
  namespace for symmetry with `ovos.intent.matched`.
- **`ovos.utterance.speak`** (PIPELINE-1 §9.6). The NL output
  exit point; symmetric to `ovos.utterance.handle`. No current
  equivalent — TTS trigger is currently implicit.
- **`ovos.intent.list` / `ovos.intent.describe`** (INTENT-4
  §10). Introspection topics served from the orchestrator's
  passive registration index.
- **`ovos.context.set` / `.unset` / `.clear` / `.list`**
  (CONTEXT-1 §5). Skill-facing API replacing Adapt-specific
  `add_context` / `remove_context` plus
  `mycroft.skill.set_cross_context`.
- **`ovos.transformer.{type}.list`** (TRANSFORM-1 §6).
  Per-type introspection of loaded transformers.
- **Materialize-default-session rule** on `forward` /
  `reply` / `response` (MSG-1 §4.3). Formalizes a "MAY"
  convenience for in-process subsystems; not currently
  implemented but compatible with current behaviour.

### 5.6 Things the specs do *not* change

- The session object's internal shape is owned by
  OVOS-SESSION-1; the field set is the closed set defined
  there plus whatever future specs claim via SESSION-1 §2.1.
  The "extra" fields current OVOS Session carries
  (`persona_id`, `system_unit`, `time_format`, `date_format`,
  …) ride through as non-normative pass-through and may be
  claimed by future per-domain specs.
- The `mycroft.*` topic prefix outside the intent layer (e.g.
  `mycroft.audio.*`) — these are not part of any spec here.
- The `<skill_id>:<intent_name>` dispatch topic — kept
  verbatim from current OVOS so no skill needs to migrate
  its handler subscription.
- **Engine-specific introspection topics.** The standard
  plugins expose their own debug / inspection topics — for
  example `intent.service.adapt.reply`,
  `intent.service.adapt.manifest`,
  `intent.service.adapt.vocab.manifest`, and
  `intent.service.padatious.get`. These are plugin-specific
  surface, parallel to the spec's generic
  `ovos.intent.list` / `ovos.intent.describe` (INTENT-4
  §10). The specs do not claim authority over them — they
  remain plugin-defined and may continue to coexist with
  the orchestrator's generic index.

### 5.7 Predecessor-topic mapping

The bus topics formalized by INTENT-4 and PIPELINE-1 replace
a number of legacy names. Implementer migration aid:

#### Registration topics (INTENT-4)

| Legacy topic | v1 replacement | Notes |
|--------------|---------------|-------|
| `register_vocab` | folded into `ovos.intent.register.keyword` | Vocabularies in v1 are inline `samples` or `file`-by-path inside the registration. |
| `register_intent` (Adapt parser) | `ovos.intent.register.keyword` | Adapt's `IntentBuilder.__dict__` payload replaced by the structured shape. |
| `padatious:register_intent` | `ovos.intent.register.template` | Same content, structured payload. |
| `padatious:register_entity` | `ovos.entity.register` | Entities are not Padatious-specific. |
| `detach_intent` | `ovos.intent.deregister` | Identity now expressed as the structured triple, not the munged `skill_id:intent_name` string. |
| `detach_skill` | `ovos.skill.deregister` | |
| `mycroft.skill.enable_intent` / `mycroft.skill.disable_intent` | `ovos.intent.enable` / `ovos.intent.disable` | First-class topics under v1, with the prefix dropped. |

#### Utterance-lifecycle topics (PIPELINE-1)

| Legacy topic | Status |
|--------------|--------|
| `recognizer_loop:utterance` | renamed to `ovos.utterance.handle` (see §5.4) |
| `complete_intent_failure` | renamed to `ovos.intent.unmatched` — follows `ovos.intent.*` namespace. |
| `ovos.utterance.cancelled` | **unchanged** — kept as the cancellation signal. |
| `ovos.utterance.handled` | **unchanged** — kept as the universal end-marker. |
| `<skill_id>:<intent_name>` | **unchanged** — dispatch topic; a plugin-bundled handler has `skill_id == pipeline_id`. |
| `mycroft.skill.handler.start` / `.complete` / `.error` | renamed to `ovos.intent.handler.start` / `.complete` / `.error` |

#### Out of scope

| Legacy topic | Status |
|--------------|--------|
| `add_context` / `remove_context` | Replaced by `ovos.context.set` / `.unset` under CONTEXT-1. |
| `mycroft.skill.set_cross_context` / `remove_cross_context` | Replaced by `ovos.context.set` / `.unset` with `scope: "shared"` under CONTEXT-1. |
| `<skill_id>.activate` | Activity-tracking emit currently in `ovos-core`; not part of any spec here. |

#### Listening-lifecycle topics (AUDIO-IN-1)

| Legacy topic | v2 replacement | Notes |
|--------------|---------------|-------|
| `recognizer_loop:record_begin` | `ovos.mic.record.started` | Capture start. `:` segment separator and implementation-role prefix dropped; no payload. |
| `recognizer_loop:record_end` | `ovos.mic.record.ended` | Capture end; pairs with the start signal. |
| `recognizer_loop:sleep` | `ovos.mic.sleep` | Controller-to-listener sleep request. |
| `mycroft.awoken` | `ovos.mic.awoken` | Sleep→awake transition; moved into the `ovos.mic.*` namespace. |
