# Common Query Pipeline Plugin Specification

**Spec ID:** OVOS-COMMON-QUERY-1 · **Version:** 2 · **Status:** Draft

This specification defines the **common query pipeline plugin** — a
pipeline plugin that answers factual questions by holding a timed
contest among skills. During its `match` phase it broadcasts the
question, collects full answers from the skills that claim they can
answer, ranks them, and — if any answer clears a confidence
threshold — returns a `Match` carrying the winning answer for its
own handler to speak. When no answer clears the threshold, `match`
returns `None` and the pipeline continues to the next stage,
including fallback.

It builds on OVOS-MSG-1 (envelope, `reply` derivation, session
carrier), OVOS-PIPELINE-1 (pipeline-plugin contract, `Match` shape,
dispatch topic shape, handler-lifecycle trio, `Match.updated_session`,
reserved intent_name registry, §4.4 blocking-match allowance), and
OVOS-SESSION-1 (session field registry, omission rule).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **common query plugin role** (§2) — a pipeline plugin whose
  `match` blocks while it runs a multi-skill contest;
- the **reserved intent_name `common_query`** (§3);
- the **question gate** (§4) — an optional pre-filter that rejects
  non-question utterances before any broadcast;
- the **early-start optimisation** (§5) — how the plugin overlaps
  its contest with upstream pipeline stages;
- the **wants-to-answer poll** (§6) — the fast ping/pong broadcast
  that filters skills down to plausible answerers;
- the **answer collection** (§7) — full-answer gathering;
- the **filtering and selection** (§8) — confidence filtering,
  denylist, opt-in fast-win, and ranking, applied against the live
  session;
- the **match construction** (§9) — the `Match`, or `None`;
- the **plugin handler** (§10) — the trivial handler that speaks the
  selected answer;
- the **skill-side protocol** (§11);
- **pipeline positioning** (§12);
- the **bus surface** (§13);
- **conformance** (§14);
- **tunable defaults** (Appendix A) and **confidence-range guidance
  for skill authors** (Appendix B).

This specification does **not** define:

- **vocabulary file format or question-classification algorithm** —
  the gate MAY use any method; only the observable behaviour
  (accept / reject) is normative.
- **skill-side answer generation** — what a skill does internally to
  produce an answer is the skill's business. The spec fixes only the
  bus contract by which the skill reports its answer.
- **the framework decorator or base class** a skill author uses to
  participate — these are conveniences, not normative. Any component
  that honours the bus contract in §11 is a valid common-query skill.
- **streaming answer delivery** — the plugin collects complete answer
  strings before selecting; incremental assembly is out of scope.

---

## 2. The common query plugin role

The common query plugin is a pipeline plugin (PIPELINE-1 §3) that
**bundles its own handler**. Its two roles are structurally separate:

- **Matcher role** (§6–§9): during `match`, the plugin runs the full
  contest — ping/pong broadcast, parallel answer collection,
  filtering, and ranking. If an answer wins, `match` returns a
  `Match` pointing at the plugin itself, carrying the answer in
  `slots`. If no answer wins, `match` returns `None` and the
  orchestrator proceeds to the next pipeline stage, including
  fallback.
- **Handler role** (§10): on receiving `<pipeline_id>:common_query`,
  the plugin reads the selected answer from the dispatch payload and
  speaks it.

Because `match` returns `None` when no good answer is found, common
query **never blocks fallback**. This is its defining difference from
a plugin that claims an utterance speculatively and only later
discovers it cannot satisfy it.

### 2.1 Blocking match — a deliberate exception to latency discipline

OVOS-PIPELINE-1 §4.4 permits `match` to block on bus I/O but
**SHOULD**s plugins to return quickly and defer expensive work to the
handler, since match-phase latency is response latency. Common query
is a deliberate, documented exception: the answer **is** the claim
decision. The plugin cannot return a `Match` and defer collection to
the handler, because whether it claims at all depends on whether any
skill produces an answer above threshold (§9). The expensive work and
the routing decision are the same act, so it must happen in `match`.

