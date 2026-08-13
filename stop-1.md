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
| `stop` | Cease activity for the inbound `session_id`. Dispatched on `<target_skill_id>:stop` where the target is the most recently activated positive pong responder, or — when no positive pong arrives in time — the most recently activated remaining `active_handlers` entry (§4.1). |

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

A stop plugin instance has **one** `pipeline_id`, in the same
namespace as a `skill_id` and indistinguishable from one
(OVOS-PIPELINE-1 §3). A deployment MAY reference that plugin from
several `session.pipeline` entries — separate confidence tiers, for
example — but an entry is a reference to a match configuration, not
an actor: every such entry resolves to the same plugin instance and
the same `pipeline_id`, and the tiers have no identities of their
own.

Because the tiers are one actor, the obligation is trivially
satisfiable and stays a **MUST**: a stop plugin instance **MUST**
emit exactly one `ovos.stop` broadcast per global stop event per
session, however many entries reference it.

`Match.skill_id` MUST equal:

- for `intent_name: "stop"` — the target selected per §4.1 (the most
  recently activated positive pong responder, or the recency fallback
  of step 5);
- for `intent_name: "global_stop"` — the stop plugin's own
  `pipeline_id` (§5.2).

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

1. Read `session.active_handlers`. If it is empty, or if its only
   entries name the stop plugin's own `pipeline_id` (§5.2), return a
   `global_stop` Match per §5.
2. Emit `ovos.stop.ping`, derived via `reply` from the inbound
   utterance Message (OVOS-MSG-1 §5.2), and collect `ovos.stop.pong`
   responses within a deployer-configured timeout (RECOMMENDED
   default: 0.5 s; SHOULD NOT exceed 1 s).
3. Identify positive responders: valid pongs (§4.2) where
   `can_handle` is `true` and `skill_id` appears in
   `session.active_handlers`. Pongs from skills not in
   `active_handlers` MUST be ignored.
4. If at least one positive responder exists, form the candidate set
   from the `active_handlers` entries whose `skill_id` is a positive
   responder, apply the candidate filter and the recency rule below,
   and construct `updated_session` removing the selected `skill_id`
   from `active_handlers` and clearing any `response_mode` entry it
   owns. Return
   `Match(skill_id=<that_skill_id>, intent_name="stop", updated_session=...)`.
   Selection MUST be restricted to positive responders: an entry that
   did not answer the ping positively MUST NOT be selected at this
   step, however recent it is.
5. If no positive responder exists but `active_handlers` is non-empty,
   the stop plugin MUST fall back to recency: form the candidate set
   from every `active_handlers` entry, apply the candidate filter and
   the recency rule below, and return a `Match` constructed exactly as
   in step 4. The plugin MUST NOT escalate to `global_stop` when no
   pong arrives. `global_stop` remains reserved for explicit
   global-stop vocabulary (§3.2) and the empty-`active_handlers` case
   (step 1).

**Candidate filter.** Before any recency comparison, an
`active_handlers` entry MUST be skipped when:

- its `skill_id` appears in `session.blacklisted_skills` (§6.3); or
- its `skill_id` equals the stop plugin's own `pipeline_id` — the
  entry PIPELINE-1 §7.1 stamps for a preceding `global_stop` dispatch
  (§5.2). The stop plugin is never its own stop target.

If no candidate survives the filter at step 4 or step 5, `match` MUST
return `None`; the orchestrator continues iteration to the next
pipeline stage.

**Recency rule.** Among the surviving candidates, select the entry
with the highest `activated_at`. If two entries share the same
`activated_at`, select the entry nearest the head of `active_handlers`
— the most recently pushed. PIPELINE-1 §7.1 is the normative home of
both the push mechanics and this tie-break; this section only applies
them.

Exactly one skill is stopped per stop utterance. `match` returns a
single `Match` naming a single target, and the orchestrator dispatches
`stop` to that target only.

### 4.2 Ping and pong shape

**`ovos.stop.ping`** — broadcast. Payload MAY be empty. The stop
plugin MUST derive the ping via `reply` from the inbound utterance
Message (OVOS-MSG-1 §5.2), so that the ping carries the inbound
`session_id` and the routing metadata of the utterance emitter.

**`ovos.stop.pong`** — shared reply topic. A handler MUST emit a
Message of type `ovos.stop.pong` derived via `reply` from the ping
(OVOS-MSG-1 §5.2), so that the pong reaches the stop plugin
regardless of where the skill is running (local or remote). `source`
and `destination` are layer-2 metadata and do not affect the topic
name.

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

A handler with no current activity for the inbound `session_id`
**MUST NOT** respond `can_handle: true`. It MAY respond
`can_handle: false` or remain silent. A handler that does not
subscribe to `ovos.stop.ping`, or does not respond within the timeout,
is treated as `can_handle: false` for that ping round. If no handler
declares stoppability, the cascade falls back to the most recently
activated remaining `active_handlers` entry per §4.1 step 5 — it does
not escalate to `global_stop`.

**Malformed and duplicate pongs.** A pong is *valid* only when it
carries a `skill_id` string and a `can_handle` boolean. The stop
plugin MUST treat the responding handler as not stoppable for that
ping round when:

- `can_handle` is absent, or is present but not a JSON boolean —
  a truthy non-boolean value MUST NOT be coerced to `true`;
- `skill_id` is absent, or does not match the `skill_id` of the
  handler that emitted the Message as identified by the MSG-1
  derivation metadata.

