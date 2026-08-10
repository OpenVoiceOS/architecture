# Active Handlers and Interactive Response Specification

**Spec ID:** OVOS-CONVERSE-1 · **Version:** 2 · **Status:** Draft

This document defines the **imperative continuous-dialog** surface
of the assistant: the session-scoped **converse-handler list** of
intent owners that recently engaged the user, the **converse**
mechanism by which a converse handler may claim an incoming
utterance before normal intent matching, and the **interactive
response** mechanism by which a handler may suspend intent matching
to collect the next utterance directly. Together these three
mechanisms carry every multi-turn flow whose continuity is held by
the handler itself rather than by a declarative gate on an intent.

It is the imperative complement to the *Intent Context
Specification* (OVOS-CONTEXT-1), whose `requires_context` and
`excludes_context` gates declaratively bias intent matching across
turns. CONTEXT-1 entries decay by themselves; the surfaces defined
here are driven by explicit handler action and explicit user
input. The two surfaces are orthogonal; §7 fixes their evaluation
order.

The **converse plugin** (§4) is a fully ordinary PIPELINE-1 §3
pipeline plugin. It returns a `Match` on the reserved intent_name
`converse` (or `response` for response-mode delivery), and the
orchestrator dispatches that Match identically to any other — on
`<skill_id>:<intent_name>`, with the full handler-lifecycle trio.
The two reserved intent_names are leased at PIPELINE-1 §7.3; no
skill may register them.

The response-mode mechanism (§5) lives entirely in
**session-resident state**. A handler enters response mode by
mutating `session.response_mode` from within its handler; the
mutation rides forward on subsequent emissions and lands in the
client's session store. No side-band bus event is required.

It builds on six companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  `Message.context`, the `session` carrier, and the
  `forward` / `reply` / `response` derivations every Message
  defined here travels in;
- the *Session Specification* (OVOS-SESSION-1) —
  the field-registry mechanism under which this spec claims its
  session fields (§2), and the omission-not-`null` rule;
- the *Session Lifecycle and State Ownership Specification*
  (OVOS-SESSION-2) — the SHOULD-project pathway (§2.4 there)
  that this spec's converse plugin role embraces fully for its
  response-mode wait state, plus the in-utterance mutation
  boundaries (§2.6 there) and the default-session ownership rule
  (§5 there);
- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the pipeline-plugin contract the converse
  plugin role conforms to (§4), `Match.updated_session` (§4.2),
  dispatch topic shape and handler-owner polymorphism (§7),
  handler-lifecycle trio, universal end-marker
  `ovos.utterance.handled`, reserved-intent-name registry (§7.3),
  and blacklist policy fields (§5.3–§5.4);
- the *Transformer Plugins Specification* (OVOS-TRANSFORM-1) —
  the six lifecycle hooks at which a transformer MAY mutate the
  session fields owned by this spec (§3.3);
- the *Intent Context Specification* (OVOS-CONTEXT-1) — the
  declarative gating primitive this spec is orthogonal to (§7).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines the converse-handler list
(`session.converse_handlers`, §2.1), the response-mode field
(`session.response_mode`, §2.2), the activation lifecycle (§3),
the converse plugin role (§4), the interactive response
mechanism (§5), the two reserved intent_names `converse` and
`response`, the bus surface (§6), the evaluation order relative
to CONTEXT-1 (§7), termination (§8), and conformance (§9).
Non-goals are listed in §10.

---

## 2. Session fields

This specification claims two session fields under SESSION-1 §2.2's
field registry. Both fields obey SESSION-1 §2.1's omission rule:
they are present-with-a-value or absent, never JSON `null`.

### 2.1 `session.converse_handlers`

`session.converse_handlers` is this specification's **converse
eligibility list** — the set of intent owners the converse plugin
(§4) will poll. It is distinct from `session.active_handlers`
(OVOS-PIPELINE-1 §7.1), which is the dispatch-recency record used by
the stop cascade. The two lists are written and drained independently:
PIPELINE-1 stamps `active_handlers` at dispatch time and the stop
plugin drains it on stop; this spec stamps `converse_handlers` at
dispatch time and the TTL prune (§3.2) decays it. A skill removed from
`active_handlers` by a **targeted stop** (STOP-1 §4) remains in
`converse_handlers` and may still be offered converse turns.

`session.converse_handlers` is a JSON array of `{skill_id,
activated_at}` objects. `skill_id` is a plain skill's id or a
pipeline plugin's `pipeline_id` (per PIPELINE-1 §7.0).
`activated_at` is a Unix-seconds wall-clock timestamp (float
precision). Because `skill_id` appears in colon-separated topic
shapes it **MUST NOT** contain `:` (MSG-1 §2.1.1); the recommended
form is ASCII letters / digits / `_` / `-` only.

**Identifier charset (stated once, for the whole spec).** A `.` in
`skill_id` is permitted. Every dotted topic this spec defines
(`<skill_id>.converse.ping` / `.pong`, §4.2) is built from a known
`skill_id` and delivered by **exact subscription** — never parsed
back into components — so it stays unambiguous even when `skill_id`
itself contains `.` (MSG-1 §2.1.1). Only the `:` prohibition is
normative. §2.2 and §4.2 rely on this paragraph and do not restate
it.

The list is ordered **head-first by recency**: index `0` is the
most recently activated owner; index `n-1` is the least recently
activated of the surviving owners. An omitted or absent
`session.converse_handlers` is equivalent to `[]`.

Each `skill_id` appears in the list **at most once**. The list is
a *recency stack with deduplication*, not an append-only log: a
re-activation of an already-listed owner removes the prior entry
and re-inserts at the head (§3.1).

Deployments **SHOULD** bound the list length. The
**RECOMMENDED default maximum is 10 entries**, which a deployer
**MAY** raise, lower, or set to "unbounded". The cap is a tuning
value, not an interoperability invariant — any bound (or none)
is conformant. The default is deliberately small: every
listed owner is a candidate for the per-utterance poll (§4.2), and
the poll sits on the serial critical path of every utterance — a
large eligibility list converts directly into user-perceived
latency, while genuine multi-turn engagement rarely involves more
than a handful of recently active owners. When the cap would be
exceeded by an
insertion, the orchestrator **MUST** drop the tail entry (the
least-recent surviving owner) before inserting the new head, and
**SHOULD** log the eviction.

