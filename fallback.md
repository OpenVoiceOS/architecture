# Fallback Pipeline Plugin Specification

**Spec ID:** OVOS-FALLBACK-1 · **Version:** 2 · **Status:** Draft

This specification defines the **fallback pipeline plugin** — a
pipeline plugin that handles utterances no earlier stage claimed.
It maintains a registry of fallback skills, constructs an ordered
handler pool from that registry and the session's preferences,
queries skills in pool order to find a willing handler, and
dispatches to the first one that claims the utterance.

It builds on four companion specifications:

- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the pipeline-plugin contract, the `Match`
  shape, dispatch, the handler-lifecycle trio, the pipeline
  composition model, and the reserved-intent-name registry;
- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  routing keys, session carrier, and derivations every Message
  defined here travels in;
- the *Session Specification* (OVOS-SESSION-1) —
  the session field registry and the omission rule;
- the *Intent and Entity Registration Bus Contract*
  (OVOS-INTENT-4) — the session-scoped registration model.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- **the fallback plugin role** (§2) — a pipeline plugin that
  delegates to registered fallback skills;
- **skill registration** (§3) — how a skill declares itself as a
  fallback handler with a default ordering priority;
- **session fields** (§4) — the two session-resident fields that
  control pool ordering and access control;
- **pool construction** (§5) — how the ordered handler pool is
  derived from registration, session preference, and policy;
- **the match contract** (§6) — the sequential per-skill query,
  selection algorithm, and Match shape;
- **the dispatch and handler contract** (§7) — what the selected
  skill receives;
- **pipeline positioning** (§8) — where fallback stages sit and
  how multiple stages interleave with other plugins;
- **bus surface** (§9);
- **conformance** (§10).

It does **not** define:

- **what a fallback skill does internally** — whether it queries a
  language model, returns a canned response, or calls an external
  service is the skill's business;
- **per-stage range boundaries** — which priority numbers belong to
  which stage is deployment configuration; §3.3 provides
  non-normative guidance on the recommended tiers;
- **query timeout values** — these are deployment-tunable; §6.1
  fixes only a recommended default ceiling.

---

## 2. The fallback plugin role

A **fallback pipeline plugin** is a pipeline plugin (PIPELINE-1 §3)
that:

- occupies one or more positions in `session.pipeline` (§8);
- maintains a registry of skills that have declared themselves
  fallback handlers, each with a default ordering priority (§3);
- at each match call, constructs an ordered handler pool from the
  registry, session preference, and policy (§5);
- queries pool members in order until a willing skill is found (§6);
- returns a `Match` delegating to that skill, or `None` if the
  pool is exhausted (§6).

The fallback plugin does not handle utterances itself.

**The defining property of a fallback skill** is that it evaluates
the utterance using its own internal logic rather than relying on
an intent model registered with the pipeline. Regular skills
declare their coverage through INTENT-4 registrations — vocabulary,
templates, or entity patterns — and a pipeline plugin selects them
by matching those patterns against the utterance. A fallback skill
declares no intent patterns; instead it receives the raw utterance
at query time (§6.1) and decides for itself whether it can respond.
This is the appropriate pattern when an utterance domain cannot be
reliably modelled as a grammar or template: open-domain question
answering, natural-language database queries, language-model
completions, and any skill whose coverage is determined
programmatically at runtime.

This mechanism is analogous to **OVOS-CONVERSE-1**'s per-skill
converse poll, where active handlers evaluate the current utterance
themselves before the plugin dispatches to one of them.

---

## 3. Skill registration

### 3.1 Register

A skill that wishes to receive fallback dispatches emits:

`ovos.fallback.register`

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The skill's identity. MUST equal `context.skill_id` of this Message. |
| `priority` | integer | yes | Default ordering hint. Lower values sort earlier when no session preference overrides. |

The plugin adds the skill to its registry. Re-registration with
the same `skill_id` replaces the prior entry. The plugin MUST NOT
index a registration where the payload `skill_id` differs from
`context.skill_id`.

### 3.2 Deregister

`ovos.fallback.deregister`

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The skill's identity. |

Removes the skill from the registry. Unknown `skill_id` is a no-op.

### 3.3 Priority guidance

