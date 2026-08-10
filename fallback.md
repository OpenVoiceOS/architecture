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
- **session fields** (§4) — the one session-resident field that
  controls pool ordering, and the reuse of the OVOS-PIPELINE-1
  denylists for access control;
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
| `skill_id` | string | yes | The skill's identity. MUST equal `context.skill_id` of this Message. |

Removes the skill from the registry. Unknown `skill_id` is a no-op.

Deregistration is scoped exactly like registration. The plugin
**MUST** key the removal by `context.session.session_id` of the
deregistration Message — never by a `session_id` carried in
`Message.data` — per **OVOS-INTENT-4 §11.3**. A deregistration
arriving under the default session removes the `"default"`-scoped
entry only; it does not remove entries registered under a specific
`session_id`.

The plugin **MUST NOT** honour a deregistration whose payload
`skill_id` differs from `context.skill_id`: without this check any
skill can evict any other skill from the registry. A mismatch is
malformed — log at WARN with both identifiers and the topic, and do
not act on it.

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
pipeline example in §8.2. A deployment using a different band
convention maps registrations onto these tiers by **band
membership** (which confidence class a number belongs to), not by
numeric rescale — the numbers label the classes; they are not a
continuous scale.

A skill author uncertain which tier applies is better off
registering at a higher number than a lower one. Pre-empting a more
precise handler with a low-confidence catch-all degrades response
quality silently.

**The catch-all handler.** The low-confidence tier is where a
deployment's catch-all handler belongs — the skill that answers
"I don't know how to answer that" when nothing else will. The
normative rule for that skill is §8.1; this section only says where
in the number space it sits. Note that a large priority number does
not by itself make a skill last: priority is an unbounded integer,
so any number can be undercut by a larger one. A catch-all is last
because of *placement* — it is the last entry of
`session.fallback_handlers`, or the only registration inside the
last stage's priority range (§8.2) — not because of the number it
chose.

### 3.4 Session-scoped registration

Registration is session-scoped per **OVOS-INTENT-4 §11.1**: the
plugin keys each entry by `context.session.session_id` of the
registration Message. Skills registered under `"default"` are
available to all sessions, because every session inherits the
`"default"` scope (**OVOS-INTENT-4 §11.2**). Skills registered
under a specific `session_id` extend the pool for that session
only.

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
   Two skills registered at the same priority **MUST** be ordered
   deterministically — the same registry and the same session
   always produce the same pool order. Ascending `skill_id` is the
   **RECOMMENDED** tie-break. Registration arrival order is not an
   acceptable tie-break: it makes the selected handler depend on
   process start order.
2. **Stage range.** If this plugin instance is configured with a
   priority range (§8.2), retain only skills whose registered
   priority falls within that range. Both bounds are **inclusive**:
   a range `[50, 74]` retains priority 50 and priority 74.

   Ranges configured across the loaded stages **SHOULD** partition
   the priority space — no gaps, no overlaps. A skill whose
   priority falls in no configured range is never queried by any
   stage; the deployment is misconfigured, and a plugin that can
   observe the gap **SHOULD** log it at WARN with the `skill_id`
   and the priority.

   The range filter applies after the preference order of step 1.
   A skill named in `session.fallback_handlers` is therefore still
   dropped by the stage whose range excludes it — the preference
   list orders the pool, it does not admit a skill into a stage.
   A skill preferred by the session but out of every stage's range
   is never queried at all.
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

using the **dotted addressed** form (**OVOS-MSG-1 §2.1.1**),
derived from the inbound utterance Message through the reply
derivation (**OVOS-MSG-1 §5.2**). The derivation is the reply one
because the queried skill always answers: a ping that arrives
response-ready lets the skill produce its pong by the same
derivation, and the pong then reaches the plugin wherever either
side runs. Carrying `context.session` forward is a property every
derivation has; it is not what selects `reply` here.

Payload:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of string | yes | The candidate utterance list the skill should evaluate. |
| `lang` | string | yes | The resolved BCP-47 language tag. |

The skill uses these to run its own evaluation logic and decide
whether it can produce a meaningful response. This is the point
at which the fallback skill parses the utterance — it may query a
knowledge base, run a classifier, call an LLM, or apply any other
internal logic. The reply carries only the decision.

The queried skill replies with:

`<skill_id>.fallback.pong`

derived from the ping through the reply derivation (**OVOS-MSG-1
§5.2**), so that the pong reaches the plugin wherever the skill
runs.

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The responding skill's identity. MUST equal the topic prefix. |
| `can_handle` | bool | yes | Whether this skill is willing to handle the current utterance. |
| `utterance` | string | yes | Echo of the utterance evaluated — the first element of the ping's `utterances`. |

