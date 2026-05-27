# Stop Pipeline Plugin Specification

**Spec ID:** OVOS-STOP-1 · **Version:** 1 · **Status:** Draft

This specification defines the **stop pipeline plugin** — a
pipeline plugin that matches utterances expressing the user's
intention to interrupt the assistant's current activity — and the
bus surface by which it cascades a stop request across the
recency-ordered list of active handlers and broadcasts a global
stop signal when no handler can absorb the request. The
intent_name `stop` is reserved at the OVOS-PIPELINE-1 §7.3
registry; no other plugin or skill may register it.

It builds on OVOS-MSG-1 (envelope, `forward` / `reply` /
`response`), OVOS-PIPELINE-1 (pipeline-plugin contract, dispatch
shape, lifecycle trio, reserved-name registry),
OVOS-SESSION-1 (session field registry — `active_handlers`,
`response_mode`), and OVOS-SESSION-2 (mutation boundaries and
session-keyed-state projection). `session.active_handlers` is
populated by OVOS-PIPELINE-1 §7.1's dispatch-time stamping rule
and drained by stop consumption defined here.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines the stop plugin role, the reserved
intent_name `stop`, the stoppability discovery and cascade
algorithm, the global broadcast namespace, and the session-scoping
obligations of stop subscribers.

It does **not** define vocabulary file formats, matching
algorithms, confidence thresholds, voice-activity detection,
microphone or audio capture control, handler-side framework
APIs, or wake-word and barge-in policies.

---

## 2. Reserved intent_name

This specification reserves a single intent_name at the
OVOS-PIPELINE-1 §7.3 registry:

| Reserved intent_name | Meaning of a Match bearing this name |
|----------------------|--------------------------------------|
| `stop` | A specific active handler should cease activity for the inbound `session_id`. Dispatched on `<target_skill_id>:stop` where the target is the LIFO head of positive pong responders (§4). |

Skills and other pipelines **MUST NOT** register `stop` under
OVOS-INTENT-4. A registration naming the reserved intent_name
is malformed per OVOS-INTENT-4 §5.3 — consumers log at WARN and
do not index. The reservation prevents competing skill-level
matches from bypassing the §4 cascade.

The stop plugin's escalation path uses the intent_name
`global_stop` for its own self-dispatch
(`<stop_plugin_id>:global_stop`, §5). This name is plugin-internal
and not reserved: a skill MAY register `global_stop` for its own
purposes, as the dispatch shape namespaces the topic under
`<stop_plugin_id>` and cannot collide with another skill's
handler.

---

## 3. The stop plugin role

The **stop plugin role** is a behavioural contract a pipeline
plugin (PIPELINE-1 §3) MAY adopt. A stop plugin is an ordinary
pipeline plugin — subject to the same denylist filtering,
first-match-wins iteration, and circuit-breaker rules — that
matches stop-command utterances and emits Matches under §2.

### 3.1 Pipeline identity

A stop plugin is loaded as one or more `pipeline_id` entries in
`session.pipeline`. The conventional three confidence tiers
`stop_high`, `stop_medium`, `stop_low` MAY be registered as
separate `pipeline_id`s or merged into one multi-tier plugin;
the choice is a deployment concern.

`Match.skill_id` MUST equal the target of the dispatch:

- for `intent_name: "stop"`, the LIFO head of positive pong
  responders (§4.2);
- for `intent_name: "global_stop"`, the stop plugin's own
  `pipeline_id`.

### 3.2 Match obligations

In addition to the general PIPELINE-1 §4 contract:

- A stop plugin MUST return `None` for any language for which it
  cannot resolve stop vocabulary.
- A stop plugin MUST read `session.active_handlers` to drive the
  cascade (§4).
- A stop plugin emitting the stoppability ping-pong (§4.2)
  performs that exchange **inside `match`**. This is a documented
  exception to the PIPELINE-1 §4.4 low-latency guidance,
  justified by the stop plugin's escape-hatch position at the
  head of the pipeline (§8). No other plugin is iterating during
  this exchange.

### 3.3 Vocabulary

A stop plugin SHOULD distinguish utterances expressing a generic
stop intention from utterances expressing an explicit
"stop everything" intention. Vocabulary file organisation is not
normative; only the resulting Match's `intent_name` is.

