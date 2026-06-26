# Persona Pipeline Plugin Specification

**Spec ID:** OVOS-PERSONA-1 · **Version:** 2 · **Status:** Draft

This specification defines the concept of a **persona** in a
voice-operating-system pipeline — a complete conversational agent
that, when active, claims every utterance that reaches its pipeline
stage and generates natural-language responses. It defines the
`persona_id` session field used to select the active persona, the
interaction rules for summoning and dismissing personas, and the
pipeline-positioning constraints that let the orchestrator enforce
deterministic skills-first behaviour with personas acting as a
fallback layer.

It builds on four companion specifications:

- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the pipeline-plugin contract, the `Match`
  shape, dispatch, the handler-lifecycle trio, and
  `session.active_handlers`;
- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  routing keys, session carrier, and derivations every Message
  defined here travels in;
- the *Session Carrier Wire Shape Specification* (OVOS-SESSION-1) —
  the session field registry and the omission rule;
- the *Active Handlers and Interactive Response Specification*
  (OVOS-CONVERSE-1) — the conversation cycle that routes follow-up
  utterances to the persona plugin during multi-turn interactions.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- **what a persona is** (§2) — the conceptual definition of a persona
  as a complete conversational agent;
- **the `persona_id` session field** (§3) — the session-resident
  field that identifies which persona, if any, is active for the
  current session;
- **no-persona mode** (§4) — the deterministic skills-only pipeline
  when no persona is active;
- **summon (activating a persona)** (§5) — how a persona is
  activated for a session;
- **dismiss (deactivating a persona)** (§6) — how a persona is
  deactivated;
- **the match contract** (§7) — how a persona plugin claims utterances
  when active;
- **the handler contract** (§8) — how the handler generates responses,
  including the out-of-band query interface;
- **multiple persona coexistence** (§9) — interaction rules when
  multiple persona plugins or identities are present;
- **pipeline positioning** (§10) — where persona stages sit in the
  pipeline relative to skills, stop, and fallback stages;
- **bus surface** (§11);
- **conformance** (§12).

It does **not** define:

- **the internal machinery** of a persona — whether the handler uses a
  language model, a rule-based engine, a retrieval system, or any
  other approach is entirely the plugin's business. The spec fixes
  only the observable bus contract;
- **persona configuration format** — the system prompt, identity
  definition, solver wiring, or capability declaration is a
  deployment concern;
- **conversation-history persistence format** — the plugin MAY hold
  history internally or project it into session fields; either is
  conformant;
- **vocabulary files, matching algorithms, or confidence thresholds**
  — the plugin decides when to claim an utterance;
- **output transformation** — cross-cutting personality shaping (tone,
  register, post-processing) is handled by dialog transformers
  (OVOS-TRANSFORM-1), not by this specification;
- **GUI, TTS, or output-layer behaviour** — response delivery beyond
  `ovos.utterance.speak` is out of scope.

---

## 2. What is a persona

A **persona** is a complete conversational agent — an assistant with
its own identity, personality, and capabilities. From the user's
perspective, summoning a persona replaces the deterministic pipeline
with a different agent. Each persona has:

- an **identity** — a `persona_id` string that uniquely names it
  within a deployment;
- a **personality** — the behaviour and response style that
  characterise it;
- **capabilities** — what it can answer or do for the user.

A persona is a black box from this specification's perspective. Whether
it is implemented with a language model, a rule-based engine, a
retrieval system, or any other approach is entirely the plugin's
business. The spec fixes only the observable bus contract.

A persona plugin is a pipeline plugin (PIPELINE-1 §3) that hosts one
or more personas. When a persona is active for a session, the plugin
claims every utterance that reaches its pipeline stage and returns a
natural-language response via its bundled handler.

**Personality and dialog transformers.** A persona is responsible for
the content of its responses. Cross-cutting output transformations —
tone, verbosity, language register, post-processing — are better
handled by dialog transformers (OVOS-TRANSFORM-1 §3.5), which run
after `ovos.utterance.speak` regardless of which pipeline stage
generated the response. A deployment MAY combine both: a persona that
provides content and dialog transformers that shape it.