**Response-mode holder exemption.** The owner named by a present,
non-expired `session.response_mode` (§2.2) **MUST NOT** be evicted
by the cap, nor pruned by the §3.2 TTL. Evicting it would strand
the pending response window: the §2.2 identity invariant would then
reject the holder's own answer and the utterance the holder
solicited would be matched by some other stage. When the cap would
be exceeded and the tail entry is the exempt holder, the
orchestrator **MUST** drop the next-least-recent non-exempt entry
instead. The exemption ends the moment `response_mode` is absent or
its `expires_at` has passed.

### 2.2 `session.response_mode`

`session.response_mode` is a structured object describing the
pending response window for the session, or **absent** when no
holder is awaiting a direct response:

| Key | Type | Required | Meaning |
|-----|------|----------|---------|
| `skill_id` | string | yes | The handler that holds response mode for this session. Charset per §2.1. |
| `expires_at` | number | yes | Unix-seconds wall-clock time after which the wait window is stale. MAY be a float; consumers MUST accept integer and float forms. Once passed, the entry is **inert** — no delivery ever happens from it (§5.2) — and it is discarded opportunistically per §5.2. The holder's framework-side timer drives the user-facing reaction. |

Absent or omitted `session.response_mode` means no holder; the
next utterance is matched normally. JSON `null` is **not** a valid
value (SESSION-1 §2.1).

The field is **session-resident state** — it rides every Message
that carries the session forward. No bus event is required to set,
clear, or modify it.

Response mode is **single-shot**: each delivery clears the field
(§5.2). A handler that wants the next utterance re-enters from
within its `:response` handler (§5.1).

**Identity invariant.** A handler MUST set `skill_id` to its own
identity. The converse plugin MUST ignore (log and discard) any
`response_mode` entry whose `skill_id` does not appear in
`session.converse_handlers` — an unrecognised owner could route
an utterance to an unintended handler.

**Single-holder invariant.** At most one holder per session. A
handler that sets `response_mode` while another entry is present
overwrites it silently; the previous holder's framework-side timer
eventually fires.

---

## 3. Activation lifecycle

Four mutation pathways apply to `session.converse_handlers`:

- **Automatic activation on dispatch** (§3.1) — orchestrator-side,
  on every `<skill_id>:<intent_name>` dispatch. The dominant path.
- **Match-phase mutation via `Match.updated_session`**
  (PIPELINE-1 §4.2) — the converse plugin uses this to
  pre-promote a poll-winner atomically with its claim.
- **Transformer-driven mutation** (§3.3) — any transformer MAY
  mutate the field; metadata transformers are the recommended
  hook.
- **Handler-side mutation** — a dispatched handler MAY mutate
  the field in-place (SESSION-2 §2.6); the change rides forward
  on the handler's emissions.

Optional TTL pruning is described in §3.2.

### 3.1 Automatic activation on dispatch

The orchestrator MUST stamp `session.converse_handlers` whenever it
dispatches to an owner via the OVOS-PIPELINE-1 §7 dispatch topic
`<skill_id>:<intent_name>`. This rule is uniform — it applies to
every dispatch including reserved intent_names `converse` (§4.3) and
`response` (§5.2). Stamping `converse_handlers` on reserved-name
dispatches is intentional: a skill that just handled a converse or
response turn remains eligible for the next poll. This differs from
OVOS-PIPELINE-1 §7.1's suppression of `active_handlers` stamps for
reserved names — the two lists have independent stamping rules.

Activation updates `session.converse_handlers` as follows:

1. Remove any existing entry whose `skill_id` matches the activating handler.
2. Insert `{skill_id: <activating handler's skill_id>, activated_at: <current Unix time>}` at index `0`.
3. If the resulting length exceeds the §2.1 cap, drop the tail
   entry.

The mutation MUST be applied to the `session` snapshot carried by
the dispatch Message (and by every Message subsequently derived
from it via `forward` / `reply` / `response`). Activation is
idempotent for an already-head owner (re-promotion to head).

A converse plugin's per-owner **poll** round-trip (§4.2) is **not
a dispatch** and does **not** cause activation. Polled owners
that decline are not added to the converse-handler list as a side
effect of being polled.

### 3.2 Optional TTL pruning

A deployment **SHOULD** configure a recency-list **time-to-live**
`T` seconds. Without TTL, entries age out only when the §2.1 size
cap evicts the tail — which never triggers if the converse-handler
count stays below the cap. A stale entry would be polled on every
utterance indefinitely.

Because each entry carries `activated_at`, TTL tracking is
**session-resident**: the orchestrator reads `activated_at` from the
inbound session and prunes any entry whose age (`now - activated_at`)
exceeds `T` at two boundaries:

1. **Pre-converse** — immediately before a converse plugin (§4)
   begins its poll iteration for the current utterance.
2. **Pre-list-emission** — immediately before answering the
   `ovos.converse.active.list` introspection request (§6.1).

The orchestrator MAY run the prune at additional boundaries; doing
so MUST NOT produce observably different behaviour from running it
only at the two boundaries above. The **RECOMMENDED default TTL is
600 seconds**; an implementation SHOULD apply it when the deployer
has not configured a value. Absent a TTL the list ages only by the
§2.1 size cap — which never triggers while the owner count stays
below the cap, so a stale owner would be polled on every utterance
forever. A deployer MAY explicitly configure "no TTL", accepting
that behaviour.

The prune **MUST NOT** remove the owner named by a present,
non-expired `session.response_mode` (the §2.1 exemption).

**Additional decay triggers.** Two optional triggers shorten the
dead-skill poll tax:

- An orchestrator or converse plugin **MAY** prune an entry whose
  owner has failed to answer `N` consecutive polls with
  `error_code: "timeout"` (§4.4). `N` is deployer-configured; the
  RECOMMENDED value is 3. The prune is applied through the same
  session-mutation channels as any other list change — the
  orchestrator at a §3.2 boundary, the plugin via
  `Match.updated_session`.