The "stop everything" vocabulary maps directly to
`intent_name: "global_stop"` without consulting
`active_handlers`. The generic vocabulary triggers the §4
cascade.

---

## 4. Generic stop — `intent_name: "stop"`

When the utterance matches generic stop vocabulary, the stop
plugin performs stoppability discovery against
`session.active_handlers` and either dispatches a directed stop
to a single handler or escalates to `global_stop`.

### 4.1 Algorithm

Inside `match`:

1. Read `session.active_handlers`. If empty, return a
   `global_stop` Match constructed per §5.
2. Emit a single `ovos.stop.ping` broadcast and collect responses
   on `ovos.stop.pong` up to a deployer-defined timeout
   (RECOMMENDED default: 0.5 seconds).
3. Identify positive responders — those whose `skill_id`
   appears in `session.active_handlers` and whose pong returned
   `can_handle: true` within the window. Pongs from skills not
   in `active_handlers` MUST be ignored; late pongs MAY be
   ignored.
4. If at least one positive responder exists, select the one
   whose position in `session.active_handlers` is most recent
   (LIFO head among positives). Construct `updated_session` by
   removing that `skill_id` from `active_handlers` and clearing
   any `response_mode` entry it owns. Return
   `Match(skill_id=<that_skill_id>, intent_name="stop",
   updated_session=...)`. PIPELINE-1 §7.1's dispatch-time push
   is suppressed for reserved-name dispatches (§7.3), so the
   removal carried in `updated_session` is the final state at
   dispatch.
5. If no positive responder exists, return a `global_stop` Match
   constructed per §5.

### 4.2 Ping and pong

`ovos.stop.ping` is a single broadcast addressing every active
handler at once; `ovos.stop.pong` is a single shared response
topic carrying the responder's `skill_id` in the payload. This
shape avoids any need for the stop plugin to enumerate per-skill
topics.

**Ping** — `ovos.stop.ping`:

Payload MAY be empty. The inbound `session_id` is carried by the
Message's session context (OVOS-MSG-1) and identifies the
session for which feasibility is being queried.

**Pong** — `ovos.stop.pong`:

