---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 7. Known gaps

- **Session preference fields not claimed by a spec.**
  SESSION-1 defines the wire shape and OVOS-SESSION-2 defines
  the lifecycle and state-ownership model; `persona_id` is
  claimed by OVOS-PERSONA-1. What remains deferred is the rest
  of the session preferences OVOS carries (`time_format`,
  `date_format`, `system_unit`, `tts_preferences`, `location`,
  …) — these need to be claimed under SESSION-1 §2.2's field
  registry by their respective owning specs (a preferences
  spec, OCP / locale specs as appropriate).
- **Text normalization of ASR output.** The basis on which a
  typed slot's value is computed (INTENT-1 §5.6). Deferred to
  its own specification.
- **A machine-checkable conformance corpus** of `template →
  sample set` pairs for INTENT-1 expansion, so expander
  conformance can be verified automatically. A parallel
  corpus of bus-message fixtures for MSG-1 would be the
  equivalent at the bus layer.
- **An end-to-end worked example.** The specs have local
  examples; none shows a single skill defining one keyword
  intent and one template intent through the whole path —
  files, registration, match, handler.
- **Conversation-level evaluation infrastructure.** Rasa
  has story-based testing and end-to-end success metrics;
  the OVOS specs have no counterpart.
- **OVOS-INTENT-2 ↔ hassil `intents` translation tool.**
  The grammar lineage (§2.1) makes a mechanical translator
  between OVOS-INTENT-2 locale resources and HA's `intents`
  YAML feasible. Such a tool would let the two corpora
  cross-pollinate without either format changing. Sits at
  injection point 3 of §3.3 conceptually but is
  build-time rather than runtime tooling.
- **GUI interactive companions.** GUI-1 §3.4 reserves the
  names `SYSTEM_confirm` (a yes/no companion to a spoken
  question) and `SYSTEM_select` (a choice companion to a spoken
  set of options) without defining them. Both need a return
  channel from the display back to the assistant, and no such
  channel is specified — defining the templates without it
  would leave every backend inventing its own. The names are
  held so that no application-defined template claims them; a
  future version defines the pair together with the return
  path.
- **An identifier in a dotted topic, at one surface.**
  MSG-1 §2.1.1 confines identifiers to the
  `<skill_id>:<intent_name>` dispatch shape and makes every
  dotted topic a static string. PIPELINE-1 §10's per-plugin
  introspection surface,
  `ovos.pipeline.<pipeline_id>.intents.list`, still builds a
  topic from a runtime identifier, which is the one place the
  family's topic surface is not enumerable from the specs
  alone. Whether the surface moves to a static topic with
  `pipeline_id` in the payload — the shape the poll families
  took — is unresolved.
- **i18n corpus.** OVOS-INTENT-2 defines the locale file
  format, and `ovos-localize` (§1.4) provides the
  operations layer; what remains is the *scale* of the
  translated corpus.
- **Scheduled events were unspecified.** No spec previously
  claimed the timed and recurring event surface a voice OS
  needs for alarms, timers, and reminders. OVOS-SCHEDULER-1 now
  claims it; the legacy `mycroft.scheduler.*` protocol is
  mapped to it, with divergences and an adoption roadmap, in
  [appendix/scheduler.md](scheduler.md).

### Bridge-specific gaps (OVOS-BRIDGE-1)

- **Multi-hop bridge cascading.** BRIDGE-1 §4.2 describes
  peer-to-peer topology between two deployments, but does not
  address cascaded bridges (deployment A ↔ bridge ↔ deployment B
  ↔ bridge ↔ deployment C). Each bridge stamps source
  independently; outer-deployment participant identity is lost
  at each inner boundary unless propagated as opaque metadata.
- **Bridge-to-bridge wire format.** The spec does not prescribe
  whether messages between peer bridges remain in MSG-1 envelope
  form or may use a different serialization. The intent is that
  they stay in MSG-1 form (per §4.2 "relays native bus messages"),
  but the transport encoding is left as a deployment concern.
- **Bridge health / heartbeat.** No `ovos.bridge.ping` /
  `ovos.bridge.pong` surface is defined. The orchestrator has no
  spec-level way to know whether a remote participant is
  reachable. Deferred to a separate observability specification
  if needed.
- **Outbound audio over the bus is specified; inbound is not.**
  BRIDGE-1 §4.2.1 describes two audio-stack placements — local
  (satellite runs STT and its own audio-output layer) and
  hub-side (hub runs the full audio stack). On the output side
  the bus surface is defined by OVOS-AUDIO-1:
  `ovos.utterance.speak.b64` → `ovos.audio.speech` for
  remote-client TTS delivery, and base64 `audio` payloads on
  `ovos.audio.queue` / `ovos.audio.play_sound` for sound
  effects. OVOS-AUDIO-IN-1 §1 explicitly scopes audio
  *capture* out ("acquisition mechanism is deployer-defined"),
  so what remains undefined is the *inbound* leg of the
  hub-side model — transmitting raw captured audio from the
  satellite to a hub-side STT as bus Message payloads (topic
  names, payload shape, session fields for codec and audio
  preferences).
- **Session-scoped pipeline plugin registration.** BRIDGE-1 §4.4
  and INTENT-4 §11 cover session-scoped intent registration for
  satellite-side skills. A satellite that implements a pipeline
  plugin (not an intent-based skill) cannot register that plugin
  on the hub; no bus surface exists for session-scoped plugin
  loading. If needed, this requires a new specification or an
  extension to OVOS-PIPELINE-1.
- **Managing mode concurrent utterance race.** BRIDGE-1 §3.4.2
  says the bridge SHOULD apply `ovos.utterance.handled` session
  updates before injecting the next utterance, and already
  defines a MAY-fallback for the race — inject using the last
  known session state, with the orchestrator supplying the
  updated session on the following `ovos.utterance.handled`.
  What is left open is the policy for a *second* utterance
  arriving before the first round resolves: whether the bridge
  queues it, drops it, or forwards it against the stale session.
  A revision may define a normative queuing policy.
- **NAT bijection and hub-side session cleanup.** When a bridge
  using `session_id` NAT (§3.2) disconnects a participant, the
  hub-side `session_id` may remain in the orchestrator's
  default-session store (SESSION-2 §5). BRIDGE-1 §3.2 says the
  bridge SHOULD emit cleanup events using the hub-side `session_id`
  before dropping the bijection, but does not define a complete
  cleanup protocol for hub-side state created during the session's
  lifetime (e.g. cross-utterance context, active handlers).
  Deferred to a separate session-lifecycle specification.