Two consequences a deployer **MUST** accept:

- **Bound interaction.** OVOS-PIPELINE-1 §4.4 lets the orchestrator
  bound each `match` by a timeout and skip a plugin that exceeds it.
  A deployment **MUST** set common query's match-timeout bound at or
  above its collection-window ceiling (§7.2), or the stage will be
  skipped mid-contest. The early-start optimisation (§5) is the
  intended way to keep the observed `match` duration low without
  shrinking the contest.
- **Positioning.** Latency is bounded by the slowest claiming skill,
  up to the ceiling; §12 positions the stage to contain that cost.

Every other field of the pipeline-plugin contract applies unchanged:
the plugin is loaded and iterated per `session.pipeline` ordering,
subject to first-match-wins iteration (PIPELINE-1 §6.2) and denylist
filtering (PIPELINE-1 §5.2–§5.4).

### 2.2 Pipeline identity

The plugin instance has one `pipeline_id` (OVOS-PIPELINE-1 §3),
typically `common_query`. A deployment MAY reference it from several
`session.pipeline` entries under different match configurations;
those entries are matcher references, not additional actors.

---

## 3. Reserved intent_name

The intent_name `common_query` is **reserved** in the
OVOS-PIPELINE-1 §7.3 registry.

| Reserved intent_name | Dispatch topic | Meaning |
|----------------------|---------------|---------|
| `common_query` | `<pipeline_id>:common_query` | The plugin's own handler: speak the answer selected during `match` (§10). |

This intent_name is **not** registered via OVOS-INTENT-4. A
registration naming `common_query` via `ovos.intent.register.*` is
malformed per PIPELINE-1 §7.3.

Full-answer requests during `match` use the static topic
`ovos.common_query.request`, target named in the payload (§7). They are sent by the plugin,
not by the orchestrator, and are not dispatches — the colon form is
reserved for the PIPELINE-1 §7 dispatch shape (OVOS-MSG-1 §2.1.1).

---

## 4. The question gate

The question gate is a **cost-optimisation pre-filter**, not the
primary quality mechanism. The confidence filter (§8) is the primary
quality gate: even with no gate, a non-question utterance produces no
answer above threshold, `match` returns `None`, and the pipeline
continues. The gate exists only to skip the broadcast cost — the
ping/pong round-trip and parallel skill invocations — for utterances
that obviously cannot produce a useful answer.

A plugin **SHOULD** apply a gate — a sentence-type classifier or any
other cheap short-circuit — to avoid running the contest for
utterances that are not question-like. Weather requests, music
commands, timers, and plain statements have no business reaching a
knowledge skill.

A deployment that omits the gate is still conformant — the confidence
filter guarantees correctness either way — but pays the broadcast cost
on every utterance. A gate trades that cost against the risk of false
negatives (a genuine question wrongly rejected), so §4.2 biases the
gate toward acceptance when in doubt.

### 4.1 Gate semantics

When configured, the gate is a binary pre-filter:

- **Accept** — the utterance is plausibly a factual question. Proceed
  to the poll (§6).
- **Reject** — the utterance is clearly an action command or
  otherwise not a factual question. `match` returns `None` without
  broadcasting.

The gate **MUST NOT** be used as a confidence scorer or ranking
layer; scoring belongs to the responding skills (§7–§8). The gate MAY
use any combination of classifiers, vocabulary heuristics, or
length thresholds. A deployment MAY skip the plugin's own gate and
rely on an upstream classifier.

### 4.2 Gate conformance

The gate **SHOULD** accept utterances that express a factual question
and **SHOULD NOT** accept unambiguous action commands with no
information intent.

The question/command boundary is fuzzy, so neither side of the gate
can be stated as a testable MUST over the whole utterance space. The
examples below are **informative** — they illustrate the intended bias,
they do not enumerate a conformance set:

| Bias | Informative examples |
|------|----------------------|
| accept | "what is the capital of France", "who invented electricity", "tell me about France" |
| reject | "play music", "set a timer", "turn off the lights" |