---

## 3. The `persona_id` session field

This specification claims one optional session field per the
OVOS-SESSION-1 §2.1 registry mechanism.

| Field | Wire type | Owner |
|-------|-----------|-------|
| `persona_id` | string | §3 (this specification) |

`persona_id` is an opaque string identifying which persona identity
is active for the current session. The value space is
deployment-defined. This specification places no constraint on the
string beyond the opaque-string rules of OVOS-SESSION-1 §2.2 (no `:`,
no whitespace).

`persona_id` is a **single value** — exactly one persona identity may
be active per session at a time. Composition of multiple personas
within a single session is not in scope.

**Semantics:**

- When `persona_id` is **absent** (not set), no persona is active.
  Persona stages MUST return `None` for all utterances except
  those matching an embedded persona command (§7.1 route 1) or
  handled by a persona-fallback stage (§7.1 route 3).
- When `persona_id` is **present and non-empty**, the corresponding
  persona is active. Persona stages whose supported identities
  include this value MUST claim utterances that reach them (§7).
- An empty string is semantically equivalent to absent.

**Propagation:**

`persona_id` follows the standard session propagation rules of
OVOS-SESSION-1 §4: it is carried unchanged across all derivations
and persists across utterances in the same session.

**Wire weight:**

Per OVOS-SESSION-1 §3.4, a producer that intends no active persona
(the default) SHOULD omit the field rather than emit an empty value.

---

## 4. No-persona mode

**No-persona mode** is the pipeline state in which no persona is
active (`persona_id` is absent from the session). In this mode:

- persona stages **MUST** decline every utterance that does not
  match an embedded persona command (§7.1 route 1);
- the pipeline operates as a purely deterministic, skill-driven
  system — only intent-matching and fallback stages handle
  utterances.

No-persona mode is the deployment default. Every session starts in
no-persona mode unless a client or layer-2 substrate sets `persona_id`
on the initial utterance. In no-persona mode, utterances that reach a
persona-fallback stage (§7.1 route 3, §9) are still handled by that
stage; all other persona stages return `None`.

---

## 5. Summon (activating a persona)

**Summon** is the act of activating a persona for a session. The
effect of summon is to set `persona_id` in the session.

- **Self-summon.** The persona plugin itself detects the summon
  utterance during `match` (§7.1 route 1), claims the utterance
  (emitting a confirmation response), and sets `persona_id` via
  `Match.updated_session`. The persona handles the summon utterance
  directly; the updated `persona_id` activates the persona for
  subsequent utterances.
- **One-off query.** The persona plugin detects an `ask` utterance
  during `match` (§7.1 route 1), claims it, and generates a response
  via its handler — but does **not** set `persona_id`. The session
  state is unchanged; the persona answers the question without
  activating permanently. This lets the pipeline handle "ask Alice
  about X" as a single utterance without altering persona state.
  If the referenced persona is not supported by this plugin, the
  plugin SHOULD return `None` (letting the utterance fall through to
  fallback) or MAY claim it with an error response.
- **External summon.** A component outside the persona plugin sets
  `persona_id` on the inbound session. The persona plugin is not
  involved in the summon utterance — it only sees the new
  `persona_id` on the next utterance and activates accordingly.
  External summon occurs whenever `persona_id` appears in the
  inbound session, placed there by:

  - the **client** on the initial utterance message;
  - a **pipeline plugin** (e.g., a skill with a registered intent)
    via handler-side session mutation (OVOS-SESSION-2 §2.6);
  - a **session sync** (`ovos.session.sync`) from any component;
  - the **orchestrator** as a policy decision.

**Unique identity:** A summon MUST reference an existing
`persona_id`. A summon that names an unknown persona has no effect:
the orchestrator or summoning component SHOULD log at WARN and leave
`persona_id` unchanged.

---

## 6. Dismiss (deactivating a persona)

**Dismiss** is the act of deactivating the active persona for a
session, returning the pipeline to no-persona mode (§4). The effect
of dismiss is to clear `persona_id` from the session.

A dismiss occurs when `persona_id` is explicitly cleared from the
session by:

