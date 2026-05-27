# Stop Pipeline Plugin Specification

**Spec ID:** OVOS-STOP-1 · **Version:** 1 · **Status:** Draft

This specification defines the **stop pipeline plugin** — a pipeline
plugin that matches utterances expressing the user's intention to
interrupt the assistant's current activity — and the bus surface by
which it cascades a stop request across the recency-ordered list of
active handlers or broadcasts a global stop signal.

The intent_name `stop` is reserved at OVOS-PIPELINE-1 §7.3.

Dependencies: OVOS-MSG-1 (envelope and derivations), OVOS-PIPELINE-1
(pipeline-plugin contract, dispatch shape, reserved-name registry,
`active_handlers` stamping), OVOS-SESSION-1 (session field registry),
OVOS-SESSION-2 (mutation boundaries), OVOS-CONVERSE-1 (`response_mode`
and `converse_handlers` field definitions).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**
are used as in RFC 2119.

---

## 1. Scope

This specification defines: the stop plugin role, the reserved
intent_name `stop`, the stoppability discovery and cascade algorithm,
the global broadcast, and the session-scoping obligations of stop
subscribers.

It does **not** define: vocabulary file formats, matching algorithms,
confidence thresholds, audio capture control, handler-side framework
APIs, wake-word and barge-in policies, or post-stop in-flight
interaction teardown (a skill-side or orchestrator-side concern).

---

## 2. Reserved intent_name

| Reserved intent_name | Meaning |
|----------------------|---------|
| `stop` | Cease activity for the inbound `session_id`. Dispatched on `<target_skill_id>:stop` where the target is the most recently activated (highest `activated_at`) positive pong responder (§4). |

Skills and other pipelines **MUST NOT** register `stop` under
OVOS-INTENT-4. A registration naming this intent_name is malformed per
OVOS-INTENT-4 §5.3 — consumers log at WARN and do not index.

The intent_name `global_stop` is **not** reserved. The stop plugin
uses it for its own self-dispatch (`<stop_plugin_id>:global_stop`, §5),
namespaced under its own `pipeline_id`.

---

## 3. The stop plugin role

A stop plugin is an ordinary pipeline plugin (PIPELINE-1 §3) that
matches stop-command utterances and returns Matches under §2. It is
subject to the same denylist filtering, first-match-wins iteration, and
circuit-breaker rules as every other pipeline plugin.

### 3.1 Pipeline identity and dispatch target

A stop plugin is loaded as one or more `pipeline_id` entries in
`session.pipeline`. Multiple confidence tiers (`stop_high`,
`stop_medium`, `stop_low`) MAY be registered as separate `pipeline_id`s
or merged into one plugin. When registered as separate entries, all
tiers MUST share a single `global_stop` dispatch handler — identified
by a common `pipeline_id` — so that exactly one `ovos.stop` broadcast
is emitted per global stop event.

`Match.skill_id` MUST equal:

- for `intent_name: "stop"` — the most recently activated positive pong
  responder selected per §4;
- for `intent_name: "global_stop"` — the shared `pipeline_id` whose
  handler emits `ovos.stop`.

### 3.2 Match obligations

A stop plugin **MUST**:

- return `None` for any language for which it cannot resolve stop
  vocabulary;
- read `session.active_handlers` to drive the §4 cascade;
- perform the ping-pong exchange (§4.2) inside `match`.

### 3.3 Vocabulary

A stop plugin SHOULD distinguish generic stop utterances from explicit
"stop everything" utterances. Generic stop triggers the §4 cascade;
"stop everything" maps directly to `intent_name: "global_stop"` without
consulting `active_handlers`.

---

## 4. Generic stop — `intent_name: "stop"`

### 4.1 Algorithm

Inside `match`:

1. Read `session.active_handlers`. If empty, return a `global_stop`
   Match per §5.
2. Emit `ovos.stop.ping` and collect `ovos.stop.pong` responses within
   a deployer-configured timeout (RECOMMENDED default: 0.5 s;
   SHOULD NOT exceed 1 s).
3. Identify positive responders: pongs where `can_handle: true` and
   `skill_id` appears in `session.active_handlers`. Pongs from skills
   not in `active_handlers` MUST be ignored; late pongs MAY be ignored.
4. If at least one positive responder exists, select the entry with the
   highest `activated_at` in `session.active_handlers`. Construct
   `updated_session` removing that `skill_id` from `active_handlers`
   and clearing any `response_mode` entry it owns. Return
   `Match(skill_id=<that_skill_id>, intent_name="stop", updated_session=...)`.