Over-acceptance wastes a round-trip; under-acceptance silently fails
the user. When in doubt, accept.

---

## 5. Early-start optimisation

Common query is a slow stage, but most of its latency can be hidden.
The plugin **MAY** subscribe to the utterance-entry topic
`ovos.utterance.handle` (OVOS-PIPELINE-1 §9.1) — the message the
orchestrator consumes to begin a new utterance, before pipeline
iteration starts. When this subscription is active, the plugin begins
the contest (gate → poll → answer collection) immediately, **in
parallel** with the upstream pipeline stages (stop, converse, intent
matchers). By the time the orchestrator calls `match` for the common
query stage, the raw responses MAY already be collected.

**Language carve-out.** OVOS-PIPELINE-1 §9.1 forbids a plugin to
re-derive the content language independently: the orchestrator
resolves it once and passes the resolved tag to every `match` call.
An early start runs *before* that resolution exists, so a plugin that
early-starts **MUST** treat the tag it starts under as **provisional**
— derived for speculative work only, never published. Specifically it
**MUST NOT** emit the provisional tag in any `Match.lang`, and it
**MUST** discard the whole cached contest when the provisional tag is
not equal to the `lang` argument the orchestrator later passes to
`match` (§5.1). The results of an early start are void unless the two
tags are equal. This is the only sanctioned pre-resolution language
derivation in this specification, and it is sanctioned because its
output can never reach the wire un-revalidated.

### 5.1 What is cached, and what is not

The early-start cache holds only the **raw skill responses** (§7) and
the **utterance** they were collected for. It does **not** hold a
selected answer. All filtering and selection (§8) is performed at
`match` time against the **live session** the orchestrator passes in —
never against the session snapshot the early start began with.

If the `lang` argument passed to `match` differs from the provisional
tag the early start collected under (§5), the cached responses
**MUST** be discarded and the contest re-run.

### 5.2 Cache keying and lifetime

The cache **MUST** be keyed by the pair `(session_id, utterance)`,
with `session_id` read from `context.session`. A cache entry is
consumed and evicted when `match` reads it. An entry is evicted
unconditionally when a new utterance arrives in the same session.

**Utterance is the exact-match key.** A cache entry **MUST NOT** be
returned for any utterance other than the exact string it was
collected for.

**Which candidate is broadcast (n-best).** `match` receives a list of
candidate utterances, while the ping, the request, and the cache key
carry a single string. The plugin **MUST** run the contest for the
**first** candidate in the list and **MUST** use that same string as
`Match.utterance`. This is the same rule PIPELINE-1 §4.1 states for a
plugin that does not track which candidate won, so a common query
`Match` is always consistent with the payload the orchestrator
forwards. A plugin **MAY** internally consider other candidates, but
the broadcast string, the cache key, and `Match.utterance` **MUST**
remain the first candidate.

---

## 6. The wants-to-answer poll

When the gate accepts (or no gate is configured), the plugin runs a
**fast broadcast poll** to filter the skill set down to those that
plausibly can answer. The poll exists to avoid invoking the
**expensive** full-answer path (§7) — which may hit the network or a
database — on skills that have no relevant knowledge. It is a cheap
local filter gating an expensive operation; that is its entire
justification.

### 6.1 Ping

The plugin broadcasts on `ovos.common_query.ping`:

```json
{
  "utterance": "what is the capital of France"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterance` | string | yes | The utterance being broadcast (§5.2, first candidate). |

The contest is identified by `context.utterance_id` (§6.4) — a
`context` field, not a payload field, carried onto the ping by
ordinary `reply` derivation and onto every pong and response the
same way.

The language the plugin runs the contest in is the `lang` **argument**
the orchestrator passed to `match` (PIPELINE-1 §9.1), or — during an
early start — the provisional tag of §5, which is revalidated against
that argument before anything is published. The plugin **MUST NOT**
re-derive the language from `context.session` when a `lang` argument
is available.

The broadcast carries no `destination`; any subscribed skill MAY
respond. The session rides in `context.session` per OVOS-MSG-1 §4.

