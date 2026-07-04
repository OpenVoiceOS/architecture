---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 7. Known gaps

- **Per-plugin behavioural specs.** OVOS-PIPELINE-1 defines
  the plugin contract (the `match` shape, the orchestrator's
  iteration semantics) but explicitly defers what each
  non-trivial plugin type actually *does*. `converse` (OVOS-CONVERSE-1),
  `stop` (OVOS-STOP-1), `common_query` (OVOS-COMMON-QUERY-1),
  `fallback` (OVOS-FALLBACK-1), `persona` (OVOS-PERSONA-1), and
  the media player (OVOS-OCP-1) have their own specs. Each
  defines its own internal behaviour and its own bus emissions
  beyond the universal lifecycle PIPELINE-1 prescribes.
- **Session preference fields not claimed by a spec.**
  SESSION-1 defines the wire shape and OVOS-SESSION-2 defines
  the lifecycle and state-ownership model; what remains
  deferred is the full set of session preferences OVOS
  carries (`time_format`, `date_format`, `system_unit`,
  `tts_preferences`, `location`, …) — these need to be
  claimed under SESSION-1 §2.2's field registry by their
  respective owning specs (a preferences spec, OCP / locale
  specs as appropriate). `persona_id` is already claimed by
  OVOS-PERSONA-1 §3.
- **Text normalization of ASR output.** The basis for slot
  value typing (INTENT-1 §5.3). Deferred to its own
  specification.
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
- **i18n corpus.** OVOS-INTENT-2 defines the locale file
  format, and `ovos-localize` (§1.4) provides the
  operations layer; what remains is the *scale* of the
  translated corpus.

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
- **Audio transmission over the bus.** BRIDGE-1 §4.2.1 describes
  two audio-stack placements — local (satellite runs STT and its
  own audio-output layer) and hub-side (hub runs the full audio
  stack, satellite transmits raw audio inbound and receives final
  audio outbound). STT placement and audio-output placement are
  symmetric and independent: each can live on the satellite or the
  hub, giving four possible combinations. The hub-side model
  requires transmitting audio as bus Message payloads (e.g.
  base64-encoded PCM or compressed frames). The **outbound**
  half is covered: OVOS-AUDIO-1 §3.4 (`ovos.utterance.speak.b64`)
  and §4.3 (`ovos.audio.speech`) define base64 delivery of
  synthesised audio to remote clients. The **inbound** half —
  transmitting captured microphone audio from a satellite to a
  hub-side STT, with session fields for codec and audio
  preferences — remains undefined.
- **Session-scoped pipeline plugin registration.** BRIDGE-1 §4.4
  and INTENT-4 §11 cover session-scoped intent registration for
  satellite-side skills. A satellite that implements a pipeline
  plugin (not an intent-based skill) cannot register that plugin
  on the hub; no bus surface exists for session-scoped plugin
  loading. If needed, this requires a new specification or an
  extension to OVOS-PIPELINE-1.
- **Managing mode concurrent utterance race.** BRIDGE-1 §3.4.2
  says the bridge SHOULD apply `ovos.utterance.handled` session
  updates before injecting the next utterance, but makes no
  guarantee when both arrive simultaneously. The handling of
  overlapping utterance rounds in managing mode — whether to
  queue, drop, or process with a stale session — is left as a
  deployment concern. A revision may define a normative
  queuing policy.
- **NAT bijection and hub-side session cleanup.** When a bridge
  using `session_id` NAT (§3.2) disconnects a participant, the
  hub-side `session_id` may remain in the orchestrator's
  default-session store (SESSION-2 §5). BRIDGE-1 §3.2 says the
  bridge SHOULD emit cleanup events using the hub-side `session_id`
  before dropping the bijection, but does not define a complete
  cleanup protocol for hub-side state created during the session's
  lifetime (e.g. cross-utterance context, active handlers).
  Deferred to a separate session-lifecycle specification.