5. If no positive responder exists, return a `global_stop` Match per §5.

### 4.2 Ping and pong shape

**`ovos.stop.ping`** — broadcast. Payload MAY be empty. The inbound
`session_id` is carried in Message context (OVOS-MSG-1).

**`ovos.stop.pong`** — shared reply topic.

```json
{ "skill_id": "example.skill", "can_handle": true }
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The `skill_id` of the responding handler. |
| `can_handle` | boolean | yes | Whether the handler has stoppable activity for the inbound `session_id`. |

`can_handle: true` asserts that the handler has user-visible or
session-affecting activity in progress for the inbound `session_id`
**and** is prepared to cease it on receipt of `<skill_id>:stop`.
A handler with no current activity MUST respond `can_handle: false`
or remain silent. A handler that does not respond within the timeout
is treated as `can_handle: false`.

### 4.3 Dispatch and stop handler obligations

The orchestrator dispatches `<target_skill_id>:stop` per PIPELINE-1 §7,
firing the handler-lifecycle trio (`ovos.intent.handler.start`,
`.complete`, `.error`).

The stop handler MUST:

- cease the activity it declared stoppable, scoped to the inbound
  `session_id`;
- treat a second `<skill_id>:stop` or `ovos.stop` arriving while
  already stopping as idempotent.

The stop handler MUST NOT interrupt activity belonging to a different
`session_id`.

### 4.4 Self-pruning of `active_handlers`

A handler that cannot be stopped SHOULD remove itself from
`session.active_handlers` by emitting any session-carrying Message
with the updated list, so that future ping rounds bypass it.

---

## 5. Global stop — `intent_name: "global_stop"`

### 5.1 Trigger conditions

A `global_stop` Match is returned in three cases:

- explicit "stop everything" vocabulary (§3.3);
- generic stop with empty `active_handlers` (§4.1 step 1);
- generic stop with no positive pong responders (§4.1 step 5).

### 5.2 Match construction

The `global_stop` Match MUST carry a fully-cleaned `updated_session`:

```
Match(
  skill_id   = <shared_pipeline_id>,
  intent_name = "global_stop",
  updated_session = <session with:
    active_handlers  → []
    converse_handlers → []
    response_mode    → absent>
)
```

All three fields are cleared atomically at match time via
`Match.updated_session` (PIPELINE-1 §4.2), before dispatch. The stop
plugin owns this cleanup entirely — no downstream component needs to
inspect or drain these fields as a consequence of a global stop.

PIPELINE-1 §7.1 stamps `<shared_pipeline_id>` onto `active_handlers`
at dispatch time (the name `global_stop` is not reserved, so stamping
suppression does not apply). This is intentional: the stop plugin MAY
participate in converse after a global stop — for example, to handle a
follow-up clarification such as "not the cooking timer" — provided it
registers a converse handler.

### 5.3 Broadcast

The handler dispatched by `<shared_pipeline_id>:global_stop` MUST emit
`ovos.stop`. Every component performing user-visible activity MUST
subscribe to `ovos.stop` and cease activity for the broadcast's
`session_id`.

`ovos.stop` is not a dispatch topic — it does not follow the
`<skill_id>:<intent_name>` shape and does not fire the handler-lifecycle
trio. The namespace `ovos.stop.*` is reserved by this specification.

---

## 6. Session interaction

### 6.1 `response_mode`

For `intent_name: "stop"`, a stop plugin MUST clear the
`session.response_mode` entry whose `owner_id` matches the dispatch
target, via `Match.updated_session`. (For `intent_name: "global_stop"`,
`response_mode` is removed entirely as part of the §5.2 Match
construction.)

An uncleared `response_mode` for a stopped skill would route the next
utterance to that skill as if it were still awaiting a response.

### 6.2 `active_handlers`

`session.active_handlers` (OVOS-PIPELINE-1 §7.1) is the stop
cascade's recency input. It is distinct from `session.converse_handlers`
(OVOS-CONVERSE-1 §2.1), the converse plugin's eligibility list.
Draining `active_handlers` on stop does **not** affect
`converse_handlers` — a stopped skill remains eligible for converse
turns until its TTL expires or it self-removes.

A stop plugin MUST drain `active_handlers` via `Match.updated_session`
(committed pre-dispatch per PIPELINE-1 §4.2):

- `stop` Match — remove the dispatch target entry only;
- `global_stop` Match — empty `active_handlers` entirely and empty
  `converse_handlers` (OVOS-CONVERSE-1 §2.1) entirely. A global stop
  ends all active engagement; leaving skills in the converse eligibility
  list would allow the converse plugin to continue polling them until
  their TTL expires.

The stamping push (PIPELINE-1 §7.1) is suppressed for the reserved
intent_name `stop`, so the removal in `updated_session` is the final
state. It is not suppressed for `global_stop`.

### 6.3 Denylists

A stop plugin MUST honour `session.blacklisted_skills` and
`session.blacklisted_intents` (PIPELINE-1 §5). A handler whose
`skill_id` appears in `blacklisted_skills` MUST NOT be pinged or
selected as a stop target.

---

## 7. Pipeline positioning

A deployment that includes the stop plugin SHOULD place the
highest-confidence stop stage **first** in `session.pipeline`, ahead
of the converse plugin and every intent-matching stage. Lower-confidence
stop stages MAY be interleaved with intent-matching stages.

Typical ordering:

```
session.pipeline: [
  "stop_high",
  "converse",
  "intent_high",
  "stop_medium",
  "intent_medium",
  "stop_low",
  "intent_low"
]
```

---

## 8. Bus surface

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.stop.ping` | stop plugin → all | Stoppability query (broadcast) |
| `ovos.stop.pong` | skill → stop plugin | Stoppability response |
| `<target_skill_id>:stop` | orchestrator → skill | Skill-directed stop dispatch |
| `<stop_plugin_id>:global_stop` | orchestrator → stop handler | Global stop dispatch |
| `ovos.stop` | stop handler → all | Universal stop broadcast |