Priority values have no normative meaning beyond ordering — lower
values appear earlier in the pool. This section provides
non-normative guidance for skill authors and deployers.

The recommended convention divides the space into three tiers:

| Range | Tier | Typical skills |
|-------|------|----------------|
| 0–49 | High confidence | Skills that search structured, bounded knowledge sources — local databases, domain-specific knowledge bases, FAQ indexes, entity lookups. These know quickly whether they have a relevant answer and are expected to be accurate when they claim the utterance. |
| 50–74 | Medium confidence | Skills that perform broader retrieval — web search, general knowledge queries — where answer quality depends on the query. They should handle the utterance if the topic is in scope but may decline when it is not. |
| 75–100 | Low confidence | General-purpose language-model chatbots and catch-all handlers that will attempt any utterance without domain restriction. These run last; they provide a response when nothing else can, not as a first choice. |

This three-tier mapping corresponds directly to the
`fallback_high` / `fallback_medium` / `fallback_low` multi-stage
pipeline example in §8.2. A deployment whose existing registrations
follow a different band convention maps them onto these tiers by
**band membership**, not by numeric rescale — see the appendix's
divergence catalogue for the mapping from the reference
implementation's bands.

A skill author uncertain which tier applies **SHOULD** register at
a higher number rather than a lower one. Pre-empting a more
precise handler with a low-confidence catch-all degrades response
quality silently.

**Default response skill.** A voice assistant SHOULD always
produce an answer rather than silent failure. Every deployment
SHOULD include a catch-all fallback skill — registered at the
highest priority number in the pool (e.g. `priority: 100`) — that
unconditionally returns `can_handle: true` and responds with a
graceful "I don't know how to answer that" message. When
`session.fallback_handlers` is set, this skill **SHOULD** be its
last entry, so it remains the last resort after all
higher-confidence handlers have declined. Without it, an utterance that no skill can handle
produces `ovos.intent.unmatched` with no user-facing response.

### 3.4 Session-scoped registration

Registration is session-scoped per **OVOS-INTENT-4 §11.1**: the
plugin keys each entry by `context.session.session_id` of the
registration Message. Skills registered under `"default"` are
available to all sessions. Skills registered under a specific
`session_id` extend the pool for that session only.

---

## 4. Session fields

This specification claims one optional session field per
**OVOS-SESSION-1 §2.2**.

| Field | Wire type | Owner |
|-------|-----------|-------|
| `fallback_handlers` | array of string | §4 (this spec) |

`session.fallback_handlers` is an ordered list of `skill_id`s
expressing a session-level preference for the fallback handler
order. When present, this list is the primary ordering input to
pool construction (§5). When absent, registered priority
determines order.

The list MAY be partial. Skills not listed but registered and
available are appended after the listed skills, sorted by
registered priority ascending.

Per-skill access control uses the existing
`session.blacklisted_skills` field (**OVOS-PIPELINE-1 §5.3**) — no
separate fallback-specific denylist is needed. To block all
fallback handling for a session, add the fallback stage(s) to
`session.blacklisted_pipelines`, or omit them from
`session.pipeline`.

---

## 5. Pool construction

On each `match` call the plugin constructs the **effective handler
pool**:

1. **Preference.** If `session.fallback_handlers` is present and
   non-empty, use it as the leading order. Append any registered
   skills not in the list, sorted by registered priority ascending.
   If absent, sort all registered skills by registered priority
   ascending.
2. **Stage range.** If this plugin instance is configured with a
   priority range (§8.2), retain only skills whose registered
   priority falls within that range.
3. **Availability.** Retain only skills present in the registry for
   the current `session_id` (including `"default"` registrations
   per §3.4).
4. **Policy.** Remove any `skill_id` present in
   `session.blacklisted_skills` (**OVOS-PIPELINE-1 §5.3**).

The result is the ordered effective pool. An empty pool causes the
plugin to return `None` immediately. No later stage adds what an
earlier stage removed.

---

## 6. Match contract

### 6.1 Per-skill query

When the effective pool is non-empty the plugin queries skills one
at a time in pool order. For each skill it sends:

`<skill_id>.fallback.ping`