### 6.2 Pong

A skill that believes it can answer responds on
`ovos.common_query.pong`, derived via `reply` (OVOS-MSG-1 §5):

```json
{
  "utterance": "what is the capital of France",
  "skill_id": "wiki.test",
  "can_answer": true,
  "latency_ms": 800
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterance` | string | yes | Echo of the ping's utterance. |
| `skill_id` | string | yes | The responding skill's identifier. |
| `can_answer` | boolean | yes | Whether the skill claims it can answer. |
| `latency_ms` | number | no | Expected time in milliseconds to produce a full answer. A **hint** for sizing the collection window (§7.2), never a commitment or an extension of any bound. |

The boolean's field name is protocol-specific: this spec's poll uses
`can_answer`, while the analogous polls of OVOS-FALLBACK-1 /
OVOS-STOP-1 use `can_handle` and OVOS-CONVERSE-1 uses `result`. Each
name is normative only within its own protocol.

**The pong check is a fast local decision.** A skill **MUST** base
`can_answer` on local, synchronous operations only — keyword
matching, vocabulary lookup, cached knowledge — and **MUST NOT**
perform network requests, database queries, or other blocking I/O
during the pong phase. The full answer comes later (§7), where I/O is
expected.

The plugin **MUST** enforce a poll-window ceiling and stop waiting
when it elapses; the responses it has by then are the claimants.
Skills **SHOULD** respond within the deployer-configured pong bound
(Appendix A). A skill that cannot answer **SHOULD** stay silent;
sending `can_answer: false` is permitted but pointless, since the
window closes on timeout or sufficiency regardless. A skill that does
not respond in time is treated as not claiming.

### 6.3 Poll window and early close

The plugin **MUST** enforce a maximum poll window (Appendix A) and
**MUST** stop waiting when it elapses. The plugin **SHOULD** close it
early once enough claimants are identified — a deployment MAY proceed
as soon as one claims.

### 6.4 The contest identifier

The contest identifier is `context.utterance_id` (OVOS-MSG-1 §4.2),
stamped once at the lifecycle source and carried onto every derived
Message. `session_id` already separates peers and conversations —
it does not separate **two contests inside one session**, which is
exactly what a repeated question or a shared default session
produces; two lifecycles carry two `utterance_id`s, and that is the
whole discrimination. This specification adds **no** correlation
field of its own.

- Nothing is echoed and nothing is generated per protocol: the ping,
  the pong, the request and the response are all `reply`-derived
  (§6.1, §7.1), and MSG-1 §4.1/§4.2 propagation carries
  `utterance_id` onto each of them with no skill-side action.
- For an **out-of-band** query (§12) the requester is the lifecycle
  source and stamps `utterance_id`; a plugin receiving one without it
  sits at lifecycle entry and **MUST** stamp a fresh one
  (MSG-1 §4.2) before deriving the contest's Messages.
- The plugin **MUST** discard any pong or response whose
  `context.utterance_id` does not equal the active contest's — a
  Message that cannot prove which contest it belongs to never
  decides one.

Contest state is keyed by `session_id` from `context.session`
alongside `utterance_id`; a pong or response whose session does not
match the active contest **MUST** be discarded.

## 7. Answer collection

After the poll window closes, the plugin requests full answers from
all claiming skills **in parallel**.

### 7.1 Full-answer request and response

The plugin sends `ovos.common_query.request` (broadcast, static
topic) once per claiming skill, naming the target in the payload:

```json
{
  "utterance": "what is the capital of France",
  "skill_id": "wiki.test"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterance` | string | yes | The utterance to answer. |
| `skill_id` | string | yes | The skill being asked. Every common-query skill subscribes to the one topic and answers only when this names it — one payload equality check. Addressing is payload; the routing pair is owned by the `reply` swap (OVOS-MSG-1 §5.2) and never carries it. |

The language is the `lang` argument passed to `match` (§6.1), not a
value re-derived from the session. These are direct
plugin-to-skill messages: the orchestrator does not participate, does
not emit the handler-lifecycle trio for them, and skills **MUST NOT**
emit lifecycle signals in response.