- the **persona plugin itself** — detecting a release intent during
  `match` (§7.1 route 1) and clearing `persona_id` via
  `Match.updated_session`;
- the **stop cascade** (OVOS-STOP-1) — clearing `persona_id` as
  part of the escape-hatch behaviour. The stop plugin SHOULD clear
  `persona_id` so that "stop" returns the session to
  deterministic mode;
- a **pipeline plugin** via handler-side session mutation;
- a **session sync** (`ovos.session.sync`) from any component.

Per OVOS-SESSION-1 §4, omitting `persona_id` from a message does
not clear it — the field is carried forward unchanged. A client
that wants to dismiss a persona must send an explicit clear (empty
string or absent field with intent to remove) via session sync or
a summoning plugin that handles the release intent.

When dismiss is detected, any in-progress generation for the session
**SHOULD** cease. The persona's per-session state (conversation
history, etc.) **SHOULD** be preserved for resumption if the same
persona is re-summoned.

---

## 7. Match contract

### 7.1 When to claim

A persona plugin's `match` function evaluates two pathways in
order:

1. **Embedded persona commands.** The plugin detects utterances
   that reference persona functionality directly — summon, release,
   one-off query (e.g. "ask Alice about X"), list personas, check
   active persona. When one of these intents is detected (via the
   plugin's own intent matching), the plugin returns a `Match`:
   - summon/release intents set or clear `persona_id` via
     `Match.updated_session`;
   - one-off queries (`ask`) claim the utterance and dispatch to the
     handler, which generates a response but does **not** change
     `persona_id` — the session state is unchanged;
   - list/check intents claim the utterance for introspection
     responses without mutating session state.
   This pathway runs **regardless** of the current
   `session.persona_id` value — it is how the plugin self-summons,
   handles one-off queries, or self-releases when no persona is yet
   active.

2. **Active-persona catch-all.** If no embedded command was
   detected, the plugin checks `session.persona_id`:
   - If `session.persona_id` is absent or empty → return `None`
     (no-persona mode, §4); unless this plugin is registered as a
     persona-fallback stage for this pipeline position (route 3).
   - If `session.persona_id` is set to a value this plugin supports
     → return a `Match`.
   - If `session.persona_id` is set to a value this plugin does NOT
     support → return `None` (let another persona stage or fallback
     handle it).

3. **Persona-fallback catch-all.** A persona plugin MAY register a
   secondary `fallback_pipeline_id` (§9) in addition to its main
   `pipeline_id`. When the pipeline invokes the plugin under its
   `fallback_pipeline_id`, the match rules are:
   - If `session.persona_id` is absent or empty → claim the utterance
     (this is the fallback case — no persona is active and no other
     stage matched).
   - If `session.persona_id` is set to a value this plugin supports
     → claim (consistent with route 2).
   - If `session.persona_id` is set to a value another plugin supports
     → return `None` (respect the active persona).

   The persona-fallback stage is how utterances are handled when no
   skill matched and no persona is active. It is positioned at the end
   of the pipeline, after all skill stages (§10).

### 7.2 Active-persona catch-all

When route 2 above applies (no embedded persona command detected,
and `persona_id` is present and supported), the plugin **MUST**
claim every utterance that reaches it, subject only to its
supported-identity check. This is the defining behavioural
characteristic of a persona: an active persona consumes everything
that reaches its pipeline stage.

The plugin **MAY** apply lightweight gate logic before claiming
(language detection, minimum utterance length, blacklist), but it
**MUST NOT** use confidence thresholds or intent-matching to decide
whether to claim — those belong to the deterministic pipeline, not
to an active persona.

### 7.3 Latency discipline

Per PIPELINE-1 §4.4, a persona plugin **SHOULD** return a `Match`
immediately and defer all computationally expensive work (generation,
model inference, network calls) to the handler phase. The match phase
is a routing decision; the generation phase belongs in the handler.

### 7.4 Match shape

When claiming, the plugin returns a `Match` (PIPELINE-1 §4.1) with:

| Field | Value |
|-------|-------|
| `skill_id` | The plugin's own `pipeline_id` (self-matching per PIPELINE-1 §7.0). |
| `intent_name` | A non-empty string chosen by the plugin (e.g. `"persona"`, `"chat"`). |
| `lang` | The resolved BCP-47 language tag of the match. |
| `slots` | MAY be empty. |
| `utterance` | The specific candidate string from the input list. |
| `updated_session` | Present when the plugin modifies session state as part of the match. |

### 7.5 Session mutation at match time

A persona plugin **MAY** mutate session state via
`Match.updated_session` (PIPELINE-1 §4.2). Typical uses:

- set or clear `session.persona_id` as part of a summon or release
  match (§7.1 route 1);
- modify any other session field it owns per the OVOS-SESSION-1 §2.1
  registry mechanism.

One-off query matches (the `ask` command) do **not** mutate
`persona_id` — the session state passes through unchanged.

The `updated_session` pathway is **only effective for a claiming
match**: a plugin that returns `None` has any match-phase session
mutations discarded at the plugin boundary.

---

## 8. Handler contract

### 8.1 Response generation

The handler dispatched on `<pipeline_id>:<intent_name>` receives the
standard dispatch payload (PIPELINE-1 §7.1): `lang`, `utterance`,
`slots`. The handler generates a natural-language response and emits
it via `ovos.utterance.speak` (PIPELINE-1 §9.6).

A handler **MAY** emit zero, one, or multiple `ovos.utterance.speak`
Messages. Multiple emissions are conveyed in order and the output stage
**SHOULD** preserve that order.

### 8.2 Handler-side session mutation

The handler **MAY** mutate session state in place per
OVOS-SESSION-2 §2.6 handler-boundary rules. All emissions via
`forward` / `reply` / `response` (OVOS-MSG-1 §5) carry the mutated
session forward.

Typical handler-side mutations:
- modifying `session.pipeline` for subsequent utterances;
- setting or clearing `session.persona_id`;
- updating per-session persona state.

### 8.3 Long-running handlers

A persona handler **MAY** block for an unbounded duration
(PIPELINE-1 §6.5). When emitting a prompt that awaits a reply, the
`ovos.utterance.speak` Message **MUST** carry `listen: true` so that
the audio output service reopens the microphone after speech.

Multi-turn interactions are handled through multiple consecutive
dispatches: the handler emits its prompt and returns; on the next
utterance, the converse plugin (OVOS-CONVERSE-1) routes
`<pipeline_id>:converse` to the persona because
`skill_id == pipeline_id` in CONVERSE-1's active-handler check.

A persona plugin that supports multi-turn **SHOULD** subscribe to
`<own_pipeline_id>:converse` to receive follow-up utterances.

### 8.4 Conversation history

A persona plugin **MAY** maintain conversation history keyed on
`session.session_id`, following the MAY-internal pathway of
OVOS-SESSION-2 §2.4. History too large to project into session-resident
fields is held in plugin-internal storage with best-effort resumption.

A persona plugin **SHOULD** project summary state into a
session-resident field registered per OVOS-SESSION-1 §2.1 — so that
resumption across orchestrator restarts or multi-orchestrator
deployments retains basic continuity even when full history is held
internally.

### 8.5 Out-of-band query

A persona plugin **MAY** expose an out-of-band query interface that
lets any component (skill, CLI, plugin) ask the persona a question
without going through the pipeline. This is a request-response
pattern on two bus topics:

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.persona.query` | any component → persona | Out-of-band query |
| `ovos.persona.answer` | persona → requesting component | Query response |

The request payload:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `persona_id` | string | yes | Target persona identity. |
| `utterance` | string | yes | Query text. |

Session context for history continuity is read from
`context.session.session_id` of the request Message.

The plugin generates a response for the specified `persona_id` using
the `reply()` derivation (OVOS-MSG-1 §5) to route back to the caller.
If `persona_id` is not supported by this plugin, the plugin **MUST**
respond with `None` rather than silently drop the request.

The response payload:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `persona_id` | string | yes | The persona that answered. |
| `utterance` | string | yes | Echo of the query. |
| `response` | string | yes | The generated response text. |

This interface is **not** part of the utterance lifecycle — it
bypasses the pipeline, the dispatch mechanism, and the
handler-lifecycle trio. It is a direct query that exists alongside
the pipeline-based flow. The use case is information retrieval
(fact lookup, classification, brief generation) where a skill or
plugin needs the persona's answer without changing conversational
state or triggering the full utterance lifecycle. A persona plugin that implements this
**MUST** still support the pipeline-based match → dispatch flow
defined in §7–§8.4.

The out-of-band query **MUST NOT** mutate `session.persona_id` or
change the active persona state. It is a stateless query within
the context provided.

### 8.6 Stop awareness

A persona handler **MUST** check for stop signals during long-running
generation and **MUST** cease generation and return promptly when a
stop signal (OVOS-STOP-1) arrives for its session.

### 8.7 Persona discovery

A persona plugin **MAY** expose a discovery interface that lets any
component enumerate the `persona_id` values it supports. This is a
request-response pattern on two bus topics:

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.persona.list` | any component → persona | Enumerate supported persona identities |
| `ovos.persona.list.response` | persona → requesting component | Response listing supported identities |

The plugin responds to `ovos.persona.list` with:

```json
{
  "pipeline_id": "<the plugin's pipeline_id>",
  "fallback_pipeline_id": "<optional: the plugin's fallback pipeline_id>",
  "personas": [
    {
      "persona_id": "alice",
      "name": "Alice",
      "tags": ["cooking", "recipes", "food"]
    },
    {
      "persona_id": "bob",
      "name": "Bob",
      "tags": ["code", "python", "debugging"]
    }
  ]
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `pipeline_id` | string | yes | The plugin's main pipeline_id. |
| `fallback_pipeline_id` | string | no | The plugin's persona-fallback pipeline_id, if registered (§9). |
| `personas` | array | yes | One object per supported persona identity. |
| `personas[].persona_id` | string | yes | The persona identity (§3). |
| `personas[].name` | string | no | Human-readable display name. |
| `personas[].tags` | string[] | no | Freeform capability or domain tags. Vocabulary is deployment-defined. |

`tags` are advisory — they exist so routing skills and UIs can make
informed summon decisions. This specification does not standardise tag
vocabulary; deployments SHOULD document their tag taxonomy.

This interface is independent of the utterance lifecycle. Each persona
plugin responds with its own supported set. A component that needs the
full deployment-wide set MUST query each persona stage individually or
use a deployment-specific aggregation layer.

---

## 9. Multiple persona coexistence

A deployment MAY load multiple persona plugins under different
`pipeline_id` values, each hosting one or more `persona_id` values.
This is the primary mechanism for pluggable personalities and
capabilities: each persona plugin is a self-contained agent with its
own identity, capabilities, and tags; a routing skill or UI selects
among them by setting `session.persona_id`.

**Identity namespace:** `persona_id` values SHOULD be unique within
a deployment. When two plugins both claim the same `persona_id`, the
first one in pipeline order claims every utterance for that identity;
the second never matches. Deployments SHOULD avoid this; if detected
at runtime (e.g. via `ovos.persona.list` responses), the orchestrator
SHOULD log at WARN.

**Capability-based routing.** A skill or UI that wants to select among
multiple loaded personas SHOULD query `ovos.persona.list`, collect the
`tags` arrays from all responses, and use them to pick a `persona_id`
to set on the session. The routing logic — tag matching, user
preference, context — is a deployment or skill concern, not a plugin
concern. The persona plugin only declares its tags; it does not
participate in selection.

**Persona-fallback pipeline_id.** A persona plugin MAY register a
secondary pipeline position — its `fallback_pipeline_id` — that acts
as a catch-all when no persona is active and no skill matched. The
plugin exposes `fallback_pipeline_id` in its `ovos.persona.list`
response (§8.7). A deployment that enables persona-fallback includes
this `pipeline_id` as a stage in its configured pipeline
(OVOS-PIPELINE-1 §5.1), positioned per §10. When invoked
under this id, the plugin applies route 3 match logic (§7.1): it
claims utterances where `session.persona_id` is absent or matches a
supported identity, and returns `None` when another persona's id is
active.

Only one persona-fallback stage SHOULD be active in a given pipeline.
When multiple plugins expose a `fallback_pipeline_id`, pipeline order
determines which one handles unmatched utterances. Deployments SHOULD
designate one persona as the fallback and exclude others'
`fallback_pipeline_id` from the active pipeline.

**Dynamic registration.** A persona plugin MAY expose bus topics
for runtime persona management without restart:

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.persona.register` | any component → persona | Register a new persona at runtime |
| `ovos.persona.deregister` | any component → persona | Deregister an existing persona |

The payload for `ovos.persona.register`:

```json
{ "persona_id": "<the persona identity>" }
```

The payload for `ovos.persona.deregister`:

```json
{ "persona_id": "<the persona identity to remove>" }
```

These topics are **MAY** — a deployment that does not need runtime
persona management can omit them. When present, the plugin validates
the `persona_id` namespace uniqueness rules above and rejects
duplicate or unknown registrations.

---

## 10. Pipeline positioning

A deployment that includes persona stages **SHOULD** place them
after deterministic intent-matching stages and after the stop stage,
so that skills handle their intents first and the escape hatch can
interrupt an active persona.

A persona plugin's main `pipeline_id` (active-persona catch-all,
§7.1 route 2) SHOULD appear after skill stages. Its optional
`fallback_pipeline_id` (persona-fallback, §7.1 route 3) SHOULD
appear after all skill stages and at or near the end of the pipeline,
before any last-resort fallback.