using the **dotted addressed** form, derived via `.reply()` from
the inbound utterance Message (**OVOS-MSG-1 §5.2**), so that
`context.session` and routing metadata propagate automatically.

Payload:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of string | yes | The candidate utterance list the skill should evaluate. |
| `lang` | string | yes | The resolved BCP-47 language tag. |

The skill uses these to run its own evaluation logic and decide
whether it can produce a meaningful response. This is the point
at which the fallback skill parses the utterance — it may query a
knowledge base, run a classifier, call an LLM, or apply any other
internal logic. The reply carries only the decision:

The queried skill replies with:

`<skill_id>.fallback.pong`

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The responding skill's identity. MUST equal the topic prefix. |
| `can_handle` | bool | yes | Whether this skill is willing to handle the current utterance. |

The boolean's field name is protocol-specific: this spec and
OVOS-STOP-1 use `can_handle`, OVOS-CONVERSE-1's poll uses `result`,
and OVOS-COMMON-QUERY-1 uses `can_answer`. Each name is normative
only within its own protocol.

The plugin waits for each skill's reply before advancing to the
next. The plugin **MUST** bound each per-skill wait by a ceiling;
the **RECOMMENDED default is 0.5 s** (matching the analogous polls
of OVOS-CONVERSE-1 §4.2 and OVOS-STOP-1 §4.1), which a deployer MAY
raise or lower. Without a ceiling, one unresponsive skill stalls
the entire fallback stage — and with it the utterance — forever.
An absent or malformed pong (missing or non-boolean `can_handle`,
mismatched `skill_id`) **MUST** be treated as `can_handle: false`
and the skill skipped; this is uniform with the silence rules of
OVOS-CONVERSE-1 (§4.2 there) and OVOS-STOP-1 (§4.2 there).

**Broadcast-poll optimisation.** As an observably equivalent
alternative to the sequential per-skill query, the plugin MAY emit
a single broadcast poll to the whole effective pool, collect the
pongs, and then select the **first willing skill in pool order** —
not in response-arrival order. Because selection remains keyed on
pool order and silence/malformed pongs still count as
`can_handle: false`, the outcome is identical to the sequential
cycle while the waits overlap instead of accumulating.

**Bus-exchange exception.** The per-skill query cycle is a
documented exception to PIPELINE-1 §4.4's low-latency guidance,
justified because fallback stage(s) are positioned after all other
intent-matching stages (§8). No further stages are blocked during
the exchange, and the query terminates as soon as a willing skill
is found. This pattern follows the precedent of OVOS-CONVERSE-1's
per-skill converse poll.

### 6.2 Selection

The plugin selects the first skill in pool order whose
`can_handle` reply is `true`. If the pool is exhausted with no
willing skill the plugin returns `None`, and the pipeline emits
`ovos.intent.unmatched` (**PIPELINE-1 §9.3**).

### 6.3 Match shape

| Field | Value |
|-------|-------|
| `skill_id` | The selected skill's `skill_id`. |
| `intent_name` | `"fallback"` — reserved per PIPELINE-1 §7.3. |
| `lang` | The resolved BCP-47 language tag. |
| `utterance` | The first element of the input candidate list (PIPELINE-1 §4.1 fallback rule). |
| `slots` | Empty. |
| `updated_session` | Present if the plugin mutates session state. |

The `skill_id` targets the selected skill, not the fallback
plugin's own `pipeline_id`.

---

## 7. Dispatch and handler contract

The orchestrator dispatches `<skill_id>:fallback` per **PIPELINE-1
§7**, firing the standard handler-lifecycle trio. The dispatch
payload is the standard shape: `lang`, `utterance`, `slots`.

The selected skill's handler:

- **MAY** emit `ovos.utterance.speak` (**PIPELINE-1 §9.6**);
- **MAY** act silently;
- **MUST** complete within the handler lifecycle (**PIPELINE-1 §8**).

No fallback-specific response protocol is required beyond
subscribing to `<own_skill_id>:fallback`.

---

## 8. Pipeline positioning

### 8.1 General placement

Fallback stage(s) **SHOULD** be placed after all deterministic
intent-matching stages and after the persona stage (if present).
Utterances that reach a fallback stage were not claimed by any
earlier stage.

