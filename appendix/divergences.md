---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 5. Where the specs differ from the reference implementation

These specifications are *prescriptive*. Some of what they
prescribe matches the reference implementation verbatim; some
is a deliberate cleanup from which the implementation diverges.
This section catalogues the known divergences between the
specifications and the reference implementation.

### 5.1 Already aligned

Formalizations of behaviour in the reference implementation
that need no change:

- The Message envelope (`type` / `data` / `context`) — matches
  `ovos-bus-client.Message`.
- `source` and `destination` as the routing pair, and the
  `Message.reply` swap that owns it — matches
  `ovos-bus-client/message.py`. The one point of departure is the
  array-valued `destination` the shipped swap still accepts; see
  §5.3.
- `ovos.common_query.ping` / `ovos.common_query.pong` — static
  broadcast poll topics with the responder's identity in
  `data.skill_id`. Match
  `ovos-common-query-pipeline-plugin/opm.py` and
  `ovos-workshop/skills/ovos.py`'s
  `__handle_common_query_ping` verbatim; the shipped pair is the
  model the other poll families were reshaped toward (§5.4).
- Player times counted in **milliseconds** (GUI-1 §3.4
  `SYSTEM_audio_player` / `SYSTEM_media_player`) — matches
  `ovos-media/media_backends/base.py`, which already carries
  `position` and seek targets in ms.
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
  topics.
- `ovos.utterance.cancelled` and `ovos.utterance.handled`
  (PIPELINE-1) — match current topic names verbatim.
- Per-utterance first-match-wins iteration (PIPELINE-1) —
  matches `ovos-core/intent_services/service.py`'s
  `handle_utterance` / `get_pipeline`.
- Per-session pipeline configuration (PIPELINE-1) — matches
  `Session.pipeline`.
- The `<skill_id>:<intent_name>` dispatch topic shape
  (PIPELINE-1) — matches OVOS practice; skills subscribe to
  these topics.

### 5.2 Prescriptive renames

| Spec | Current | Prescribed | Notes |
|------|---------|------------|-------|
| INTENT-3 v1.1 | "host" | "orchestrator" | Editorial; conformance unchanged. |
| PIPELINE-1 | `mycroft.skill.handler.start` / `.complete` / `.error` | `ovos.intent.handler.start` / `.complete` / `.error` | Renamed into the `ovos.intent.*` namespace for uniformity. Breaks every existing handler-lifecycle observer; the migration cost is real. |
| PIPELINE-1 | `recognizer_loop:utterance` | `ovos.utterance.handle` | See §5.4 entry. Migration touches `ovos-dinkum-listener`, `ovos-simple-listener`, `ovos-audio`, and `ovos-core/intent_services/service.py`. |
| PIPELINE-1 | `complete_intent_failure` | `ovos.intent.unmatched` | Follows `ovos.intent.*` namespace; pairs with `ovos.intent.matched`. |
| COMMON-QUERY-1 | `question:query` / `question:query.response` | `ovos.common_query.request` / `ovos.common_query.response` | The shipped plugin broadcasts `question:query` and collects `question:query.response`, correlating answers by the `phrase` string. Both names fail MSG-1 §2.1.1 — `:` outside the dispatch shape, and a root that names neither the ecosystem nor the domain. The v1 pair is two **static** topics; the skill being asked is named in `data.skill_id` (COMMON-QUERY-1 §7.1.1), never in the topic. The one colon topic kept is the dispatch `<pipeline_id>:common_query`. |

### 5.2.1 Topics to remove from ovos-core

The following topics exist in current ovos-core but are **not
defined by any spec** and should be removed or replaced:

