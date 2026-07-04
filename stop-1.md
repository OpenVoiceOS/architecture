# Stop Pipeline Plugin Specification

**Spec ID:** OVOS-STOP-1 · **Version:** 2 · **Status:** Draft

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

A stop plugin MAY be loaded as multiple `pipeline_id` entries in
`session.pipeline` (e.g. separate confidence tiers). When multiple
entries are deployed, the implementation MUST ensure exactly one
`ovos.stop` broadcast is emitted per global stop event per session.

`Match.skill_id` MUST equal:

- for `intent_name: "stop"` — the most recently activated positive pong
  responder selected per §4;
- for `intent_name: "global_stop"` — the `pipeline_id` whose handler
  emits `ovos.stop`.

A stop plugin MUST:

- return `None` for any language for which it cannot resolve stop
  vocabulary;
- read `session.active_handlers` to drive the §4 cascade;
- perform the ping-pong exchange (§4.2) inside `match`.

### 3.2 Vocabulary

A stop plugin SHOULD provide explicit "stop everything" vocabulary that
maps directly to `intent_name: "global_stop"` without running the §4
cascade. Generic stop utterances run the cascade per §4.

---

## 4. Generic stop — `intent_name: "stop"`

### 4.1 Algorithm

Inside `match`:

1. Read `session.active_handlers`. If empty, return a `global_stop`
   Match per §5.
2. Emit `ovos.stop.ping` and collect `ovos.stop.pong` responses within
   a deployer-configured timeout (default: 0.5 s; SHOULD NOT exceed 1 s).
3. Identify positive responders: pongs where `can_handle: true` and
   `skill_id` appears in `session.active_handlers`. Pongs from skills
   not in `active_handlers` MUST be ignored; late pongs MAY be ignored.
4. If at least one positive responder exists, select the entry with the
   highest `activated_at` in `session.active_handlers`. If two entries
   share the same `activated_at`, select the entry appearing latest in
   the list (most recently stamped). Construct
   `updated_session` removing that `skill_id` from `active_handlers`
   and clearing any `response_mode` entry it owns. Return
   `Match(skill_id=<that_skill_id>, intent_name="stop", updated_session=...)`.
5. If no positive responder exists, return a `global_stop` Match per §5.

### 4.2 Ping and pong shape

**`ovos.stop.ping`** — broadcast. Payload MAY be empty. `session_id`
is carried in Message context per OVOS-MSG-1.

**`ovos.stop.pong`** — shared reply topic. A handler MUST emit a
Message of type `ovos.stop.pong` derived via `reply` (OVOS-MSG-1 §5),
so that routing metadata is preserved and the pong reaches the stop
plugin regardless of where the skill is running (local or remote).
`source` and `destination` are layer-2 metadata and do not affect the
topic name.