When a handler emits more than one pong in a ping round, the first
valid pong wins and later pongs from the same `skill_id` MUST be
ignored. Pongs arriving after the timeout MUST be ignored: the
selection made at step 4 or step 5 is final for that utterance.

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

A `global_stop` Match is returned in two cases:

- explicit "stop everything" vocabulary (§3.2);
- generic stop with an `active_handlers` list that is empty or holds
  only the stop plugin's own entry (§4.1 step 1).

A generic stop with no positive pong responders does **not** trigger
`global_stop`; it falls back to the recency-selected target (§4.1
step 5).

### 5.2 Match construction

The `global_stop` Match MUST carry a fully-cleaned `updated_session`:

```
Match(
  skill_id   = <the stop plugin's pipeline_id>,
  intent_name = "global_stop",
  updated_session = <session with:
    active_handlers  → []
    converse_handlers → []
    response_mode    → absent>
)
```

All three fields are cleared atomically at match time via
`Match.updated_session` (PIPELINE-1 §4.2), before dispatch.

`Match.skill_id` here is the `pipeline_id` of the stop plugin
instance whose `match` produced this Match — the identity the
`global_stop` dispatch topic addresses and PIPELINE-1 §7.1 stamps.
There is no second or shared identifier: a plugin has one
`pipeline_id` whatever the number of `session.pipeline` entries that
reference it (§3.1, PIPELINE-1 §3), so "the stop plugin's
`pipeline_id`" is unambiguous everywhere it appears.

`global_stop` is not a reserved intent_name, so PIPELINE-1 §7.1
stamping suppression does not apply: the orchestrator pushes the
stop plugin's `pipeline_id` onto `active_handlers` after committing
this `updated_session`. The committed post-dispatch state is therefore not
an empty list — `active_handlers` holds exactly one entry, the stop
plugin's own. §4.1 excludes that entry from stop candidacy and §4.1
step 1 treats such a list as empty, so a following generic stop
resolves to `global_stop` again rather than to the stop plugin
itself.

### 5.3 Broadcast

The handler dispatched by `<pipeline_id>:global_stop` MUST emit
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
state. It is not suppressed for `global_stop`: the committed state
after a `global_stop` dispatch is `active_handlers ==
[<the stop plugin's own entry>]`, not `[]` (§5.2). `converse_handlers`
carries no such stamp and stays empty.

### 6.3 Denylists

A stop plugin MUST honour `session.blacklisted_skills` and
`session.blacklisted_intents` (PIPELINE-1 §5):

- `blacklisted_skills`: a handler whose `skill_id` appears in this list
  MUST NOT be selected as a stop target. The plugin skips such entries
  in the §4.1 candidate filter, before any recency comparison;
- `blacklisted_intents`: entries are qualified
  `<skill_id>:<intent_name>` pairs (PIPELINE-1 §5.4); a bare
  intent_name never matches. A stop plugin MUST NOT return a `Match`
  whose `<Match.skill_id>:<Match.intent_name>` appears in
  `blacklisted_intents`. A `stop` utterance that would resolve to
  `global_stop` (§4.1 step 1) is subject to the
  `<pipeline_id>:global_stop` entry, not to a `stop` entry.
  This list does not affect the ping broadcast.

Both rules are plugin-side obligations that duplicate, at match time,
the filter PIPELINE-1 §5.4 places on the orchestrator. The
strengthening is intentional: it keeps a blacklisted target out of the
recency selection instead of discarding the whole Match after the fact.

---

## 7. Pipeline positioning

A deployment that includes the stop plugin SHOULD place the
highest-confidence stop stage **first** in `session.pipeline`, ahead
of the converse plugin and every intent-matching stage. Lower-confidence
stop stages MAY be interleaved with intent-matching stages.

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
- derive `ovos.stop.ping` via `reply` from the inbound utterance Message (§4.2);
- collect pong responses within a deployer-configured timeout (§4.1);
- ignore pongs from skills absent from `session.active_handlers` (§4.1);
- ignore malformed, duplicate and late pongs, and treat the responder as not stoppable (§4.2);
- treat non-responding handlers as `can_handle: false` (§4.2);
- select the stop target among positive pong responders only, when any exist (§4.1 step 4);
- break `activated_at` ties in favour of the entry nearest the head of `active_handlers` (§4.1, PIPELINE-1 §7.1);
- exclude its own `pipeline_id` and blacklisted skills from the candidate set before the recency comparison, and return `None` when no candidate survives (§4.1);
- stop exactly one skill per stop utterance (§4.1);
- clear `session.response_mode` for the dispatch target via `Match.updated_session` (§6.1);
- drain `active_handlers` via `Match.updated_session` (§6.2);
- on `global_stop`, also empty `converse_handlers` via `Match.updated_session` (§6.2);
- return `global_stop` only for explicit global-stop vocabulary or an `active_handlers` list that is empty or holds only its own entry (§3.2, §4.1 step 1);
- with no positive pong responder and non-empty `active_handlers`, target the highest-`activated_at` surviving candidate with `intent_name: "stop"` rather than escalating (§4.1 step 5);
- honour `session.blacklisted_skills` and `session.blacklisted_intents` per §6.3;
- subscribe to `<own_pipeline_id>:global_stop` and emit `ovos.stop` (§5.3);
- emit exactly one `ovos.stop` broadcast per global stop event per session, however many `session.pipeline` entries reference it (§3.1).

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