Every deployment **SHOULD** include a catch-all fallback skill
registered at the bottom of the priority pool (see §3.3) that
always returns `can_handle: true`. This ensures the user receives
a response to every utterance rather than silent `ovos.intent.unmatched`.

### 8.2 Multiple stages and priority interleaving

A deployment MAY load multiple fallback plugin instances at
different positions in `session.pipeline`. Each instance is
configured with a **priority range** — a `[min, max]` integer
interval — that restricts which registered skills it considers
(§5 step 2). This allows fallback skills to be interleaved with
other pipeline stages.

Example:

```
session.pipeline: [
  "stop_high",
  "converse",
  "intent_high",
  "fallback_high",    ← priority range [0, 49]
  "intent_medium",
  "fallback_medium",  ← priority range [50, 74]
  "persona",
  "fallback_low"      ← priority range [75, 100]
]
```

A skill registered at `priority: 10` is queried by `fallback_high`
before `intent_medium` runs. A skill registered at `priority: 80` is
queried by `fallback_low` only after both `intent_medium` and `persona`
have declined. A single-stage deployment sets no range restriction.

Within any stage, pool construction and session ordering (§5) work
identically regardless of how many stages are present.

---

## 9. Bus surface

| Topic | Form | Direction | Purpose |
|-------|------|-----------|---------|
| `ovos.fallback.register` | broadcast | skill → fallback | Register as a fallback handler (§3.1). |
| `ovos.fallback.deregister` | broadcast | skill → fallback | Deregister (§3.2). |
| `<skill_id>.fallback.ping` | dotted addressed | fallback → skill | Query: willing to handle this utterance? (§6.1). |
| `<skill_id>.fallback.pong` | dotted addressed | skill → fallback | Reply: willing or not (§6.1). |
| `<skill_id>:fallback` | dispatch | orchestrator → skill | Dispatch to the selected skill (§7). |

---

## 10. Conformance

### A fallback pipeline plugin **MUST**:

- expose a `match(utterances, lang, session) → Match | None`
  operation per PIPELINE-1 §4;
- maintain a session-scoped registry of skills with integer
  priorities per §3.4;
- subscribe to `ovos.fallback.register` and
  `ovos.fallback.deregister` (§3);
- reject any registration where payload `skill_id` ≠
  `context.skill_id` (§3.1);
- construct the effective handler pool per §5 on each match call;
- query skills via `<skill_id>.fallback.ping` / `.pong` — either
  sequentially in pool order or via the observably equivalent
  broadcast-poll optimisation (§6.1);
- bound each per-poll wait by a ceiling and treat an absent or
  malformed pong as `can_handle: false` (§6.1);
- select the first willing skill in pool order (§6.2);
- return a `Match` with `intent_name: "fallback"` targeting the
  selected skill (§6.3);
- return `None` when no skill in the pool is willing (§6.2).

### A fallback pipeline plugin **SHOULD**:

- use the recommended 0.5 s default per-poll ceiling unless the
  deployer configures otherwise (§6.1);
- when configured with a priority range, apply it as §5 step 2.

### A fallback pipeline plugin **MAY**:

- mutate session state via `Match.updated_session` (§6.3).

### A skill registered as a fallback handler **MUST**:

- emit `ovos.fallback.register` with its `skill_id` and `priority`
  before receiving fallback dispatches (§3.1);
- subscribe to `<own_skill_id>.fallback.ping` and reply with
  `<own_skill_id>.fallback.pong` (§6.1);
- subscribe to `<own_skill_id>:fallback` to receive dispatches (§7).

### A deployment **SHOULD**:

- position fallback stage(s) after deterministic intent-matching
  and persona stages in `session.pipeline` (§8.1).

---

## See also

- **OVOS-PIPELINE-1** — pipeline-plugin contract, Match shape,
  dispatch, reserved intent-name registry (§7.3).
- **OVOS-MSG-1** — envelope, derivations, and routing keys.
- **OVOS-SESSION-1** — session field registry and the omission
  rule.
- **OVOS-INTENT-4** — session-scoped registration model (§11).
- **OVOS-CONVERSE-1** — the dotted-addressed per-skill query
  pattern this specification follows.
- **OVOS-PERSONA-1** — the persona stage that precedes fallback.