A typical ordering with both positions:

```
session.pipeline: [
  "stop_high",          # interrupt (escape hatch)
  "converse",           # active-handler poll
  "skill_high",         # deterministic registered intents
  "skill_medium",
  "persona",            # active-persona catch-all (route 2)
  "persona_fallback",   # persona-fallback catch-all (route 3)
  "fallback_low"        # last-resort fallback
]
```

`persona` and `persona_fallback` are different pipeline_id values
registered by the same plugin. A deployment that does not use the
persona-fallback feature simply omits `persona_fallback` from the
pipeline.

A deployment **MAY** place a persona stage earlier when the persona
is specialised for a domain that should pre-empt general-purpose
matchers. Multiple persona stages at different pipeline positions
are conformant.

---

## 11. Bus surface

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `<pipeline_id>:<intent_name>` | orchestrator → persona | Active-persona dispatch (§8.1, §7.1 routes 1–2) |
| `<pipeline_id>:converse` | orchestrator → persona | Follow-up dispatch during multi-turn interactions (§8.3) |
| `<fallback_pipeline_id>:<intent_name>` | orchestrator → persona | Persona-fallback dispatch (§7.1 route 3, §9) |
| `ovos.persona.query` | any component → persona | Out-of-band query (§8.5) |
| `ovos.persona.answer` | persona → any component | Query response (§8.5) |
| `ovos.persona.list` | any component → persona | Enumerate supported persona identities (§8.7) |
| `ovos.persona.list.response` | persona → any component | Supported-identity listing (§8.7) |
| `ovos.persona.register` | any component → persona | Runtime persona registration (§9) |
| `ovos.persona.deregister` | any component → persona | Runtime persona deregistration (§9) |
| `ovos.persona.activated` | persona → broadcast | A persona has become active for a session (best-effort) |
| `ovos.persona.dismissed` | persona → broadcast | A persona has been dismissed from a session (best-effort) |