Each skill emits its result on `ovos.common_query.response`
(static topic, derived via `reply` per OVOS-MSG-1 §5):

```json
{
  "utterance": "what is the capital of France",
  "skill_id": "wiki.test",
  "answer": "Paris is the capital of France.",
  "conf": 0.85
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterance` | string | yes | Echo of the request's utterance. |
| `skill_id` | string | yes | The responding skill's identifier. **MUST** equal the request's target (§7.1.1). |
| `answer` | string | conditional | The natural-language answer. **MUST** be present when the skill has one. |
| `conf` | number | conditional | Self-reported confidence in `[0, 1]`. **MUST** be present when `answer` is present (Appendix B). |

A skill that cannot produce an answer after all **MUST** still
respond, with no `answer` field, so early termination can fire.
Responses whose session does not match the active collection, or
whose `context.utterance_id` does not match it (§6.4), **MUST** be
discarded.

#### 7.1.1 Payload identity, duplicates, and malformed responses

Identity is payload, verified against the contest's own state:

- **Binding.** The plugin **MUST** discard a response whose payload
  `skill_id` does not name a skill it requested an answer from in
  this contest (§7.1). Without this check every downstream decision
  that consumes `skill_id` — the denylist (§8 step 2),
  deduplication, tie-breaking (§8.1), the answering-skill slot (§9)
  — acts on an identity the responder chose freely. The topics of
  this family are static strings; no identity is ever recovered by
  parsing a topic.

### 7.2 Collection window

The collection window has two values (Appendix A): an **initial
window** and a **hard ceiling**. The plugin **MUST** enforce the
ceiling. The window closes at the earliest of: every claimant has
responded (early termination, which the plugin **MUST** support), the
current window expiring with no outstanding claimant, or the ceiling.

**Extension trigger.** The window starts at the initial value and
extends toward the ceiling **only while at least one claimant is still
outstanding** — a skill that sent a claiming pong and has sent no
response yet. When the initial window expires with no outstanding
claimant, the plugin **MUST** select immediately at that point; it
**MUST NOT** wait out the remaining time to the ceiling. When
claimants are still outstanding, the plugin **MAY** keep waiting, up
to and never beyond the ceiling. A claimant that has not responded by
the ceiling is treated as declining, and the ceiling is the absolute
bound on the stage's contribution to response latency.

**Sizing from `latency_ms`.** When `latency_ms` values are available
from pongs (§6.2), the plugin **SHOULD** size the initial window to
the maximum `latency_ms` across claimants, clamped to the ceiling;
otherwise it **SHOULD** use the fixed initial window. `latency_ms` is a **hint**: it never raises the ceiling, an
implausible value (negative, non-numeric, above the ceiling) falls
back to the fixed initial window, and a plugin **MAY** ignore it
entirely. A skill cannot inflate the stage's budget; the worst it can
do is fail to be waited for.

---

## 8. Filtering and selection

Filtering and selection run at `match` time against the **live
session** (§5.1), in order:

1. **Minimum self-confidence.** Discard responses whose `conf` is
   below the deployer-defined threshold (Appendix A).
2. **Denylist.** Discard responses whose `skill_id` appears in the
   live `session.blacklisted_skills` (PIPELINE-1 §5.3). **This step is
   load-bearing, not defence in depth.** PIPELINE-1 §5.3 makes the
   orchestrator a backstop by checking `Match.skill_id` against the
   denylist after a plugin returns; for common query that check can
   never fire, because `Match.skill_id` is the plugin's own
   `pipeline_id` (§9), never the answering skill's. A common query
   plugin that skips this step silently speaks answers from
   blacklisted skills and nothing downstream will catch it.