- **`ovos.session.update_default`** —
  emitted by `SessionManager` to broadcast the default
  session. SESSION-2 §7 acknowledges that a deployment MAY publish
  default-session state on a topic of its own as a diagnostic, but
  assigns no normative name and no consumer. This ad-hoc topic is
  retired: a component that needs the default-session state
  subscribes to `ovos.utterance.handled` (PIPELINE-1 §9.5) and reads
  the session it carries, or adopts the session of any other
  assistant-emitted Message on the default session (SESSION-2 §2.7).
  See §5.7 for the migration mapping and §5.5 for the removed-mechanism
  record.

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
- **`destination` is a single string** (MSG-1 §3.3). A Message
  addresses one consumer or all of them; there is no
  multi-address form, and a producer wanting several named
  consumers emits one Message each. Current
  `ovos-bus-client/message.py` accepts an array: the `reply`
  swap tests `isinstance(dst, list)` and takes `dst[0]` as the
  new `source`, silently dropping every other member. Any
  deployment that relies on the array form loses recipients on
  the first swap already; under v1 the shape is simply not
  available.
- **Poll correlation is `context.utterance_id`** (PIPELINE-1 §9.1.1,
  COMMON-QUERY-1 §6.4, FALLBACK-1 §6.1, CONVERSE-1 §4.2). No
  poll family mints a correlation key of its own — the earlier
  COMMON-QUERY-1 `query_id` is removed. Nothing in shipped code
  carries the field: the common-query plugin matches answers by
  the `phrase` string, and `ovos-bus-client`'s
  `CollectionMessage` carries a `(handler_id, query_id)` pair
  that the derivations do not propagate. See §5.4 for the field
  itself.
- **A wrong-typed session field is treated as omitted, loudly**
  (SESSION-1 §2). A field whose value is of the wrong type is
  filled from the default as if absent, and the violation
  **SHOULD** be logged at WARN naming both the field and the type
  received. Current `Session` deserialization accepts whatever it
  is handed and produces no diagnostic, so a typo'd client field
  fails silently for the life of the session.

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
  (clearing or setting `session.response_mode`, pre-promoting an
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
  MSG-1 §2.1.1 naming conventions: `:` as a segment
  separator, an implementation-role prefix, and no pairing
  with the terminal `ovos.utterance.handled`. Migration cost
  is real — every audio-input service and intent-service
  handler is affected. A transitional deployment MAY
  subscribe to both names during migration.
- **Transformer priority is ascending** (TRANSFORM-1 §4).
  The reference implementation sorts transformer chains
  descending (`reverse=True`), where the lowest number runs
  *last* and — because each transformer's output overwrites
  its predecessor's — effectively "wins". That is the exact
  inverse of the spec's ascending convention (lower = earlier,
  default 50); an inverse-convention priority assignment MUST
  be renumbered, since run unrenumbered the chain executes
  backwards. The TRANSFORM-1 notes in
  [rationale.md](rationale.md) record why ascending was chosen.
- **`context.utterance_id` is the universal correlation key**
  (PIPELINE-1 §9.1.1). The orchestrator stamps it exactly once at
  lifecycle entry — the utterance arrival, the out-of-band
  request, the UI command — and it propagates across `forward` /
  `reply` / `response` under the same MUST that governs
  `session`. Everything derived from that lifecycle carries the
  same value: transformer passes, the pipeline contest, every
  poll and pong, the dispatch, the terminal events. Two Messages
  belong to the same lifecycle iff their `utterance_id` matches,
  and that is the whole correlation rule — the poll specs
  (FALLBACK-1, COMMON-QUERY-1, CONVERSE-1) define nothing of
  their own. Nothing in current OVOS has an equivalent: no
  component stamps a lifecycle identifier, so a plugin holding a
  poll window today separates this round's answers from the
  previous round's by the window's own timing alone. A component
  **MUST NOT** overwrite an `utterance_id` already present, which
  makes the field the one piece of `context` a bridge or a
  transformer must carry through untouched.
- **Identifiers appear in topics only in the dispatch shape**
  (MSG-1 §2.1.1). `<skill_id>:<intent_name>` is the single
  identifier-bearing topic shape and the single one a consumer
  parses; every dotted topic is a **static string** fixed by the
  specification that defines it. Addressing beyond the topic is
  payload, and the routing pair belongs to the `reply` swap
  alone. The consequence is that a monitor, a bridge allowlist,
  or a conformance harness can enumerate a deployment's complete
  topic surface from the specifications. Current OVOS mints
  topics from identifiers freely — `<skill_id>.converse.ping`,
  `<skill_id>.stop.ping`, `<skill_id>.fallback.ping` (in
  flight), `question:action.<skill_id>`, `<skill_id>.activate` —
  so no such enumeration is possible today, and each poll family
  needed its own reshaping.