`ovos.persona.activated` payload: `{ "persona_id": "...", "session_id": "..." }`.
`ovos.persona.dismissed` payload: `{ "persona_id": "...", "session_id": "..." }`.
These are advisory signals emitted on a best-effort basis; consumers
**MUST NOT** rely on them for correctness. Session state is authoritative.

All dispatch topics follow the PIPELINE-1 §7 topic shape and fire the
handler-lifecycle trio (PIPELINE-1 §8). The persona handler emits
`ovos.utterance.speak` (PIPELINE-1 §9.6) for each natural-language
response it generates.

A persona plugin **SHOULD** respond to
`ovos.pipeline.<own_pipeline_id>.intents.list` per PIPELINE-1 §10,
listing the intent names it dispatches on.

---

## 12. Conformance

### A persona pipeline plugin **MUST**:

- expose a `match(utterances, lang, session) → Match | None`
  operation per PIPELINE-1 §4;
- return a `Match` with `skill_id` equal to its own `pipeline_id`
  (self-matching identity, PIPELINE-1 §7.0);
- evaluate embedded persona commands (summon, release, one-off
  query, list, check) in `match` **before** checking
  `session.persona_id`, and handle each according to its type —
  set or clear `persona_id` for summon/release, leave it unchanged
  for one-off queries (§7.1 route 1);