3. **Fast-win (deployment-opt-in, default off).** A deployment MAY
   enable a fast-win rule: when enabled, if any surviving response
   carries `conf ≥` the fast-win threshold (Appendix A), the plugin
   MAY stop waiting immediately and select it, and the check MAY
   fire during collection (§7.2), short-circuiting the window. The
   rule is **off by default**: `conf` is self-reported and skills
   share no calibrated confidence scale (Appendix B), so selecting
   the first response to cross a threshold turns answer selection
   into a nondeterministic latency race — the fastest confident
   skill wins, not the best one. Absent explicit deployer opt-in,
   the plugin **SHOULD** wait for all claimants whose reported
   `latency_ms` is within the ceiling before selecting.
4. **Selection.** Select the highest-`conf` survivor. Ties MAY be
   broken by any deployer-defined heuristic; the algorithm is not
   normative. When a reranker is configured, the plugin **SHOULD**
   pass all survivors to it and use its ranking in place of raw
   `conf` ordering; the reranker interface is a deployment concern.

If no response survives, the contest has no winner — `match` returns
`None` (§9).

## 9. Match construction

After selection (§8):

- If **no response survived**, the plugin **MUST** return `None`. The
  orchestrator proceeds to the next pipeline stage, including
  fallback. A contest with no winner is an expected outcome, not an
  error.
- If **an answer won**, the plugin **MUST** return a `Match` with:
  - `skill_id`: the plugin's own `pipeline_id`
  - `intent_name`: `"common_query"` (reserved, §3)
  - `lang`: the `lang` **argument** the orchestrator passed to `match`
    — the contest was run in that language (§6.1), so the plugin
    reports it back verbatim. The plugin **MUST NOT** report a value
    it derived itself, and in particular **MUST NOT** report an
    early-start provisional tag (§5).
  - `utterance`: the first candidate from the input list, which is the
    string the contest was run for (§5.2)
  - `slots`: `{ "answer": "<the selected answer string>" }` — the
    only field the handler needs (§10)
  - `updated_session`: omitted

`updated_session` is **omitted**, not set to a copy of the inbound
session: PIPELINE-1 §4.1 defines an absent `updated_session` as
"carry the inbound session unchanged", which is precisely this
plugin's intent, and an echoed snapshot would claim a mutation the
plugin did not make.

The plugin **MUST NOT** mutate the session: common query does not
activate handlers, change `persona_id`, or modify any session field —
so it never emits an `updated_session` at all.

---

## 10. The plugin handler

When the orchestrator dispatches `<pipeline_id>:common_query`, the
handler runs and fires the handler-lifecycle trio per PIPELINE-1 §8
(`ovos.intent.handler.start`, `.complete`, `.error`).

The handler is intentionally trivial — all contest work completed
during `match` (§6–§8). It:

1. Reads `answer` from `slots` in the dispatch payload.
2. Speaks it via `ovos.utterance.speak` per OVOS-PIPELINE-1.
3. Emits `ovos.intent.handler.complete`.

The handler **MUST NOT** re-dispatch to skills or perform additional
collection. `ovos.intent.handler.error` is reserved for crashes and
unrecoverable handler failures.

---

## 11. Skill-side protocol

A skill participates by handling two topics (see §13 for the full bus
surface):

1. On `ovos.common_query.ping`, perform a **fast local check** for a
   likely answer. If yes, respond on `ovos.common_query.pong` with
   `can_answer: true`, the echoed `utterance`,
   its own `skill_id`, and optionally `latency_ms` — deriving the
   pong via `reply` so `context.utterance_id` rides along (§6.4). If no, stay
   silent.
2. On `ovos.common_query.request` naming it in `data.skill_id`,
   produce the best answer — network calls, DB queries, and full
   generation are appropriate here — and
   emit it on `ovos.common_query.response` (via `reply`,
   OVOS-MSG-1 §5) with the echoed `utterance`,
   its own `skill_id`, `answer`, and `conf` (§7.1.1). A request
   naming a different skill is not addressed to it and **MUST** be
   ignored.
   If no answer can be produced, emit the response with no `answer`
   field so early termination can fire.
3. The skill **MUST NOT** call `ovos.utterance.speak` from its
   `common_query` handler. Speaking is the plugin's responsibility
   (§10).

---

## 12. Pipeline positioning