- The orchestrator **SHOULD** remove an owner's entry from
  `session.converse_handlers` on `ovos.skill.deregister`
  (OVOS-INTENT-4 §8.4) for that `skill_id`, scoped to the
  `session_id` the deregistration carries. An unloaded skill
  cannot answer a poll, and INTENT-4 §8.4 makes the removal a
  no-op when no entry exists.

Because `activated_at` is session-resident, TTL pruning is
**resumption-safe**: a session re-sent after an orchestrator restart
carries its own timestamps and is pruned correctly on the first
inbound utterance.

### 3.3 Transformer-driven mutation

A transformer (OVOS-TRANSFORM-1) operates at one of six fixed
lifecycle hooks. **Any** transformer **MAY** mutate
`session.converse_handlers` and `session.response_mode` in the
normal course of its work — the session carrier is part of the
mutable state TRANSFORM-1 permits transformers to operate on.

A transformer that mutates these fields **SHOULD** be a
**metadata transformer** (OVOS-TRANSFORM-1 §3.3). Metadata
transformers run **after** the utterance-transformer chain and
**before** pipeline iteration begins, which is the natural point
to assert or revoke conversational state for the current
utterance: the utterance text is final, no plugin has yet seen
it, and the converse plugin (§4) and intent stages will observe
the post-mutation state on the same iteration.

Mutations at other hooks are **permitted** and have well-defined
effects on the lifecycle, but the timing is less natural:

| Hook | Mutation timing relative to this spec's surfaces |
|------|-------------------------------------------------|
| Audio transformer (pre-STT) | Runs before the entry topic; mutations land before §3.1, §3.2, §3.3 metadata phase, and §4 iteration — earliest possible. |
| Utterance transformer (post-STT) | Runs before metadata; mutations land before pipeline iteration and converse, like metadata, but mixed with utterance-text mutation. |
| **Metadata transformer (post-utterance)** | **SHOULD** be the chosen hook. Mutations land between utterance-finalization and pipeline iteration; the converse plugin and intent stages observe them on the same utterance. |
| Intent transformer (post-match, pre-dispatch) | Mutations land after a converse plugin has already iterated for this utterance; the new state takes effect only from the *next* utterance. |
| Dialog transformer (post-skill, pre-TTS) | Mutations land after the dispatch trio has completed. Same "next utterance only" timing. |
| TTS transformer (post-TTS, pre-playback) | Mutations land after audio rendering. Effect is identical to a dialog-transformer mutation in this spec's terms. |

A transformer is a trusted in-process component (TRANSFORM-1 §6)
and may mutate any owner's entry on the session it operates on.
The §2.1 invariants MUST still hold after the mutation; the
orchestrator MAY normalize a non-conformant mutation into a
compliant list.

A transformer that mutates `session.response_mode` directly
affects subsequent response-mode dispatch decisions per §5.2 —
removing the field cancels the pending response; replacing it
with a different holder transfers the wait to that holder.

---

## 4. The converse plugin role

The **converse plugin role** is a behavioural contract that a
pipeline plugin (OVOS-PIPELINE-1 §3) MAY adopt. A pipeline plugin
that adopts the role examines `session.response_mode` and
`session.converse_handlers` during its `match` operation, polls
each eligible converse handler via the §4.2 round-trip, populates
`Match.updated_session` for any session mutations it needs
(clearing `session.response_mode` on delivery, pre-promoting
a poll-winner), and returns a `Match` on the reserved intent_name
`converse` when an owner claims (or `response` when response-mode
delivery applies, per §5.2).

A converse plugin is a pipeline plugin **in every respect, with
no exceptions**. Its `match` MAY emit bus Messages (the poll
round-trip), MAY mutate session via `Match.updated_session` (the
PIPELINE-1 §4.2 mechanism), and its returned `Match` dispatches
per PIPELINE-1 §7 normally. There is no dispatch suppression and
no out-of-band signalling between the plugin and the orchestrator.

The plugin is a **pure matcher** — a matcher whose only output is
its return value: its `match` produces a `Match` whose `skill_id`
is some *other* component's identity (the claiming handler, or the
response-mode holder), never its own `pipeline_id`
(OVOS-PIPELINE-1 §7.0), and the plugin bundles no handler of its
own. The reserved
intent_names are dispatched to those handler-owners, which
subscribe to `<own_skill_id>:converse` / `<own_skill_id>:response`.
The plugin **SHOULD** publish the intent_names it produces matches
on (`converse`, `response`) to the orchestrator's per-pipeline
passive index `ovos.pipeline.<pipeline_id>.intents.list` per
PIPELINE-1 §10, so observers can enumerate the reserved-name
surfaces this plugin handles.

A deployment **SHOULD** load **at most one** converse plugin.
Loading zero is a valid choice: `session.converse_handlers` is
still maintained per §3, but no owner is offered the chance to
claim utterances. The behaviour of a deployment that loads
**several** converse plugins is **unspecified by this
specification** — each would poll the same eligible set on the
same utterance, duplicating the round-trip and racing for the
claim, and the resulting ordering is a property of
`session.pipeline` rather than of this contract.

A deployment that wants response-mode delivery (§5) **SHOULD**
configure a converse plugin at the **front** of `session.pipeline`
(typically index `0`). PIPELINE-1's first-match-wins iteration
(§6.2 there) then ensures that response-mode delivery and converse
claims pre-empt other pipeline stages naturally, without any
"skip the rest of the pipeline" rule on the orchestrator side.

### 4.1 Iteration

When invoked, a converse plugin MUST proceed in this order:

1. **Response-mode pre-emption.** If `session.response_mode` is
   present and its `expires_at` is in the future, the plugin
   MUST first verify the §2.2 identity invariant: after the §3.2
   TTL prune (when configured), the holder's `skill_id` MUST
   appear in `session.converse_handlers` — a pruned or
   unrecognised holder MUST NOT receive delivery; the plugin
   logs it, discards it opportunistically per §5.2, and
   proceeds to step 2. For a valid holder, the
   plugin MUST return a `Match` per §5.2 for the holder, on
   `intent_name == "response"`, and SKIP the converse-handler poll.
   The orchestrator dispatches per PIPELINE-1 §7; the holder's
   response handler runs.

   If `session.response_mode` is present but `expires_at` has
   passed, the plugin MUST treat it as stale and proceed to
   step 2 — no response-mode delivery happens, for this
   utterance or any later one. Discarding the field is
   opportunistic, per §5.2.