- **The poll families are broadcast contests on static topics**
  (STOP-1 §4.2, CONVERSE-1 §4.2, FALLBACK-1 §6, COMMON-QUERY-1
  §6). One ping per round on a static topic, parallel pongs
  correlated by `utterance_id`, explicit declines, a bounded
  window that closes early once nothing still unanswered can
  change the outcome, and selection in pool order rather than
  arrival order. `ovos.stop.ping` / `.pong` was the model;
  converse and fallback were reshaped to match it, and common
  query already had the shape at the ping stage. Current OVOS
  runs each family differently and none of them this way — §5.7
  carries the topic mapping. Two consequences carry the weight:
  - Candidacy stops travelling in the topic. A converse
    candidate self-checks its own `skill_id` against
    `context.session.converse_handlers` (CONVERSE-1 §4.2)
    instead of being addressed by name; a common-query skill
    checks `data.skill_id`. Shipped `ovos-core`'s
    `converse_service.py` and `stop_service.py` instead emit one
    addressed ping **per candidate** in a loop, so a round costs
    as many Messages as there are candidates and the candidate
    set is fixed at emit time rather than read from the session
    the ping carries.
  - The pong stops being a second, differently-shaped name.
    Shipped skills answer `<skill_id>.converse.ping` on the
    shared `skill.converse.pong`, and `<skill_id>.stop.ping` on
    `skill.stop.pong` — a per-skill request paired with a
    broadcast reply. Under v1 both halves are static and
    symmetric.
- **The in-flight fallback implementation diverges from
  FALLBACK-1 as well as V0 does.** `ovos-core#808` and
  `ovos-workshop#465` replace the broadcast
  `ovos.skills.fallback.ping` with a skill-addressed
  `<skill_id>.fallback.ping` / `<skill_id>.fallback.pong` pair,
  keeping the broadcast topic bound alongside it. That is the
  opposite direction from FALLBACK-1 §6, which prescribes one
  broadcast `ovos.fallback.ping` per round, answered in parallel
  on `ovos.fallback.pong`. The divergence is catalogued here
  because that work is unmerged and reads as conformance work:
  it is not.
- **The persona pipeline is stoppable under its own
  `pipeline_id`** (PERSONA-1 §6, §8.6). The persona pipeline answers
  the STOP-1 ping affirmatively for a session it holds a
  conversation on, and is stopped by the ordinary per-skill stop
  dispatch addressed to its `pipeline_id` — no drain-list entry
  and no stop-plugin configuration is involved. Current OVOS has
  no persona pipeline in the stop cascade at all, so there is no
  shipped behaviour to change; what the ruling forecloses is the
  alternative shape, in which the stop plugin would have had to
  know that persona exists.
- **Enable and disable are cross-skill control messages**
  (INTENT-4 §3.2, §8.5). The payload-identity check — payload
  `skill_id` **MUST** equal `context.skill_id`, or the message
  is rejected — is scoped to registration and deregistration,
  where a mismatch would be a remote uninstall. `ovos.intent.enable`
  and `ovos.intent.disable` are exempt: the payload names the
  **target**, `context.skill_id` names the **source**, and the two
  legitimately differ, because an admin UI or a conflict-resolving
  skill suppressing another skill's intent is the point of the
  surface. An orchestrator **MAY** block cross-skill control as
  deployment hardening; the policy's shape is deployment-defined.
  Cross-*session* control needs no field of its own — the message
  affects the scope of whichever session its `context` declares.
  Current `mycroft.skill.enable_intent` / `disable_intent` carry no
  identity check in either direction, and no notion of a target
  distinct from a source.

### 5.5 New topics with no direct precedent

- **`ovos.intent.matched`** (PIPELINE-1 §9.2). The
  positive-match broadcast notification. No current equivalent.
