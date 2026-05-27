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
active_handlers stamping), OVOS-SESSION-1 (`active_handlers` and
`response_mode` field registry), OVOS-SESSION-2 (mutation boundaries).

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
APIs, or wake-word and barge-in policies.

---

## 2. Reserved intent_name

| Reserved intent_name | Meaning |
|----------------------|---------|
| `stop` | Cease activity for the inbound `session_id`. Dispatched on `<target_skill_id>:stop` where the target is the most recently activated (highest `activated_at`) positive pong responder (§4). |

Skills and other pipelines **MUST NOT** register `stop` under
OVOS-INTENT-4. A registration naming this intent_name is malformed per
OVOS-INTENT-4 §5.3 — consumers log at WARN and do not index.

The intent_name `global_stop` is **not** reserved. The stop plugin
uses it for its own self-dispatch (`<stop_plugin_id>:global_stop`,
§5), namespaced under its own `pipeline_id`.

---

## 3. The stop plugin role

A stop plugin is an ordinary pipeline plugin (PIPELINE-1 §3) that
matches stop-command utterances and returns Matches under §2. It is
subject to the same denylist filtering, first-match-wins iteration,
and circuit-breaker rules as every other pipeline plugin.

### 3.1 Pipeline identity and dispatch target

A stop plugin is loaded as one or more `pipeline_id` entries in
`session.pipeline`. `Match.skill_id` MUST equal:

- for `intent_name: "stop"` — the most recently activated positive
  pong responder selected per §4;
- for `intent_name: "global_stop"` — the stop plugin's own `pipeline_id`.

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
2. Emit `ovos.stop.ping` and collect `ovos.stop.pong` responses up to
   a deployer-defined timeout (RECOMMENDED default: 0.5 s).
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
| `can_handle` | boolean | yes | Whether the handler can stop for the inbound `session_id`. |

A handler that does not respond within the timeout is treated as
`can_handle: false`. A handler MUST respond `can_handle: true` only
when it has stoppable activity for the inbound `session_id`.

### 4.3 Dispatch

The orchestrator dispatches `<target_skill_id>:stop` per the standard
PIPELINE-1 §7 contract. The target's stop handler fires the
handler-lifecycle trio (`ovos.intent.handler.start`, `.complete`,
`.error`).

The handler MUST cease only the activity keyed to the inbound
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

```
Match(
  skill_id=<own_pipeline_id>,
  intent_name="global_stop",
  updated_session=<session with active_handlers emptied
                  and response_mode removed>
)
```

PIPELINE-1 §7.1 stamps `<stop_plugin_id>` onto `active_handlers` at
dispatch time (the name `global_stop` is not reserved, so suppression
does not apply).

### 5.3 Broadcast

The stop handler dispatched by `<stop_plugin_id>:global_stop` MUST
emit `ovos.stop` — a universal broadcast. Every component performing
user-visible activity MUST subscribe to `ovos.stop` and cease activity
for the broadcast's `session_id`.

`ovos.stop` is not a dispatch topic — it does not follow the
`<skill_id>:<intent_name>` shape and does not fire the handler-lifecycle
trio. The namespace `ovos.stop.*` is reserved by this specification.

---

## 6. Session interaction

### 6.1 `response_mode`

A stop plugin SHOULD clear `session.response_mode` via
`Match.updated_session`:

- for `intent_name: "stop"` — clear the entry whose `owner_id` matches
  the dispatch target;
- for `intent_name: "global_stop"` — remove `response_mode` entirely.

### 6.2 `active_handlers`

A stop plugin MUST drain `active_handlers` via `Match.updated_session`
(committed pre-dispatch per PIPELINE-1 §4.2):

- `stop` Match — remove the dispatch target entry only;
- `global_stop` Match — empty `active_handlers` entirely.

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

- return `Match` with `intent_name` of exactly `"stop"` or
  `"global_stop"`, never any other value;
- set `Match.skill_id` per §3.1;
- return `None` when no stop vocabulary matches or the requested
  `lang` is unsupported;
- read `session.active_handlers` to drive the cascade (§4.1);
- communicate all session mutations through `Match.updated_session`;
- remove the dispatch target from `active_handlers` on a `stop` Match;
  empty `active_handlers` entirely on a `global_stop` Match (§6.2);
- honour `session.blacklisted_skills` and `session.blacklisted_intents` (§6.3);
- subscribe to `<own_pipeline_id>:global_stop` and emit `ovos.stop`
  from that handler.

### Stop pipeline plugin — SHOULD:

- clear `session.response_mode` via `Match.updated_session` (§6.1);
- return `global_stop` immediately when `active_handlers` is empty,
  without emitting a ping.

### Deployment — SHOULD:

- place the highest-confidence stop stage first in `session.pipeline` (§7);
- configure stop vocabulary for every supported language.

### Skill — MUST:

- subscribe to both `<own_skill_id>:stop` and `ovos.stop`;
- on `<own_skill_id>:stop`, cease activity for the inbound `session_id`;
- on `ovos.stop`, cease all activity for the inbound `session_id`;
- treat duplicate stop dispatches and `ovos.stop` broadcasts as
  idempotent.

### Skill — SHOULD:

- subscribe to `ovos.stop.ping` and reply on `ovos.stop.pong` with
  `skill_id` and `can_handle` reflecting feasibility for the inbound
  `session_id`, or remain silent if it has no stoppable activity;
- remove itself from `session.active_handlers` when it cannot be
  stopped (§4.4).

### Non-skill component performing user-visible activity — MUST:

- subscribe to `ovos.stop` and cease activity for the inbound
  `session_id` on receipt.

### Orchestrator — MUST:

- treat OVOS-INTENT-4 registrations naming `stop` as malformed —
  log at WARN and decline to index.

---

## See also

- OVOS-PIPELINE-1 — pipeline contract, dispatch, active_handlers
- OVOS-SESSION-1 — session field registry
- OVOS-SESSION-2 — mutation boundaries
- OVOS-MSG-1 — Message envelope and derivations