2. **Converse-handler iteration.** The plugin reads
   `session.converse_handlers` (after the §3.2 TTL prune, when
   configured) as its eligible set. Blacklist policy is applied
   by the PIPELINE-1 §5.3–§5.4 backstop, not by this plugin.

3. **Poll iteration and selection.** The plugin polls the
   eligible set via §4.2 and selects the **first claimer in
   recency order** — the earliest entry in
   `session.converse_handlers`, which is ordered head-first by
   recency (§2.1). Where two entries carry an equal
   `activated_at`, the entry **nearest the head** is the more
   recent (OVOS-PIPELINE-1 §7.1). Selection is **never** by
   response-arrival order.

   The plugin SHOULD issue poll requests in parallel — the poll
   runs on the serial critical path of every utterance, and
   sequential per-owner waits multiply the per-owner timeout by
   the list length.

   **Sequential and parallel forms are equivalent.** Polling the
   list head-first and stopping at the first `result: true`
   yields the same claimer as polling every owner concurrently
   and then selecting the first `true` in list order, because
   selection is keyed on list position and silence or a
   malformed pong counts as `result: false` (§4.2) in both
   forms. A deployment MAY choose either; the parallel form only
   overlaps the waits instead of accumulating them.

   The plugin SHOULD skip any owner whose `skill_id` appears in
   `session.blacklisted_skills` — doing so avoids an
   unnecessary round-trip before the PIPELINE-1 §5.3
   backstop would reject the resulting Match anyway.

4. **Match.** If a claimer is found, the plugin MUST return a
   `Match` on `intent_name == "converse"` per §4.3. If no
   eligible owner claims, the plugin's `match` MUST return
   `null` and the orchestrator proceeds to the next stage in
   `session.pipeline` per PIPELINE-1 §6.

### 4.2 The poll round-trip

For each entry in the eligible set, the converse plugin
asks "do you want to claim this utterance?" via an addressed
bus round-trip. The round-trip is **not** a PIPELINE-1 §7
dispatch — it does not use the `<skill_id>:<intent_name>` topic
shape, it does not fire the handler-lifecycle trio, and it does
not activate the polled owner.

The plugin MUST emit a **poll request** on the topic
`<skill_id>.converse.ping`, where the leading component is
the owner being asked. The plugin **MUST** derive the ping via
`reply` (OVOS-MSG-1 §5.2) from the **inbound utterance Message**,
so that `context.session` and the routing keys propagate
automatically and the ping reaches the owner whether it runs
locally or behind a satellite transport. The Message:

- carries the full inbound session snapshot;
- carries `data.skill_id` equal to the topic prefix (the owner
  being polled);
- carries `data.utterances` (the candidate list per PIPELINE-1
  §4.1) and `data.lang` (the active language).

The owner MUST emit a Message of type `<skill_id>.converse.pong`
derived via `reply` (OVOS-MSG-1 §5.2), so that routing metadata is
preserved and the response reaches the converse plugin regardless
of whether the skill runs locally or remotely (e.g. via a satellite
transport). `source` and `destination` are layer-2 metadata and do
not affect the topic name. The response carries `data`:

| Key | Type | Required | Meaning |
|-----|------|----------|---------|
| `skill_id` | string | yes | The owner answering; MUST equal the topic prefix and the polled `skill_id`. |
| `result` | boolean | yes | `true` ⇒ the owner claims the utterance; `false` ⇒ the owner declines. |
| `error_code` | string | no | Optional structured reason (see §4.4) when `result` is `false`. |

The boolean's field name is protocol-specific: this spec's poll
uses `result`, while the analogous polls of OVOS-FALLBACK-1 /
OVOS-STOP-1 use `can_handle` and OVOS-COMMON-QUERY-1 uses
`can_answer`. Each name is normative only within its own protocol.

Both topics use the **dotted addressed** form (`<skill_id>.<verb>`).
The poll is **not** a PIPELINE-1 §7 dispatch, so it avoids the `:`
separator that OVOS-MSG-1 §2.1.1 reserves for the dispatch shape
`<skill_id>:<intent_name>` and uses `.` instead. The owner subscribes
to its own `<own_skill_id>.converse.ping` and replies on
`<own_skill_id>.converse.pong`; the plugin emits the identical strings.
Exact-subscription delivery keeps these topics unambiguous for any
conformant `skill_id`; see §2.1.

The poll is the **handler's decision point**. The owner MUST
inspect `data.utterances` and `data.lang` — including any NLU
or intent-parsing it needs — and commit to a claim decision
before replying. `result: true` means the handler has decided
it will handle the utterance and MUST do so fully when
`<skill_id>:converse` is dispatched (§4.3). `result: false`
means the handler declines; it MUST NOT perform any user-facing
work (`ovos.utterance.speak`, set context, etc.) in response to the poll.
An owner that wants to be removed from `session.converse_handlers`
SHOULD include `error_code: "done"` in a declining response
(§4.4) — the poll response is the designated channel for
self-deactivation requests.

A converse plugin MUST wait for the poll reply with a
**deployer-configured per-owner timeout** — an unbounded wait
would stall the utterance's serial critical path indefinitely.
The **RECOMMENDED default timeout is `0.5` seconds**, which a
deployer MAY raise or lower. An owner
that does not respond within the timeout is treated as
`result: false`.

**Aggregate poll ceiling.** The per-owner timeout alone does not
bound the stage: a sequential cycle over `n` owners costs up to
`n ×` the per-owner timeout. A converse plugin **MUST** therefore
also bound the **whole poll iteration** by an aggregate ceiling.
The ceiling is **deployment-configured** — it depends on the
§2.1 list cap and on whether the plugin polls sequentially or in
parallel — and when it elapses every owner that has not yet
answered is treated as `result: false`, selection proceeding over
the pongs already in hand per §4.1 step 3. Because the converse
plugin sits at the **front** of `session.pipeline` (§4), a
deployment **MUST** set any OVOS-PIPELINE-1 §4.4 match-phase
bound for this stage at or above its aggregate ceiling; a
match-phase bound shorter than the ceiling kills the poll
mid-collection on every utterance the stage handles.