- **`ovos.intent.unmatched`** (PIPELINE-1 §9.3). Renamed from
  `complete_intent_failure`; follows the `ovos.intent.*`
  namespace for symmetry with `ovos.intent.matched`.
- **`ovos.utterance.speak`** (PIPELINE-1 §9.6). The NL output
  exit point; symmetric to `ovos.utterance.handle`. No current
  equivalent — TTS trigger is currently implicit.
- **`ovos.utterance.speak.b64`** (AUDIO-1 §3.4). Variant of
  `ovos.utterance.speak` for remote-client delivery: the audio
  output service runs the same TTS pipeline but emits synthesised
  audio as base64 via `ovos.audio.speech` instead of queuing for
  local playback. Used by bridges serving satellites without TTS
  (BRIDGE-1 §4.2.4).
- **`ovos.audio.speech`** (AUDIO-1 §4.3). Base64-encoded
  synthesised audio broadcast; emitted in response to
  `ovos.utterance.speak.b64`. Carries a `listen` flag. Remote
  clients (e.g. satellites relayed by a bridge) decode and play
  the audio themselves.
- **`ovos.audio.queue`** / **`ovos.audio.play_sound`** (AUDIO-1
  §4.1, §4.2). Sound-effect playback topics. Payloads accept
  either a `uri` or inline base64 `audio` field, enabling
  cross-host audio delivery without shared filesystem access.
- **`ovos.stt.failed`** (AUDIO-IN-1 §5). Terminal event a MUST
  when a capture yields no usable transcription: the service
  emits it *instead of* `ovos.utterance.handle`, never both, and
  no handler lifecycle follows. `ovos.utterance.handle` with an
  empty or phantom `utterances` list is non-conformant, which is
  what makes the failure observable rather than a silence a
  client has to distinguish from "still transcribing". The
  payload MAY be empty; `context.session` rides along like every
  other emission from the service. Current
  `ovos-dinkum-listener` emits
  `recognizer_loop:speech.recognition.unknown` — a name that
  fails MSG-1 §2.1.1 on the `:` separator and the
  implementation-role root, and one no other listener is
  obliged to emit. See §5.7 for the mapping.
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
  `reply` / `response` (MSG-1 §5). Formalizes a "MAY"
  convenience for in-process subsystems; compatible with
  existing behaviour.

#### Removed mechanisms — session push topics

The specs define **no topic on which any participant pushes a
session at another**. Two mechanisms in shipped code do exactly
that, and neither has a home in the spec family. This appendix is
the one place that record lives; the specs themselves describe only
what exists.

- **`ovos.session.sync`** — a bare session broadcast. Shipped code
  implements `handle_session_sync` and a bare-sync bootstrap on bus
  connect. The spec has no equivalent: a named session is
  client-authoritative (SESSION-2 §2.5), so a server-side push
  fights the ownership model and is overwritten by the next inbound
  client Message. Every legitimate mutation has an in-lifecycle
  boundary instead (SESSION-2 §2.6), and a proactive interaction
  propagates by the adoption rule (SESSION-2 §3.2): the client
  adopts the session of any user-facing Message it renders.
- **`ovos.session.update_default`** — a default-session broadcast
  (see §5.2.1), emitted from `connect_to_bus` via
  `_broadcast_default_session`. The spec converges the default
  session two other ways (SESSION-2 §2.7): co-located processes
  derive their initial view from the same deployment configuration,
  and thereafter adopt the session of the Messages they act on or
  render. That leaves one writer per session and removes the
  multi-writer clobber the broadcast makes possible.

**V1 removal scope.** `handle_session_sync` and its subscription;
the bare-sync bootstrap on connect; `_broadcast_default_session` and
its `connect_to_bus` call site. The only bridging behaviour code
keeps during migration is the connect handshake: a bare sync on
connect maps to `ovos.session.update_default` until both mechanisms
are gone.

### 5.6 Things the specs do *not* change

- The session object's internal shape is owned by
  OVOS-SESSION-1; the field set is the closed set defined
  there plus whatever future specs claim via SESSION-1 §2.2.
  The "extra" fields current OVOS Session carries
  (`persona_id`, `system_unit`, `time_format`, `date_format`,
  …) ride through as non-normative pass-through and may be
  claimed by future per-domain specs.