Dispatch topics (`<…>:stop`, `<…>:global_stop`) fire the
handler-lifecycle trio. No other topic in this table does.

---

## 9. Conformance

### Stop pipeline plugin — MUST:

- return `intent_name` of exactly `"stop"` or `"global_stop"` (§2, §3.1);
- set `Match.skill_id` per §3.1;
- return `None` when no stop vocabulary matches or `lang` is unsupported;
- collect pong responses within a deployer-configured timeout (§4.1);
- ignore pongs from skills absent from `session.active_handlers` (§4.1);
- clear `session.response_mode` via `Match.updated_session` (§6.1);
- drain `active_handlers` via `Match.updated_session` (§6.2);
- on `global_stop`, also empty `converse_handlers` via `Match.updated_session` (§6.2);
- return `global_stop` when `active_handlers` is empty or no positive pong responder exists (§4.1 steps 1, 5);
- honour `session.blacklisted_skills` and `session.blacklisted_intents` (§6.3);
- subscribe to `<own_pipeline_id>:global_stop` and emit `ovos.stop` (§5.3).

### Stop pipeline plugin — SHOULD:

- configure the ping-pong timeout to not exceed 1 s (§4.1);
- skip the ping and return `global_stop` immediately when `active_handlers` is empty (§4.1 step 1);
- place all confidence tiers under a shared `pipeline_id` for `global_stop` (§3.1).

### Deployment — SHOULD:

- place the highest-confidence stop stage first in `session.pipeline` (§7);
- configure stop vocabulary for every supported language.

### Skill — MUST:

- subscribe to both `<own_skill_id>:stop` and `ovos.stop`;
- on `<own_skill_id>:stop`, cease stoppable activity for the inbound `session_id` (§4.3);
- on `ovos.stop`, cease all activity for the inbound `session_id`;
- treat duplicate stop dispatches and `ovos.stop` broadcasts as idempotent.

### Skill — SHOULD:

- subscribe to `ovos.stop.ping` and reply on `ovos.stop.pong` with
  `can_handle` reflecting stoppable activity for the inbound `session_id` (§4.2);
- remove itself from `session.active_handlers` when it cannot be stopped (§4.4).

### Non-skill component performing user-visible activity — MUST:

- subscribe to `ovos.stop` and cease activity for the inbound `session_id`.

### Orchestrator — MUST:

- treat OVOS-INTENT-4 registrations naming `stop` as malformed —
  log at WARN and decline to index (§2).

---

## See also

- OVOS-PIPELINE-1 — pipeline contract, dispatch, active_handlers
- OVOS-SESSION-1 — session field registry
- OVOS-SESSION-2 — mutation boundaries
- OVOS-MSG-1 — Message envelope and derivations