**Malformed and foreign pongs.** A pong is valid only when its
`skill_id` equals the polled owner and the topic prefix, and it
arrives within the timeout for the current `(owner, utterance)`
round-trip. A converse plugin **MUST** ignore a pong whose
`skill_id` is mismatched, that names an owner not in the current
eligible set, or that arrives after the timeout for its round-trip
has elapsed. A pong with a missing or non-boolean `result` **MUST**
be treated as `result: false`. Where an owner answers more than
once, the **first valid pong per owner** wins and later ones are
ignored. A converse plugin MUST NOT use the reply from owner *i*
for utterance *u* to satisfy a different utterance *u′* or a
different owner *j*; the round-trip is per-`(owner, utterance)`.

**Bus-exchange exception.** The poll round-trip is a documented
exception to OVOS-PIPELINE-1 §4.4's low-latency guidance. It is
justified because the poll **is** the handler's decision point:
the claim cannot be resolved without asking the owners, and no
cheaper signal stands in for the answer. The exchange is bounded
twice over — per-owner timeout and aggregate ceiling — and
terminates as soon as the eligible set is exhausted. OVOS-FALLBACK-1
§6.1 documents the same exception for its own per-skill query.


### 4.3 The match for a converse claim

When some owner replies `result: true`, the converse plugin
returns a `Match` (PIPELINE-1 §4.1) shaped as follows:

- `skill_id` = the claiming owner;
- `intent_name` = the reserved value `converse`;
- `lang` = the active language;
- `utterance` = the chosen candidate;
- `slots` = `{}` (a converse claim is not a slot-bearing
  match);
- `updated_session` = OPTIONAL. The converse plugin MAY use
  this PIPELINE-1 §4.2 channel to pre-promote the claimer to
  the head of `session.converse_handlers`, or perform any other
  session mutation it wants downstream consumers to see. When
  the plugin omits `updated_session`, the orchestrator's §3.1
  automatic-activation rule applies on dispatch and produces
  observably equivalent state.

The orchestrator dispatches `<skill_id>:converse` per
PIPELINE-1 §7 — identically to any other intent dispatch, with
the full handler-lifecycle trio and end-marker per §8.

**Semantic contract.** By the time `:converse` is dispatched the
handler has already committed — it inspected the utterance in the
poll and returned `result: true`. The handler MUST handle it
fully; all user-facing work (`ovos.utterance.speak`, set context,
etc.) happens here, not in the poll.

### 4.4 Error codes

Where a structured reason is emitted in a
`<skill_id>.converse.pong` payload's `error_code` field,
the value SHOULD be drawn from:

| Code | Meaning |
|------|---------|
| `timeout` | **Plugin-synthesised only.** The owner did not respond within the per-owner timeout or the aggregate ceiling (§4.2). It never appears on the wire — by definition no pong arrived — and an owner MUST NOT emit it. It exists so plugin logs and metrics can name the silence. |
| `not_eligible` | The owner is no longer present on the converse-handler list (raced removal). |
| `handler_error` | The owner attempted to decide but an internal error prevented it. |
| `killed` | The poll was terminated by an interrupt signal (see §5.4) before the owner could decide. |
| `done` | The owner explicitly signals it is finished with this conversation thread and requests removal from `session.converse_handlers`. |

When a plugin receives `error_code: "done"` it removes the
declining owner from `session.converse_handlers`:

- **When returning a `Match`** (another owner claimed on the same
  iteration): remove the `"done"` owner via `Match.updated_session`
  (the PIPELINE-1 §4.2 channel).
- **When returning `null`** (no owner claimed): there is no Match
  to carry `updated_session`, and in-place mutation of the inbound
  session object is not visible past the plugin boundary
  (PIPELINE-1 §4.2). A pipeline plugin does not write session state
  outside `Match.updated_session` — that channel discipline is what
  makes declined-plugin mutations safely discardable (PIPELINE-1
  §4.2). The removal therefore happens on the plugin's **next
  claiming match** for the session, or the entry ages out via the
  §3.2 TTL — whichever comes first. No plugin-side bookkeeping is
  needed for this: the owner is polled again on the next utterance
  and answers `error_code: "done"` again, so the removal is
  re-derived from the current poll round rather than remembered
  across utterances (§9.2 — the plugin holds no cross-utterance
  state). The redundant polls are cheap.

Removal is the only effect of `"done"` — it does not affect any
other iteration step.

Implementations MAY define additional codes; consumers MUST
treat unknown codes as opaque diagnostic strings.

---

## 5. Interactive response collection

A handler may suspend intent matching and receive the next user
utterance directly by setting `session.response_mode` (§2.2).
The mechanism is entirely session-resident; no side-band bus
events are required.

### 5.1 Entering and leaving response mode

A handler enters response mode by mutating `session.response_mode`
from within its dispatched handler. The mutation sets:

```json
{
  "skill_id": "<skill_id>",
  "expires_at": <now + timeout_seconds>
}
```

A handler entering response mode SHOULD emit an
`ovos.utterance.speak` (PIPELINE-1 §9.6) posing the question to
the user. That emission MUST be derived from the dispatch context
via `forward` (MSG-1 §5), carrying `session.response_mode`
downstream automatically. When such an emission is made, it
**MUST** carry `listen: true` (PIPELINE-1 §9.6) — the handler is
explicitly expecting a follow-up utterance, and every output
consumer must re-open the user input channel once delivery is
complete. Omitting `listen: true` is non-conformant: the user is
asked a question but the input channel is never re-opened.

If no `ovos.utterance.speak` is emitted, the handler MUST emit
`ovos.session.sync` (SESSION-2 §2.7) — the handler-lifecycle
`.complete` is orchestrator-emitted from stale dispatch context
and does not reflect handler-side mutations, especially for
out-of-process handlers.

A handler **leaves** response mode by removing `session.response_mode`
in its dispatched handler and emitting any Message carrying the
updated session forward.