- The `mycroft.*` topic prefix outside the intent layer (e.g.
  `mycroft.audio.*`) — these are not part of any spec here.
- The `<skill_id>:<intent_name>` dispatch topic — kept
  verbatim from current OVOS so no skill needs to migrate
  its handler subscription. Under MSG-1 §2.1.1 it is now also
  the *only* topic shape built from identifiers; keeping it
  unchanged is what let every other identifier-bearing topic go
  static (§5.4).
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
a number of predecessor names. The mapping:

#### Registration topics (INTENT-4)

| Predecessor topic | v1 replacement | Notes |
|--------------|---------------|-------|
| `register_vocab` | folded into `ovos.intent.register.keyword` | Vocabularies in v1 are inline `samples` or `file`-by-path inside the registration. |
| `register_intent` (Adapt parser) | `ovos.intent.register.keyword` | Adapt's `IntentBuilder.__dict__` payload replaced by the structured shape. |
| `padatious:register_intent` | `ovos.intent.register.template` | Same content, structured payload. |
| `padatious:register_entity` | `ovos.entity.register` | Entities are not Padatious-specific. |
| `detach_intent` | `ovos.intent.deregister` | Identity now expressed as the structured triple, not the munged `skill_id:intent_name` string. |
| `detach_skill` | `ovos.skill.deregister` | |
| `mycroft.skill.enable_intent` / `mycroft.skill.disable_intent` | `ovos.intent.enable` / `ovos.intent.disable` | First-class topics under v1, with the prefix dropped. |

#### Utterance-lifecycle topics (PIPELINE-1)

| Predecessor topic | Status |
|--------------|--------|
| `recognizer_loop:utterance` | renamed to `ovos.utterance.handle` (see §5.4) |
| `complete_intent_failure` | renamed to `ovos.intent.unmatched` — follows `ovos.intent.*` namespace. |
| `ovos.utterance.cancelled` | **unchanged** — kept as the cancellation signal. |
| `ovos.utterance.handled` | **unchanged** — kept as the universal end-marker. |
| `<skill_id>:<intent_name>` | **unchanged** — dispatch topic; a plugin-bundled handler has `skill_id == pipeline_id`. |
| `mycroft.skill.handler.start` / `.complete` / `.error` | renamed to `ovos.intent.handler.start` / `.complete` / `.error` |
| `ovos.session.update_default` | **retire** — subscribe to `ovos.utterance.handled` (PIPELINE-1 §9.5) to read updated default-session state; or adopt the session of any assistant-emitted Message on the default session (SESSION-2 §2.7). See §5.2.1 and §5.5. |
| `ovos.session.sync` (incl. the bare-sync connect bootstrap) | **retire** — no spec defines a session push topic; session mutates at the SESSION-2 §2.6 boundaries and converges by adoption (SESSION-2 §2.7, §3.2). See §5.5. |

#### Poll-family topics (STOP-1, CONVERSE-1, FALLBACK-1, COMMON-QUERY-1)

Every poll family resolves to a static ping / pong pair, with the
responder's identity in `data.skill_id` and the round identified by
`context.utterance_id` (§5.4).

| Predecessor topic | v1 replacement | Notes |
|--------------|---------------|-------|
| `<skill_id>.stop.ping` | `ovos.stop.ping` | One broadcast for the round instead of one addressed ping per active handler. Silence is not a decline: with no positive pong the plugin falls back to recency over `session.active_handlers` (STOP-1 §4.1), where shipped `stop_service.py` defaults a non-responder to `False` and can end the round with no target at all. |
| `skill.stop.pong` | `ovos.stop.pong` | Already a shared reply topic; renamed for symmetry with the ping. |
| `<skill_id>.converse.ping` | `ovos.converse.ping` | Candidacy moves from the topic to a `session.converse_handlers` membership self-check (CONVERSE-1 §4.2). |
| `skill.converse.pong` | `ovos.converse.pong` | Shared reply topic renamed; payload gains `skill_id`, `result`, and an optional `error_code`. |
| `ovos.skills.fallback.ping` | `ovos.fallback.ping` | Already a broadcast; renamed into the `ovos.fallback.*` root. Declines become explicit so the window can close early. |
| `ovos.skills.fallback.pong` | `ovos.fallback.pong` | As above. Note the in-flight `<skill_id>.fallback.ping` / `.pong` work (§5.4) moves *away* from this shape. |
| `ovos.common_query.ping` / `.pong` | **unchanged** | Already static broadcast topics with identity in the payload. |
| `question:query` | `ovos.common_query.request` | Static topic; the skill being asked is named in `data.skill_id`. See §5.2. |
| `question:query.response` | `ovos.common_query.response` | Answer correlated by `utterance_id`, not by the `phrase` string. |