Common query is a slow stage. A deployment **SHOULD** place it after
all intent-matching stages and before the fallback stage(s): intent
matchers are tried first, and fallback still runs if common query
finds no answer. When a persona catch-all (OVOS-PERSONA-1 §10) is also
present, common query precedes it, so deterministic question-answering
is preferred over a persona's generated reply.

```
session.pipeline: [
  "stop_high",
  "converse",
  "skill_high",
  "skill_medium",
  "common_query",
  "fallback_medium",
  "fallback_low"
]
```

With early start enabled (§5), the contest begins as the utterance
arrives, so its wall-clock cost is largely amortised against the
upstream stages by the time the orchestrator reaches it. Without
early start, the stage blocks for the full collection window.

---

## 13. Bus surface

| Topic | Direction | Purpose | Defined in |
|-------|-----------|---------|------------|
| `ovos.common_query.ping` | plugin → all skills | Wants-to-answer poll | §6.1 |
| `ovos.common_query.pong` | skill → plugin | Claim, via `reply` | §6.2 |
| `ovos.common_query.request` | plugin → claiming skill (target in `data.skill_id`) | Full-answer request (during match) | §7.1 |
| `ovos.common_query.response` | claiming skill → plugin | Full answer or decline, via `reply` | §7.1, §11 |
| `<pipeline_id>:common_query` | orchestrator → plugin | Handler dispatch (reserved intent_name) | §3, §10 |

The one colon-form topic (`<pipeline_id>:common_query`) is the
orchestrator's dispatch and follows the PIPELINE-1 §7 dispatch shape.
Dotted-form topics (`ovos.common_query.request`,
`ovos.common_query.response`) are plugin- and skill-emitted
non-dispatch messages per MSG-1 §2.1.1; the target and responder
identities travel in `data.skill_id` (§7.1.1) — never in the topic.
`ovos.common_query.ping` is a broadcast. Pong and
answer responses are both derived via `reply` (OVOS-MSG-1 §5). Every
poll/response message is correlated by `context.utterance_id`
(§6.4) and echoes the `utterance`.

---

## 14. Conformance

### A common query pipeline plugin **MUST**:

- expose a blocking `match(utterances, lang, session) → Match | None`
  per PIPELINE-1 §4 (§2.1);
- broadcast `ovos.common_query.ping` and collect
  `ovos.common_query.pong` within a bounded poll window (§6.3);
- correlate every contest by `context.utterance_id` (MSG-1 §4.2),
  stamping a fresh one only at lifecycle entry for an out-of-band
  query that arrived without it, and discard pongs and responses
  whose `utterance_id` or session does not match the active contest
  (§6.1, §6.4, §7.1);
- discard a response whose payload `skill_id` does not name a skill
  this contest requested an answer from (§7.1.1);
- request full answers via `ovos.common_query.request`, one per
  claimant named in `data.skill_id`, in parallel, and collect within
  a bounded window (§7.1, §7.2);
- extend the collection window past the initial value only while a
  claimant is outstanding, never past the ceiling, and never on the
  strength of a reported `latency_ms` (§7.2);
- apply confidence filtering and the denylist against the **live
  session** passed to `match`, not against any early-start snapshot
  (§5.1, §8);
- honour the live `session.blacklisted_skills` itself (§8 step 2) —
  the PIPELINE-1 §5.3 orchestrator backstop cannot see the answering
  skill, because `Match.skill_id` is the plugin's `pipeline_id`;
- run the contest for the first candidate utterance and report that
  same string as `Match.utterance` (§5.2, §9);
- return `None` when no response survives, letting the pipeline reach
  fallback (§9);
- return a `Match` with `skill_id` = its own `pipeline_id`,
  `intent_name` = `"common_query"`, `lang` = the `lang` argument
  passed to `match`, and `slots.answer` = the selected answer when one
  wins (§9);
- not mutate the session — `Match.updated_session` is omitted (§9);
- treat any language tag derived before the orchestrator's resolution
  as provisional, never publish it, and discard the early-start
  contest unless it equals the `lang` argument (§5, §5.1);
