---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 11. Role-to-repository map and per-spec adoption status

This section names, for each role the specifications address, the
OpenVoiceOS repository that plays it, and states an adoption status
for each specification. Product names are permitted here, unlike in
the spec bodies. Nothing in this section binds a spec's conformance
criteria.

### 11.1 Role-to-repository map

| Role | Repository |
|------|------------|
| Orchestrator (PIPELINE-1) | `ovos-core` |
| Message relay / bus daemon (MSG-1, SESSION-1 §2.5) | unassigned |
| Session carrier and default-session client (SESSION-1, SESSION-2) | `ovos-bus-client` |
| Skill host (INTENT-3, workshop layer) | `ovos-workshop` |
| Intent engine, keyword (INTENT-4) | `ovos-adapt-pipeline-plugin` |
| Intent engine, template (INTENT-4) | `ovos-padatious-pipeline-plugin` |
| Pipeline plugin, converse (CONVERSE-1) | `ovos-core` |
| Pipeline plugin, stop (STOP-1) | `ovos-core` |
| Pipeline plugin, fallback (FALLBACK-1) | `ovos-core` |
| Pipeline plugin, common query (COMMON-QUERY-1) | `ovos-common-query-pipeline-plugin` |
| Pipeline plugin, persona (PERSONA-1) | `ovos-persona` |
| Transformer chains (TRANSFORM-1) | `ovos-core` |
| Audio input service (AUDIO-IN-1) | `ovos-dinkum-listener` |
| Audio output service (AUDIO-1) | `ovos-audio` |
| Media player (OCP-1) | `ovos-media` |
| GUI service (GUI-1) | `ovos-gui` |
| Bus bridge (BRIDGE-1) | `HiveMind-core` |
| Scheduler (SCHEDULER-1) | `ovos-bus-client` |

No shipped relay rejects a malformed session carrier the way
SESSION-1 §2.5 requires. The rejection duty exists only in
`ovos-bus-client`'s fake-bus test double. `ovos-messagebus`'s own
description names it a bus daemon, not a validating relay, so the
message relay role stays unassigned. A future edition of this table
reassigns that row the day a relay's own documentation claims the
duty.

The scheduler and the default-session client both land on
`ovos-bus-client`. The reference scheduling utility and the
session-carrier machinery live in that package rather than in a
dedicated service. Nothing in the specifications requires them to
run as separate processes.

### 11.2 Per-spec adoption status

Status values: **implemented** (the shipped behaviour matches the
spec), **partially implemented** (a working subset exists, but a
material clause has no shipped counterpart or diverges), and
**proposed** (the spec describes a protocol no shipped component
speaks). Statuses and gaps come from the divergence catalogue and
the "protocol nothing speaks" findings of the architecture design
review; that review's synthesis document carries the underlying
evidence and the spec-clause references each row summarizes.

#### Intent stack

| Spec | Status | Largest gap |
|------|--------|-------------|
| OVOS-INTENT-1 | implemented | — |
| OVOS-INTENT-2 | partially implemented | Resource loaders still accept regex and digit-wildcard files that no resource role defines. |
| OVOS-INTENT-3 | partially implemented | `required_slots` has no producer, no engine support, and no skill-facing authoring API. |
| OVOS-INTENT-4 | partially implemented | Registration and deregistration consumers trust the payload's `skill_id` instead of the message context. A remote actor can deregister another skill's intents. |

#### Bus stack

| Spec | Status | Largest gap |
|------|--------|-------------|
| OVOS-MSG-1 | partially implemented | The reference Message accepts unknown top-level keys, applies no guard in `response()`, and still round-trips the multi-address `destination` the spec drops. |
| OVOS-SESSION-1 | partially implemented | The carrier-rejection duty (§2.5) has no shipped implementation. The session field set is not closed in practice: `is_speaking` gates stop routing while unclaimed by any spec. |
| OVOS-SESSION-2 | implemented | — |
| OVOS-BRIDGE-1 | partially implemented | Bridges route by session identifier rather than by destination, and relay poll-family broadcasts to participants that host no skill in the session. The spec forbids both. |

#### Orchestrator stack

| Spec | Status | Largest gap |
|------|--------|-------------|
| OVOS-PIPELINE-1 | partially implemented | The `Match` shape has no shipped counterpart: `updated_session`, the match-phase timeout, and the closed-for-good rule are all missing. |
| OVOS-TRANSFORM-1 | partially implemented | Transformer chains execute in descending priority order, the inverse of the spec's ascending convention, so an unrenumbered chain runs backwards. |
| OVOS-CONTEXT-1 | partially implemented | Cross-skill context still writes through a bus emission the spec retires in favor of a session-carried mutation. |
| OVOS-CONVERSE-1 | partially implemented | The converse plugin still speaks a skill-addressed ping/pong pair of its own rather than the reserved broadcast topics. |
| OVOS-STOP-1 | partially implemented | Only the `stop` intent name is rejected as reserved. Converse, response, fallback, and common_query remain indexable, and the stop round still addresses one ping per candidate instead of a single broadcast. |
| OVOS-PERSONA-1 | partially implemented | The activated and dismissed lifecycle events fire only for self-performed transitions, so a consumer cannot rely on them for every transition the spec describes. |
| OVOS-FALLBACK-1 | partially implemented | In-flight work moves toward a skill-addressed ping/pong pair, the opposite of the spec's single broadcast round. |
| OVOS-COMMON-QUERY-1 | proposed | The shipped plugin speaks a different topic pair, discovers responders at load time, and hides the answering skill on the match. None of that matches the spec's scatter-gather protocol. |

#### I/O stack

| Spec | Status | Largest gap |
|------|--------|-------------|
| OVOS-AUDIO-IN-1 | partially implemented | The language ladder (§5.1) has no implementation path. The listener does not hold the session fields the ladder reads. |
| OVOS-AUDIO-1 | partially implemented | The invented audio wire fields (§4.0) go unused. Every shipped component carries audio in different fields. |
| OVOS-GUI-1 | partially implemented | The shipped template catalogue and the spec's names diverge across the board, and an unrecognised template fails without a diagnostic. |

#### Media and scheduling stacks

| Spec | Status | Largest gap |
|------|--------|-------------|
| OVOS-OCP-1 | partially implemented | The status answer omits `next_track`, and nested playlists are flattened rather than preserved. |
| OVOS-SCHEDULER-1 | partially implemented | A bridge that rewrites a remote component's identity breaks the scheduler's identity comparison, which assumes the orchestrator's view of that identity is stable. |