#### Listening-failure topic (AUDIO-IN-1)

| Predecessor topic | v2 replacement | Notes |
|--------------|---------------|-------|
| `recognizer_loop:speech.recognition.unknown` | `ovos.stt.failed` | Terminal event when STT produced no usable transcription; a MUST rather than a listener-specific courtesy. See §5.5. |

#### Out of scope

| Predecessor topic | Status |
|--------------|--------|
| `add_context` / `remove_context` | Replaced by `ovos.context.set` / `.unset` under CONTEXT-1. |
| `mycroft.skill.set_cross_context` / `remove_cross_context` | Replaced by `ovos.context.set` / `.unset` with `scope: "shared"` under CONTEXT-1. |
| `<skill_id>.activate` | Activity-tracking emit currently in `ovos-core`; not part of any spec here. |

#### Listening-lifecycle topics (AUDIO-IN-1)

| Predecessor topic | v2 replacement | Notes |
|--------------|---------------|-------|
| `recognizer_loop:record_begin` | `ovos.listener.record.started` | Capture start. `:` segment separator and implementation-role prefix dropped; no payload. |
| `recognizer_loop:record_end` | `ovos.listener.record.ended` | Capture end; pairs with the start signal. |
| `recognizer_loop:sleep` | `ovos.listener.sleep` | Controller-to-listener sleep request. |
| `mycroft.awoken` | `ovos.listener.awoken` | Sleep→awake transition; moved into the `ovos.listener.*` namespace. |

### 5.8 Bus bridge (BRIDGE-1)

- **BRIDGE-1 defines source-stamping and destination-based routing;
  current HiveMind bridges route primarily by session_id.**
  HiveMind groups messages by `session_id` and delivers them to the
  peer that owns that session. BRIDGE-1 prescribes `destination` as
  the primary signal because two peers sharing the same `session_id`
  (including `"default"`) cannot be distinguished by session_id
  alone. HiveMind deployments that use per-peer `session_id`s are
  conformant with either model; deployments that share the
  `"default"` session across multiple peers must migrate to
  destination-based routing for client isolation.
- **Poll-family broadcasts are filtered by what a participant
  hosts, not by addressing.** BRIDGE-1 §3.2 enumerates the poll
  families — `ovos.converse.ping` / `.pong`,
  `ovos.common_query.ping` / `.pong` / `.request` / `.response`,
  `ovos.fallback.ping` / `.pong`, `ovos.stop.ping` / `.pong` —
  and forbids relaying any of them to a participant with **no
  skill registered in the Message's session**: a pure client has
  nothing a poll could address, and relaying to it both exposes
  internal handler enumeration and invites answers to polls it
  cannot serve. A participant that does host skills in the
  session receives the broadcasts unfiltered and its skills
  decide for themselves. The rule exists because the `reply`
  derivation gives a poll a `destination`, so destination-based
  routing alone would relay it. Current HiveMind bridges apply
  no such filter; a satellite that hosts no skills still sees
  the poll traffic of every session it shares.
- **No existing implementation fully conforms to OVOS-BRIDGE-1.**
  The bridge spec formalizes a role that exists in deployments
  (the HiveMind gateway, the bus client, any inbound message
  fan-in) with a tighter normative core — source-stamping and
  session-preservation requirements — than current implementations
  provide.