**Round correlation.** The plugin keys poll state by
`session_id`, read from `context.session`. A pong whose `utterance`
or whose session does not match the poll in flight **MUST** be
discarded: a stale or replayed pong otherwise decides a round it
never evaluated. The round-trip is one exchange per
`(skill_id, utterance)` pair. Because at most one fallback poll is
in flight per session (the session lock and round model of this
section), the utterance itself uniquely identifies the round — no
opaque poll id is needed, unlike OVOS-COMMON-QUERY-1 §6.4's
`query_id`, which serves overlapping contests.

The `utterance` field is REQUIRED going forward. A pong that omits
it is a legacy producer; the plugin **SHOULD** accept such a pong,
correlating it by session and by the `skill_id` it is still
waiting on. A pong that carries `utterance` and does not match
**MUST** be discarded — tolerance covers absence, never
disagreement.

The boolean's field name is protocol-specific: this spec and
OVOS-STOP-1 use `can_handle`, OVOS-CONVERSE-1's poll uses `result`,
and OVOS-COMMON-QUERY-1 uses `can_answer`. Each name is normative
only within its own protocol.

The plugin waits for each skill's reply before advancing to the
next. The plugin **MUST** bound each poll wait by a ceiling.
Without a ceiling, one unresponsive skill stalls the entire
fallback stage — and with it the utterance — forever. An absent or
malformed pong (missing or non-boolean `can_handle`, mismatched
`skill_id`, mismatched `utterance`) **MUST** be treated as
`can_handle: false` and the skill skipped; this is uniform with the
silence rules of OVOS-CONVERSE-1 (§4.2 there) and OVOS-STOP-1
(§4.2 there).

**The poll is the decision.** Unlike OVOS-COMMON-QUERY-1, whose
ping only filters plausible answerers before a separate answer
round, the fallback ping is the whole contest: the pong is the
claim, and there is no second round in which a slow evaluator can
catch up. A skill that cannot answer the ping in time is not
delayed, it is skipped. This follows OVOS-CONVERSE-1 §4.2, where
the poll is likewise the decision point.

**Ceiling calibration.** The **RECOMMENDED default ceiling is
0.5 s**, matching the analogous polls of OVOS-CONVERSE-1 §4.2 and
OVOS-STOP-1 §4.1 — with the caveat that OVOS-STOP-1 §4.1 also caps
its ceiling at 1 s, which this specification does not. That default
is calibrated for skills whose evaluation is local: a database
lookup, a classifier, a vocabulary test. It is not a budget for a
model-backed skill. A deployment that runs a model-backed fallback
stage — an LLM chatbot, a remote question-answering service —
**MUST** configure that stage's ceiling above the stage's actual
evaluation latency. Leaving the default in place silently converts
every such skill into a non-responder, and the utterance falls
through to a lower-confidence handler that answered faster.

**Stage collection ceiling.** The worst-case time a fallback stage
spends inside `match` is bounded by:

- **sequential form** — `pool_size × per-poll ceiling`, since each
  wait runs to the ceiling before the next begins;
- **broadcast form** — one poll window, independent of pool size.

Both are the stage's *collection ceiling*. OVOS-PIPELINE-1 §4.4
lets the orchestrator bound each `match` by a timeout and skip a
plugin that exceeds it, so a deployment **MUST** set the fallback
stage's match-timeout bound at or above that stage's collection
ceiling, exactly as OVOS-COMMON-QUERY-1 §2.1 requires for the
common-query collection window. A shorter bound kills the stage
mid-poll on every utterance it handles, and the failure is silent:
the stage simply never claims. The broadcast form is the
straightforward way to keep the ceiling constant as the pool grows.

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

**Activation.** `fallback` is a reserved intent_name, but it is not
one of the reserved names whose `session.active_handlers` push is
suppressed — the per-row rule is in the OVOS-PIPELINE-1 §7.3
registry, and the fallback row does push. A fallback dispatch is a
fresh activation, not the continuation of an already-active skill,
so the dispatch pushes the selected `skill_id` onto
`session.active_handlers` per OVOS-PIPELINE-1 §7.1. This is what
makes a running fallback skill stoppable through OVOS-STOP-1, and
what lets a conversational fallback — a language-model chatbot, for
instance — take the next utterance through OVOS-CONVERSE-1 without
a second registration.

---

## 8. Pipeline positioning

### 8.1 General placement