There is no orchestrator-side wait timer. `expires_at` is
evaluated against the **orchestrator's clock** at match time.
Deployments where the handler runs on a different machine SHOULD
account for clock skew. No bus signal is emitted on timeout; the
handler manages its own user-facing timer.

### 5.2 Delivery of the next utterance

When the next inbound utterance on the session arrives at the
converse plugin's `match`, the plugin consults
`session.response_mode`:

1. **State check.** If `session.response_mode` is absent, the
   plugin proceeds to converse-handler iteration (§4.1 step 2). If
   present but `expires_at` has passed, the plugin treats it as
   stale and proceeds to converse-handler iteration. No
   response-mode delivery happens.

   **Discarding a stale entry.** An expired entry is **inert**:
   §4.1 step 1 and this step reject it on every subsequent
   utterance, so leaving it in the session changes no routing
   decision. It is cleared opportunistically, by whichever of
   these happens first, and no mutation exception to
   PIPELINE-1 §4.2 is created for either:

   - the plugin removes the field via `Match.updated_session` on
     its next claiming match for the session (a converse claim
     per §4.3, or a later response-mode delivery);
   - the orchestrator **MAY** drop an expired `response_mode`
     entry when it commits **any** `Match.updated_session` for
     the session — the entry is already unreachable, so dropping
     it is observably equivalent to keeping it.

   When the plugin's `match` returns `null` there is no
   `updated_session` channel and the entry simply stays until one
   of the above occurs, or until the session ends.

2. **Match for delivery.** With a valid response-mode entry,
   the plugin removes `session.response_mode` (single-shot
   delivery) and returns a `Match`:

   - `skill_id` = the holder (`session.response_mode.skill_id`);
   - `intent_name` = the reserved value `response`;
   - `lang` = the active language;
   - `utterance` = the first candidate;
   - `slots` = `{}`;
   - `updated_session` = the inbound session with
     `session.response_mode` removed (single-shot delivery).

   PIPELINE-1's first-match-wins iteration (§6.2 there) means
   subsequent pipeline stages do not run for this utterance —
   the converse plugin claimed first. This pre-emption is
   structural and depends on the deployer configuring the
   converse plugin at the front of `session.pipeline` (§4
   leading paragraph).

3. **Orchestrator dispatch.** The orchestrator dispatches
   `<skill_id>:response` per PIPELINE-1 §7 — identically to
   any other intent dispatch, with the full handler-trio and
   §3.1 activation.

   The dispatch `data` is the standard PIPELINE-1 §7.1 payload:

   | Key | Type | Required | Meaning |
   |-----|------|----------|---------|
   | `lang` | string | yes | The active language. |
   | `utterance` | string | yes | The first candidate. |
   | `slots` | object | yes | `{}` — response-mode delivery is not slot-bearing. |

   `skill_id` and `intent_name` are not repeated in the payload —
   they are the topic's `<skill_id>:<intent_name>` prefix and
   suffix (PIPELINE-1 §7.1).

4. **Handler.** The handler subscribed to `<skill_id>:response`
   runs and processes the awaited utterance. When it returns,
   PIPELINE-1 §8's trio `.complete` fires and the universal
   end-marker `ovos.utterance.handled` follows per §8.

   **Semantic contract.** A `:response` handler is the captive
   recipient of an utterance it explicitly solicited — the
   utterance is the direct answer to a question the handler
   already posed. The handler captures it as expected input;
   no NLU parsing decision is required to determine whether
   to handle it.

   **Continuing the response window.** Response mode is
   single-shot — each delivery clears the field. A handler
   that wants a further response round re-mutates
   `session.response_mode` from within its
   `<own_skill_id>:response` handler to set a fresh
   `{skill_id, expires_at}`, and emits any Message that
   carries the mutated session forward.

### 5.3 Cancellation

A handler cancels its own pending response by mutating
`session.response_mode` to absent (removing the field) in its
dispatched handler, and emitting any Message carrying the
mutated session forward. The next utterance on the session
sees no response-mode and is routed normally.

The orchestrator MAY additionally implement a deployer-defined
**user-cancel** signal — typically a configured "escape" or
"cancel" phrase recognised in the inbound utterance before
delivery. Recognition and reaction (clearing
`session.response_mode`, notifying the holder via a side-band
event the framework catches) are deployment / framework
concerns and **out of scope for this spec**.

A §3.3 transformer-driven mutation of `session.response_mode`
that removes or replaces the current holder also cancels the
pending response for the current utterance and onward.

### 5.4 Interruptibility

An interrupt signal that terminates a response-mode wait or an
in-flight converse poll is owned by OVOS-STOP-1 and is **out of
scope here**. The converse plugin and handlers are expected to
react to such a signal through
the framework's killable-thread machinery (or equivalent) —
not through any bus topic this spec owns.

OVOS-STOP-1 distinguishes two stop paths with different effects on
`session.converse_handlers`:

- A **targeted stop** (`intent_name: "stop"`, STOP-1 §4) removes the
  stopped skill from `session.active_handlers` only. The skill remains
  in `session.converse_handlers` and may still claim converse turns.
- A **global stop** (`intent_name: "global_stop"`, STOP-1 §5) empties
  both `session.active_handlers` and `session.converse_handlers` via
  `Match.updated_session`. The subsequent `global_stop` dispatch then
  stamps the stop plugin itself back onto the lists (§3.1 here is
  uniform, and `global_stop` is not a reserved intent_name), so the
  converse plugin sees at most the stop plugin on the next
  utterance's poll.

Clearing `session.response_mode` on a stop is owned by
**OVOS-STOP-1 §6.1** — a targeted stop clears the entry whose
`skill_id` matches the dispatch target, and a global stop removes
the field entirely, both via the stop plugin's
`Match.updated_session`. This spec adds no rule of its own there.

The polled-owner-side
reaction when an in-flight poll is interrupted is to emit
`<skill_id>.converse.pong` with `result: false` and
`error_code: "killed"` (§4.4) if still able to do so, or to fall back
on the per-owner timeout. An interrupted owner MUST NOT be removed from
`session.converse_handlers` as a side effect of the poll interrupt —
that removal is reserved for the global stop path above.