- after the summon/release check, read `session.persona_id` and
  return `None` when the field is absent or empty (§7.1);
- return `None` when `session.persona_id` is set to a value it does
  not support (§7.1);
- claim every utterance that reaches it when `session.persona_id` is
  set to a value it supports, subject only to lightweight gate logic
  (§7.2);
- set `Match.lang` to the resolved language of the match;
- subscribe to `<own_pipeline_id>:<intent_name>` to receive its own
  dispatch;
- derive each `ovos.utterance.speak` emission from the dispatch
  Message per OVOS-MSG-1 §5 derivation semantics (PIPELINE-1 §9.6);
- cease generation and return promptly on stop signals for its session
  (§8.6).

### A persona pipeline plugin **SHOULD**:

- return a `Match` immediately and defer generation to the handler
  phase (§7.3);
- subscribe to `<own_pipeline_id>:converse` to support multi-turn
  interactions (§8.3);
- carry `listen: true` on `ovos.utterance.speak` when used as a
  prompt that awaits a reply (§8.3);
- respond to `ovos.pipeline.<own_pipeline_id>.intents.list` per
  PIPELINE-1 §10;
- respond to `ovos.persona.list` with its supported `persona_id`
  values (§8.7);
- project summary state into a session-resident field registered per
  OVOS-SESSION-1 §2.1 for resumption safety (§8.4).