Fallback stage(s) **SHOULD** be placed after all deterministic
intent-matching stages. Only the low-confidence, catch-all fallback
stage **SHOULD** additionally follow the persona stage (if present):
it is the last-resort tier (§3.3), and a persona claims everything
that reaches it (OVOS-PERSONA-1 §7.2), so anything placed behind
persona is unreachable while a persona is active — the catch-all
belongs there regardless. Higher-confidence fallback stages
(`fallback_high`, `fallback_medium`) **MAY** precede the persona
stage, interleaved with intent-matching stages by priority range, per
§8.2. Utterances that reach a fallback stage were not claimed by any
earlier stage.

**Default response skill.** A voice assistant should always produce
an answer rather than fail silently. Every deployment **SHOULD**
include a catch-all fallback skill that unconditionally returns
`can_handle: true` and responds with a graceful "I don't know how
to answer that" message. That skill **SHOULD** be reachable only
after every other fallback handler has declined — as the last entry
of `session.fallback_handlers` when that list is set, and inside
the range of the last fallback stage when stages are ranged (§8.2).
§3.3 places it in the low-confidence tier. Without it, an utterance
that no skill can handle produces `ovos.intent.unmatched`
(**PIPELINE-1 §9.3**) and no user-facing response.

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
  "common_query",
  "persona",
  "fallback_low"      ← priority range [75, 100]
]
```

A skill registered at `priority: 10` is queried by `fallback_high`
before `intent_medium` runs. A skill registered at `priority: 80` is
queried by `fallback_low` only after `intent_medium`,
`fallback_medium`, `common_query`, and `persona` have all declined.
`fallback_high` and `fallback_medium` interleave with the
intent-matching stages ahead of `persona`; only `fallback_low`, the
catch-all tier, follows it — per §8.1 and OVOS-PERSONA-1 §10, which
places persona after `common_query` and states the same ordering
from its side. A single-stage deployment sets no range restriction.

The configured ranges are inclusive at both bounds and **SHOULD**
partition the priority space (§5 step 2). Because priority is an
unbounded integer, the last stage's range **SHOULD** be open above
— `[75, ∞)` rather than `[75, 100]` — so that a skill registering
above the convention is still queried somewhere.

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
  `context.skill_id` (§3.1), and any deregistration with the same
  mismatch (§3.2);
- key registration and deregistration by
  `context.session.session_id`, never by a `session_id` in
  `Message.data` (§3.2, §3.4);
- construct the effective handler pool per §5 on each match call,
  ordering equal priorities deterministically (§5 step 1);
- query skills via `<skill_id>.fallback.ping` / `.pong` — either
  sequentially in pool order or via the observably equivalent
  broadcast-poll optimisation (§6.1);
- bound each poll wait by a ceiling and treat an absent or
  malformed pong as `can_handle: false` (§6.1);
- discard a pong whose `utterance` or session does not match the
  poll in flight (§6.1);
- select the first willing skill in pool order (§6.2);
- return a `Match` with `intent_name: "fallback"` targeting the
  selected skill (§6.3);
- return `None` when no skill in the pool is willing (§6.2).

### A fallback pipeline plugin **SHOULD**:

- use the recommended 0.5 s default poll ceiling unless the
  deployer configures otherwise (§6.1);
- when configured with a priority range, apply it as §5 step 2, and
  log at WARN a registered skill that falls in no range (§5);
- tolerate a pong that omits `utterance` as a legacy producer
  (§6.1);

### A fallback pipeline plugin **MAY**:

- mutate session state via `Match.updated_session` (§6.3).

### A skill registered as a fallback handler **MUST**:

- emit `ovos.fallback.register` with its `skill_id` and `priority`
  before receiving fallback dispatches (§3.1);
- subscribe to `<own_skill_id>.fallback.ping` and reply with
  `<own_skill_id>.fallback.pong`, derived through the reply
  derivation and echoing the evaluated `utterance` (§6.1);
- subscribe to `<own_skill_id>:fallback` to receive dispatches (§7).

### A deployment **SHOULD**:

- position fallback stage(s) after deterministic intent-matching
  and persona stages in `session.pipeline` (§8.1);
- include a catch-all fallback skill, reachable only after every
  other fallback handler has declined (§8.1);
- configure priority ranges that partition the priority space, with
  the last stage open above (§8.2).

### A deployment **MUST**:

- set each fallback stage's match-timeout bound at or above that
  stage's collection ceiling (§6.1, OVOS-PIPELINE-1 §4.4);
- raise the poll ceiling of a model-backed fallback stage above
  that stage's evaluation latency (§6.1).

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