```json
{ "skill_id": "music.skill", "can_handle": true }
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The `owner_id` of the responding handler. |
| `can_handle` | boolean | yes | Whether the handler can process a stop request *for the inbound `session_id`*. |

The ping-pong is a request/response pair under OVOS-MSG-1 §5.3.
A handler that does not respond within the timeout window is
treated as `can_handle: false`. Silence indicates only that the
handler did not opt in to the discovery protocol.

A handler MUST respond only when it has stoppable activity for
the inbound `session_id`. A handler that is active for one
`session_id` but cannot stop its activity for the inbound
`session_id` MUST either respond `can_handle: false` or remain
silent.

### 4.3 Dispatch and skill stop handler

The Match's dispatch follows the standard PIPELINE-1 §7 contract.
The orchestrator emits `<target_skill_id>:stop`; the target
skill's stop handler runs as an ordinary intent handler, firing
the standard handler-lifecycle trio
(`ovos.intent.handler.start`, `.complete`, `.error`).

The skill's stop handler MUST cease only the activity keyed to
the inbound `session_id`. A skill serving multiple sessions in
parallel MUST NOT interrupt activity belonging to a different
session.

### 4.4 Self-pruning of `active_handlers`

A handler that cannot be stopped — by design, by current state,
or for the relevant session — SHOULD remove itself from
`session.active_handlers` so that future ping rounds bypass it
entirely. The removal MAY be communicated by emitting any
session-carrying Message with the updated list; the orchestrator
and downstream consumers observe the next inbound session
without the entry. Self-pruning is the static complement to the
runtime ping-pong; together they keep the discovery cost
proportional to the number of genuinely stoppable handlers.

---

## 5. Global stop escalation

When a stop utterance cannot be cascaded to a specific handler,
the stop plugin escalates to a global broadcast. The escalation
path uses the intent_name `global_stop` as a plugin-internal
self-dispatch (not a reserved name; see §2).

A `global_stop` self-dispatch is emitted in three cases:

- explicit "stop everything" vocabulary match (§3.3);
- generic stop with empty `active_handlers` (§4.1 step 1);
- generic stop with no positive pong responders (§4.1 step 5).

### 5.1 Match construction

A `global_stop` Match MUST be:

`Match(skill_id=<own_pipeline_id>, intent_name="global_stop",
updated_session=...)`

where `updated_session` is the inbound session with:

- `active_handlers` emptied — global stop terminates every
  active handler, and the cleared list is the accurate
  post-stop state. Because the `global_stop` dispatch is a
  pipeline-plugin self-dispatch, PIPELINE-1 §7.1's stamping
  push is suppressed (§7.0) — `updated_session` is the final
  state at dispatch;
- `response_mode` removed entirely (§6.1).

### 5.2 Dispatch and broadcast

The orchestrator dispatches `<stop_plugin_id>:global_stop`.
Because `skill_id` equals the stop plugin's own `pipeline_id`,
the dispatch is uniquely routed to the stop handler regardless
of any skill that happens to use the same intent_name. The
stop handler emits the universal broadcast:

| Topic | Direction | Summary |
|-------|-----------|---------|
| `ovos.stop` | stop handler → all | Cease all activity. |

Payload MAY be empty. Every component performing user-visible
activity (audio playback, TTS, media, timers, animation, the
converse plugin's polling loop) MUST subscribe to `ovos.stop`
and cease its activity on receipt.

`ovos.stop` is not a dispatch — it does not follow the
`<skill_id>:<intent_name>` shape and does not fire the
handler-lifecycle trio. The trio fires for the `global_stop`
dispatch itself.

The namespace `ovos.stop.*` is reserved by this specification
for future stop-related signals.

### 5.3 Session scoping

A subscriber to `ovos.stop` MUST cease only the activity it has
running for the broadcast's inbound `session_id`. A TTS engine
speaking concurrently for two sessions stops only the session
that issued the stop; a media player playing for one session
does not interrupt another session's playback.

---

## 6. Session interaction

### 6.1 `response_mode`

If `session.response_mode` is present, a stop plugin SHOULD clear
it via `Match.updated_session` whenever it matches:

- for `intent_name: "stop"`, clear the entry whose `owner_id`
  matches the dispatch target;
- for `intent_name: "global_stop"`, remove `response_mode`
  entirely.

Per PIPELINE-1 §6.3, session mutations carried by
`Match.updated_session` are committed at dispatch time; the
clear is durable even if the handler crashes mid-execution. The
semantics and ownership of the field are defined elsewhere; this
spec only requires that a stop operation cancels any wait the
stopped target was holding.

### 6.2 `active_handlers`

`session.active_handlers` is populated by PIPELINE-1 §7.1's
dispatch-time stamping rule on every ordinary dispatch. The
stamping push is suppressed for reserved intent_names (§7.3) —
`stop`, `converse`, `response` — so reserved-name dispatches
never add to the list.

A stop plugin MUST drain the list via `Match.updated_session`,
which is committed pre-dispatch (PIPELINE-1 §4.2):

- a `stop` Match MUST remove the dispatch target entry only,
  leaving the rest of the list intact;
- a `global_stop` Match MUST empty `active_handlers` entirely
  (§5.1).

Pre-dispatch mutation is the cleanest defined boundary for
session state changes: the orchestrator commits
`updated_session`, suppresses the push (because the dispatch
is on a reserved name or self-addressed to the stop plugin),
and the downstream lifecycle sees the post-stop list. No
consumer-side removal protocol is needed; no race between an
optimistic pre-removal and a failed handler exists because
removal and dispatch happen atomically at the same boundary.

### 6.3 Denylists

A stop plugin MUST honour `session.blacklisted_skills` and
`session.blacklisted_intents` (PIPELINE-1 §5.3–§5.4). A handler
whose `owner_id` appears in `blacklisted_skills` MUST NOT be
pinged or selected as a stop target.

---

## 7. Pipeline positioning

A deployment that includes the stop plugin SHOULD place the
highest-confidence stop stage **first** in `session.pipeline` —
ahead of the converse plugin and every intent-matching stage.
This positioning is the user-facing escape hatch from any
in-flight conversational state and is also what makes the
in-match ping-pong of §3.2 latency-safe: no other plugin is
iterating during the discovery window.

Lower-confidence stop stages MAY be interleaved with
intent-matching stages so that ambiguous stop-like utterances do
not pre-empt ordinary intents. A typical ordering:

```
session.pipeline: [
  "stop_high",            # escape hatch
  "converse",
  "intent_matcher_high",
  "stop_medium",
  "intent_matcher_medium",
  "stop_low",
  "intent_matcher_low"
]
```

---

## 8. Bus surface summary

| Topic | Direction | Purpose | Defined in |
|-------|-----------|---------|------------|
| `ovos.stop.ping` | stop plugin → all | Stoppability ping (broadcast) | §4.2 |
| `ovos.stop.pong` | skill → stop plugin | Stoppability response (shared) | §4.2 |
| `<target_skill_id>:stop` | orchestrator → target skill | Skill-directed stop dispatch | §4.3 |
| `<stop_plugin_id>:global_stop` | orchestrator → stop handler | Global stop dispatch | §5.2 |
| `ovos.stop` | stop handler → all | Universal stop broadcast | §5.2 |

Dispatch topics fire the handler-lifecycle trio per PIPELINE-1
§7; no other topic in this table does.

---

## 9. Conformance

### A stop pipeline plugin **MUST**:

- match per PIPELINE-1 §4, returning either `intent_name: "stop"`
  or `intent_name: "global_stop"` and never any other value;
- set `Match.skill_id` per §3.1 — the dispatch target for `stop`,
  the plugin's own `pipeline_id` for `global_stop`;
- return `None` when no stop vocabulary matches or when no
  vocabulary is available for the requested `lang`;
- read `session.active_handlers` to drive the cascade (§4.1);
- communicate session mutations exclusively through
  `Match.updated_session`;
- on a `stop` Match, remove the dispatch target from
  `session.active_handlers` via `Match.updated_session`; on a
  `global_stop` Match, empty `active_handlers` entirely (§6.2);
- honour `session.blacklisted_skills` and
  `session.blacklisted_intents` (§6.3);
- subscribe to `<own_pipeline_id>:global_stop` and emit `ovos.stop`
  from that handler.

### A stop pipeline plugin **SHOULD**:

- clear `session.response_mode` via `Match.updated_session` (§6.1);
- skip the ping-pong and return `global_stop` immediately when
  `active_handlers` is empty.

### A deployment that includes a stop plugin **SHOULD**:

- place the highest-confidence stop stage first in
  `session.pipeline` (§7);
- configure stop vocabulary for every supported language.

### A skill that participates in stop **MUST**:

- subscribe to **both** `<own_skill_id>:stop` and `ovos.stop`.
  `<own_skill_id>:stop` carries the skill-directed cascade
  dispatch (§4.3); `ovos.stop` carries the global broadcast
  (§5.2). The two subscriptions are not alternatives — a skill
  receives one or the other depending on the cascade outcome,
  and must be ready for either;
- on receiving `<own_skill_id>:stop`, cease only the activity
  keyed to the inbound `session_id`;
- on receiving `ovos.stop`, cease all activity keyed to the
  inbound `session_id`;
- treat duplicate stop dispatches and `ovos.stop` broadcasts as
  idempotent.

### A skill that participates in stop **SHOULD**:

- subscribe to `ovos.stop.ping` and reply on `ovos.stop.pong`
  with its own `skill_id` and `can_handle` reflecting feasibility
  *for the inbound `session_id`* — or remain silent if it has no
  stoppable activity for that session;
- remove itself from `session.active_handlers` when it cannot
  be stopped, so that future ping rounds bypass it (§4.4).

### Every non-skill component performing user-visible activity **MUST**:

- subscribe to `ovos.stop` and cease activity keyed to the
  inbound `session_id` on receipt.

### The orchestrator **MUST**:

- treat OVOS-INTENT-4 registrations naming `stop` as malformed
  per INTENT-4 §5.3 — log at WARN and decline to index.
  Registrations naming `global_stop` are accepted normally;
  `global_stop` is not reserved.

---

## See also

- *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
- *Active Handlers and Interactive Response Specification* (OVOS-CONVERSE-1)
- *Bus Message Specification* (OVOS-MSG-1)
- *Session Carrier Wire Shape Specification* (OVOS-SESSION-1)
- *Session Lifecycle and State Ownership Specification* (OVOS-SESSION-2)