### A persona pipeline plugin **SHOULD**:

- include `tags` per persona in its `ovos.persona.list` response so
  that routing skills and UIs can make informed summon decisions (§8.7,
  §9).

### A persona pipeline plugin **MAY**:

- support multiple `persona_id` values (§9);
- hold conversation history in plugin-internal storage per the
  MAY-internal pathway of OVOS-SESSION-2 §2.4 (§8.4);
- set `session.persona_id` via `Match.updated_session` when the match
  resolves the persona identity (§7.5);
- expose an out-of-band query interface on `ovos.persona.query` /
  `ovos.persona.answer` (§8.5);
- register a `fallback_pipeline_id` and apply route 3 match logic when
  invoked under it (§7.1, §9);
- support runtime persona management on `ovos.persona.register` /
  `ovos.persona.deregister` (§9).

### A deployment that includes persona plugins **SHOULD**:

- position persona stages after deterministic skills and after the
  stop stage in `session.pipeline` (§10);
- position the persona-fallback stage (`fallback_pipeline_id`) after
  all skill stages and before last-resort fallback (§10);
- ensure `persona_id` values do not overlap across loaded persona
  plugins (§9);
- designate at most one persona-fallback stage in the active pipeline
  (§9).

---

## See also

- *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
  — the pipeline-plugin contract, the `Match` shape, dispatch
  polymorphism, the handler-lifecycle trio, and `ovos.utterance.speak`.
- *Bus Message Specification* (OVOS-MSG-1) — the envelope and
  derivations used for all bus communication.
- *Session Carrier Wire Shape Specification* (OVOS-SESSION-1) — the
  session field registry and the omission rule; the persona spec
  claims the `persona_id` field via §2.1.
- *Session Lifecycle and State Ownership Specification*
  (OVOS-SESSION-2) — the SHOULD-project / MAY-internal state
  pathways and the mutation boundaries.
- *Stop Pipeline Plugin Specification* (OVOS-STOP-1) — the stop
  cascade that clears `persona_id` on dismiss (§6).
- *Active Handlers and Interactive Response Specification*
  (OVOS-CONVERSE-1) — the conversation cycle that routes follow-up
  utterances to the persona plugin via `<pipeline_id>:converse`
  (§8.3).