- key all contest state by `session_id` from `context.session`,
  alongside `utterance_id` (§6.4);
- speak the selected answer from `slots.answer` in the handler
  without re-dispatching to skills (§10).

### A common query pipeline plugin **SHOULD**:

- apply a question gate — classifier or other cheap short-circuit —
  to skip the contest for non-question-like utterances; gate-less
  deployments are conformant but pay the broadcast cost on every
  utterance (§4);
- subscribe to the utterance-arrival event and run the contest early,
  in parallel with upstream stages (§5);
- size the collection window from claimants' `latency_ms`, treating it
  as a hint and ignoring implausible values (§7.2);
- close the collection window on all-responded, or on fast-win only
  when the deployer has enabled it (§7.2, §8 step 3);
- use a reranker when configured (§8 step 4).

### A skill that participates in common query **MUST**:

- on `ovos.common_query.ping`, perform only a fast local check;
  **MUST NOT** perform network requests or blocking I/O during the
  pong phase (§6.2);
- echo the `utterance` verbatim in every pong and response (the
  evaluated candidate, §5.2), and derive both via `reply` so
  `context.utterance_id` propagates for correlation (§6.2, §6.4,
  §7.1);
- emit answers on `ovos.common_query.response` via `reply`
  (§7.1, §11);
- report its own `skill_id` in the payload, and ignore any request
  whose `data.skill_id` names another skill (§7.1.1);
- include `conf` whenever `answer` is present (§7.1);
- respond even when no answer can be produced (no `answer` field), so
  early termination can fire (§7.2);
- not call `ovos.utterance.speak` from the `common_query` handler
  (§11);
- not emit handler-lifecycle signals in response to
  `ovos.common_query.request` (§7.1).

### A skill that participates in common query **SHOULD**:

- respond to the pong within the deployer-configured bound
  (Appendix A, §6.2);
- report `conf` using the Appendix B ranges so values interoperate;
- include `latency_ms` in its pong so the plugin can size an adaptive
  collection window (§6.2);
- ignore unknown fields in `ovos.common_query.ping`.

---

## Appendix A — Tunable defaults (RECOMMENDED)

All values are deployer-configurable; these are the RECOMMENDED
defaults. They are guidance, not protocol — a deployment that tunes
them is conformant.

| Knob | Default | Section |
|------|---------|---------|
| Pong response bound (skill-side target) | 100 ms | §6.2 |
| Poll-window ceiling | 500 ms | §6.3 |
| Collection-window initial | 3 s | §7.2 |
| Collection-window ceiling | 5 s | §7.2 |
| Minimum self-confidence | 0.5 | §8 step 1 |
| Fast-win threshold (only when fast-win is enabled; default off) | 0.9 | §8 step 3 |

---

## Appendix B — Confidence-range guidance for skill authors

`conf` is self-reported and not calibrated across skills. These
ranges are RECOMMENDED so independently authored skills produce
comparable values; a reranker (§8 step 4) is the proper fix when
calibration matters.

| Range | Meaning |
|-------|---------|
| 0.0–0.3 | weak signal; something, but low certainty |
| 0.3–0.5 | partial match; can attempt an answer |
| 0.5–0.7 | reasonable answer; fairly confident |
| 0.7–0.9 | strong answer; confident |
| 0.9–1.0 | definitive answer; certain (use sparingly) |

---

## See also

- *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
  — the pipeline-plugin contract, the §4.4 blocking-match allowance
  and latency discipline, the `Match` shape, the dispatch model, the
  handler-lifecycle trio, the `ovos.utterance.handle` entry topic
  (§9.1), and the reserved intent_name registry.
- *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  `context.session` carrier, and `reply` derivation used for pong and
  answer responses.
- *Session Specification* (OVOS-SESSION-1) — the
  field-registry mechanism, the omission rule, and `session.lang`.
- *Session Lifecycle and State Ownership Specification*
  (OVOS-SESSION-2) — session-keyed state and mutation boundaries.