```json
{ "skill_id": "example.skill", "can_handle": true }
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The `skill_id` of the responding handler. |
| `can_handle` | boolean | yes | Whether the handler has stoppable activity for the inbound `session_id`. |

The boolean's field name is protocol-specific: this spec and
OVOS-FALLBACK-1 use `can_handle`, OVOS-CONVERSE-1's poll uses
`result`, and OVOS-COMMON-QUERY-1 uses `can_answer`. Each name is
normative only within its own protocol.

`can_handle: true` asserts that the handler has user-visible or
session-affecting activity in progress for the inbound `session_id`
**and** is prepared to cease it on receipt of `<skill_id>:stop`.

A handler with no current activity MUST respond `can_handle: false`
or remain silent. A handler that does not subscribe to
`ovos.stop.ping`, or does not respond within the timeout, is treated
as `can_handle: false` and is ineligible as a stop target for that
ping round. If no handler declares stoppability, the cascade escalates
to `global_stop` per §4.1 step 5.

### 4.3 Dispatch and stop handler obligations

The orchestrator dispatches `<target_skill_id>:stop` per PIPELINE-1 §7,
firing the handler-lifecycle trio (`ovos.intent.handler.start`,
`.complete`, `.error`).

The stop handler MUST:

- cease the activity it declared stoppable, scoped to the inbound
  `session_id`;
- not initiate a second stop sequence if a stop dispatch arrives while
  already stopping — the duplicate MUST be treated as a no-op.

The stop handler MUST NOT interrupt activity belonging to a different
`session_id`.

`Match.updated_session` is committed before dispatch (PIPELINE-1 §4.2)
and is not rolled back if the stop handler emits the `.error`
lifecycle event.

### 4.4 Self-pruning of `active_handlers`

A handler that cannot be stopped SHOULD remove itself from
`session.active_handlers` by emitting any session-carrying Message
with the updated list, so that future ping rounds bypass it.

---

## 5. Global stop — `intent_name: "global_stop"`

### 5.1 Trigger conditions

A `global_stop` Match is returned in three cases:

- explicit "stop everything" vocabulary (§3.2);
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
`Match.updated_session` (PIPELINE-1 §4.2), before dispatch.

`<shared_pipeline_id>` is the `pipeline_id` of the stop plugin
instance whose `match` produced this Match. When multiple stop
instances are deployed (§3.1), they MUST deduplicate so that
exactly one `ovos.stop` broadcast is emitted per global stop event
per session.

PIPELINE-1 §7.1 stamps `<shared_pipeline_id>` onto `active_handlers`
at dispatch time (the name `global_stop` is not reserved, so stamping
suppression does not apply). This is intentional: the stop plugin MAY
participate in converse after a global stop — for example, to handle a
follow-up clarification — provided it registers a converse handler.

### 5.3 Broadcast

The handler dispatched by `<shared_pipeline_id>:global_stop` MUST emit
`ovos.stop`. Every component performing user-visible activity MUST
subscribe to `ovos.stop` and cease activity for the `session_id`
carried in Message context per OVOS-MSG-1.

`ovos.stop` is not a dispatch topic — it does not follow the
`<skill_id>:<intent_name>` shape and does not fire the handler-lifecycle
trio. The namespace `ovos.stop.*` is reserved by this specification.

---

## 6. Session interaction

### 6.1 `response_mode`

For `intent_name: "stop"`, a stop plugin MUST clear the
`session.response_mode` entry whose `skill_id` matches the dispatch
target, via `Match.updated_session`. If no such entry exists, the
field is left unchanged. For `intent_name: "global_stop"`,
`response_mode` is removed entirely as part of the §5.2 Match
construction.

### 6.2 `active_handlers`

`session.active_handlers` (OVOS-PIPELINE-1 §7.1) is the stop
cascade's recency input. It is distinct from `session.converse_handlers`
(OVOS-CONVERSE-1 §2.1), the converse plugin's eligibility list.

A stop plugin MUST drain `active_handlers` via `Match.updated_session`
(committed pre-dispatch per PIPELINE-1 §4.2):

- `stop` Match — remove the dispatch target entry only;
- `global_stop` Match — empty `active_handlers` entirely and empty
  `converse_handlers` (OVOS-CONVERSE-1 §2.1) entirely.

The stamping push (PIPELINE-1 §7.1) is suppressed for the reserved
intent_name `stop`, so the removal in `updated_session` is the final
state. It is not suppressed for `global_stop`.

### 6.3 Denylists

A stop plugin MUST honour `session.blacklisted_skills` and
`session.blacklisted_intents` (PIPELINE-1 §5):

- `blacklisted_skills`: a handler whose `skill_id` appears in this list
  MUST NOT be selected as a stop target;
- `blacklisted_intents`: applies to the dispatched intent_name (`"stop"`
  or `"global_stop"`). A stop plugin MUST NOT return a `Match` whose
  `<Match.skill_id>:<Match.intent_name>` appears in
  `blacklisted_intents`. A `stop` utterance that
  would resolve to `global_stop` (§4.1 steps 1 or 5) is subject to the
  `global_stop` entry, not the `stop` entry. This list does not affect
  the ping broadcast.

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
- return `None` when no stop vocabulary matches or `lang` is unsupported (§3.1);
- collect pong responses within a deployer-configured timeout (§4.1);
- ignore pongs from skills absent from `session.active_handlers` (§4.1);
- treat non-responding handlers as `can_handle: false` (§4.2);
- clear `session.response_mode` for the dispatch target via `Match.updated_session` (§6.1);
- drain `active_handlers` via `Match.updated_session` (§6.2);
- on `global_stop`, also empty `converse_handlers` via `Match.updated_session` (§6.2);
- return `global_stop` when `active_handlers` is empty or no positive pong responder exists (§4.1 steps 1, 5);
- honour `session.blacklisted_skills` and `session.blacklisted_intents` per §6.3;
- subscribe to `<own_pipeline_id>:global_stop` and emit `ovos.stop` (§5.3);
- emit exactly one `ovos.stop` broadcast per global stop event per session (§3.1).

### Stop pipeline plugin — SHOULD:

- configure the ping-pong timeout to not exceed 1 s (§4.1);
- provide explicit "stop everything" vocabulary mapping to `global_stop` without cascade (§3.2).

### Deployment — SHOULD:

- place the highest-confidence stop stage first in `session.pipeline` (§7);
- configure stop vocabulary for every supported language.

### Skill — MUST:

- subscribe to both `<own_skill_id>:stop` and `ovos.stop`;
- on `<own_skill_id>:stop`, cease stoppable activity for the inbound `session_id` (§4.3);
- on `ovos.stop`, cease all activity for the inbound `session_id`;
- treat a duplicate `<own_skill_id>:stop` or `ovos.stop` as a no-op (§4.3).

### Skill — SHOULD:

- subscribe to `ovos.stop.ping` and respond with a `reply`-derived
  `ovos.stop.pong` carrying `can_handle` for the inbound `session_id` (§4.2);
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