---

## 6. Bus topics

### 6.1 Active-list introspection

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.converse.active.list` | observer → orchestrator | Snapshot the current `session.converse_handlers` after the §3.2 pre-list prune has run. |

The orchestrator MUST reply on `ovos.converse.active.list.response`
with `data`:

| Key | Type | Meaning |
|-----|------|---------|
| `converse_handlers` | array of `{skill_id, activated_at}` | The post-prune list, matching the §2.1 wire shape. |

The session this snapshot describes is read from `context.session.session_id` of the response Message.

### 6.2 Converse poll round-trip

| Topic | Direction | Purpose | Shape |
|-------|-----------|---------|-------|
| `<skill_id>.converse.ping` | converse plugin → owner | Poll the owner with the current utterance (§4.2). | Dotted addressed (non-dispatch). |
| `<skill_id>.converse.pong` | owner → converse plugin | Owner's poll reply `{skill_id, result, error_code?}` (§4.2). | Dotted addressed (non-dispatch). |

The poll round-trip is **not** a PIPELINE-1 §7 dispatch and
does not fire the handler-lifecycle trio.

### 6.3 Reserved-intent-name dispatches

| Topic | Direction | Purpose | Shape |
|-------|-----------|---------|-------|
| `<skill_id>:converse` | orchestrator → owner | Converse claim dispatch — the orchestrator dispatches when the converse plugin returns a `Match` on `intent_name == "converse"` (§4.3). | OVOS-PIPELINE-1 §7 standard dispatch (fires handler-trio, stamps context, etc.). |
| `<skill_id>:response` | orchestrator → holder | Response-mode delivery dispatch — the orchestrator dispatches when the converse plugin returns a `Match` on `intent_name == "response"` (§5.2). | OVOS-PIPELINE-1 §7 standard dispatch (fires handler-trio, stamps context, etc.). |

The full bus surface this spec adds is **six topics**: one
introspection pair, one poll pair, and two reserved-intent-name
dispatch topics. Response-mode entry, exit, and cancellation
(§5.1 / §5.3) require no bus topic.

---

## 7. Interaction with intent context (OVOS-CONTEXT-1)

This spec's imperative surfaces and CONTEXT-1's declarative
surfaces are orthogonal: `session.intent_context` membership and
`session.converse_handlers` membership are independent.

The converse plugin runs at whatever position the deployer
configured in `session.pipeline`. PIPELINE-1's first-match-wins
iteration means a converse or response-mode claim pre-empts later
stages (including CONTEXT-1 gates) for that utterance — deployers
wanting strict pre-emption configure the converse plugin at the
front of `session.pipeline`.

Two notes: the CONTEXT-1 §4 decay tick fires on **every** dispatch
this spec produces (`:converse` and `:response` are ordinary
PIPELINE-1 §7 dispatches). The §3.2 TTL prune and the CONTEXT-1
§4 prune are independent and neither triggers the other.

---

## 8. Termination and the universal end-marker

Every utterance flow defined by this specification MUST emit
PIPELINE-1's universal end-marker `ovos.utterance.handled`
exactly once, after all handler work for the utterance has
concluded. Emission is the orchestrator's responsibility per
PIPELINE-1 §9 — this spec adds no new end-marker emission
mechanism and no new payload fields. Dispatches on
`<skill_id>:converse` and `<skill_id>:response` are ordinary
PIPELINE-1 §7 dispatches; the end-marker that follows each is
identical to the one that follows any other intent dispatch.

---

## 9. Conformance

### 9.1 Orchestrator

An **orchestrator** that claims conformance to this
specification MUST:

- claim the two session fields under SESSION-1 §2.2's field
  registry per §2.1–§2.2, in conformance with SESSION-1 §2.1's
  omission-not-`null` rule;
- maintain `session.converse_handlers` in-utterance via §3.1
  (automatic activation on every dispatch — uniform, no
  special cases), accept transformer mutations per §3.3, and
  apply the converse plugin's `Match.updated_session`
  mutations per PIPELINE-1 §4.2;
- run the §3.2 TTL prune at both boundaries when a deployer
  TTL is configured, never removing a non-expired
  `response_mode` holder (§2.1 exemption);
- enforce the §2.1 size cap, subject to the same exemption;
- dispatch every successful converse-plugin `Match` per
  OVOS-PIPELINE-1 §7 normally — no dispatch suppression, no
  out-of-band paths, full handler-trio (§8 there);
- emit `ovos.utterance.handled` per §8 (PIPELINE-1 §9.5) on
  every terminal dispatch this spec produces;
- **reject** registrations under OVOS-INTENT-4 that name an
  intent `converse` or `response` (PIPELINE-1 §7.3 registry;
  such registrations are malformed under OVOS-INTENT-4 §5.3/§6.3);
- answer the introspection request of §6.1 from its own
  session-state index, after running the pre-list prune of
  §3.2 (when a TTL is configured).

The orchestrator does **not** maintain response-mode wait
timers and does **not** synthesise dispatches on timeout,
cancel, or interrupt — those are owned by the
`session.response_mode` field's `expires_at` (§5.2) and by the
handler's framework (user-facing timer, cancel reaction).
Session persistence between utterances is owned by
OVOS-SESSION-2 (client-held for named sessions,
orchestrator-held for default).

### 9.2 Converse plugin

A **converse plugin** that claims the role defined in §4 MUST:

- load as a pipeline plugin per OVOS-PIPELINE-1 §3;
- be a **pure matcher** — `match` returns objects whose
  `skill_id` is another component's identity; the plugin bundles
  no handler of its own;
- follow the §4.1 iteration order in `match`:
  - check `session.response_mode` first; verify the §2.2 identity
    invariant against the post-prune `session.converse_handlers`
    (a pruned or unrecognised holder MUST NOT receive delivery —
    discard the entry via `Match.updated_session` and continue);
    if present, non-expired, and held by a listed owner, return a
    `Match` on `intent_name == "response"` clearing the field via
    `Match.updated_session` (§5.2); if expired, clear via
    `Match.updated_session` and continue;
  - conduct the §4.2 poll on the eligible set (SHOULD run in
    parallel; MUST bound it by both the per-owner timeout and the
    aggregate ceiling; MUST ignore malformed, foreign, and late
    pongs; MUST select the first claimer in recency order with the
    §4.1 tie-break; SHOULD skip `blacklisted_skills`); return a
    `Match` on `intent_name == "converse"` for the claimer, or
    `null` if none claim;
- return a conformant PIPELINE-1 §4.1 `Match`;
- MUST NOT dispatch `<skill_id>:converse` or `:response`
  directly — those are PIPELINE-1 §7 dispatches owned by the
  orchestrator.

The plugin SHOULD publish `converse` and `response` to the
per-pipeline passive index `ovos.pipeline.<pipeline_id>.intents.list`
(PIPELINE-1 §10). Internal activation policy (priority ordering,
consecutive-claim caps, allow/deny lists) is plugin business and
out of scope. The plugin holds **no cross-utterance state** — all
decision input comes from `session.response_mode` and
`session.converse_handlers`.

### 9.3 Handler

A **handler** that participates in any surface of this spec MUST:

- subscribe to `<own_skill_id>:converse` and perform the
  user-facing work when a converse claim is dispatched;
- subscribe to `<own_skill_id>:response` to receive utterances
  under response mode (§5.2).

The handler SHOULD subscribe to `<own_skill_id>.converse.ping`
to participate in polls. On each poll it MUST inspect
`data.utterances` / `data.lang`, commit to a claim decision, and
reply on `<own_skill_id>.converse.pong` via `.reply`
(MSG-1 §5.2) with `result: true` or `false`. A handler replying
`result: true` MUST handle the utterance fully when `:converse`
is dispatched. Silence is treated as `result: false` /
`error_code: "timeout"` — the handler remains in
`converse_handlers` but will never win a claim.

Response-mode entry and exit are per §5.1. The handler MUST NOT
rely on any bus event to set or clear response mode — the session
field is the wire surface.

A handler that wants to be removed from
`session.converse_handlers` SHOULD decline its next converse poll
with `error_code: "done"` (§4.4) — the plugin SHOULD remove it via
`Match.updated_session` (immediately when another owner claims the
same utterance, otherwise on its next claiming match or via the
§3.2 TTL). Without `"done"`, a
declining owner's position decays naturally once newer owners
are activated or the §3.2 deployer TTL expires.

A handler MUST NOT register an intent named `converse` or
`response` under OVOS-INTENT-4 — these names are reserved at
OVOS-PIPELINE-1 §7.3.

### 9.4 Observer

An **observer** MAY consume the introspection topic of §6.1 to
discover the current converse-handler list.

An observer that sees a Message on `<skill_id>:converse` or
`<skill_id>:response` is seeing an ordinary OVOS-PIPELINE-1 §7
dispatch, distinguishable from other intent dispatches solely
by the reserved `intent_name`. The handler-trio and end-marker
fire as for any dispatch.

An observer that reads `session.response_mode` from any
Message can identify whether the session has a pending
response window (field present, `expires_at` in the future)
and who holds it (`skill_id`).

### 9.5 Transformer

A **transformer** (OVOS-TRANSFORM-1) that mutates
`session.converse_handlers` or `session.response_mode`:

- SHOULD do so at the metadata-transformer hook (TRANSFORM-1
  §3.3) by default, per §3.3 here;
- MAY do so at any other transformer hook with the
  documented timing consequences of §3.3;
- MUST preserve the §2.1 invariants on `converse_handlers`
  (head-first recency, no duplicates, size cap), or accept
  that the orchestrator will normalize the result;
- MUST understand that mutating `session.response_mode`
  directly affects the converse plugin's delivery decision on
  the current and subsequent utterances per §5.2 — removing
  the field cancels the pending response.

---

## 10. Non-goals

This specification deliberately does not:

- prescribe **how** a converse plugin implements its poll
  round-trips beyond §4.1 (parallelism recommended; selection
  by first claimer in recency order);
- prescribe **what** a handler should say or do when it claims,
  leaves response mode, or its response window times out;
- prescribe converse-plugin **activation policy** (priority
  ordering, consecutive-claim caps, allow/deny lists);
- define a **conversation transcript** or cross-utterance
  memory — handlers manage their own state;
- define **escape phrases**, wake-word interactions, barge-in,
  or any signal originating below the utterance layer;
- define the **wire shape of an interrupt signal** — owned
  by OVOS-STOP-1 (§5.4);
- define **how session persists between utterances** — owned
  by OVOS-SESSION-2;
- replace CONTEXT-1's declarative continuous-dialog surface —
  the two coexist per §7;
- expose **dedicated bus events** for entering, leaving, or
  modifying response mode — the session field and in-handler
  mutation (SESSION-2 §2.6) are the wire surface;
- introduce **any dispatch-mechanism exception** to PIPELINE-1
  §6 or §7 — the two reserved intent_names dispatch through
  the standard §7 path; the reservation is a namespace lease,
  not a dispatch modification.

---

## See also

- **OVOS-PIPELINE-1** — pipeline-plugin contract, `Match` shape and
  `updated_session` (§4.2), match-phase latency discipline (§4.4),
  dispatch and `active_handlers` (§7.1), reserved intent-name
  registry (§7.3).
- **OVOS-MSG-1** — envelope, the `session` carrier, and the
  `forward` / `reply` / `response` derivations (§5).
- **OVOS-SESSION-1** — session field registry and the
  omission-not-`null` rule.
- **OVOS-SESSION-2** — in-utterance mutation boundaries (§2.6) and
  the default-session ownership rule (§5).
- **OVOS-STOP-1** — interrupt signal, and the owner of
  `response_mode` clearing on stop (§6.1).
- **OVOS-FALLBACK-1** — the sibling per-skill poll, whose
  bus-exchange exception this specification mirrors (§6.1).
- **OVOS-CONTEXT-1** — the declarative continuous-dialog surface
  this specification is orthogonal to (§7).
- **OVOS-TRANSFORM-1** — the six lifecycle hooks at which the
  session fields claimed here MAY be mutated (§3.3).
- **OVOS-INTENT-4** — registration model and `ovos.skill.deregister`
  (§8.4).
