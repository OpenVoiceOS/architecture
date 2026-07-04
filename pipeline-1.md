# Utterance Lifecycle and Pipeline Specification

**Spec ID:** OVOS-PIPELINE-1 · **Version:** 2 · **Status:** Draft

This document defines the **utterance lifecycle** — the path an
utterance takes from the moment it enters the assistant to the moment
the assistant is done with it — and the **pipeline plugin**
abstraction the **orchestrator** runs to decide what to do with each
utterance.

This is the **foundational bus specification for voice assistant
input/output**: it defines the natural-language entry point
(`ovos.utterance.handle`, §9.1), the pipeline plugin abstraction
the orchestrator iterates, and the natural-language exit point
(`ovos.utterance.speak`, §9.6). Intent registration and skill
dispatch are an optional layer built on top of this mechanism.

It builds on two companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  routing keys, session carrier, and derivations every Message
  defined here travels in;
- the *Intent Definition Specification* (OVOS-INTENT-3) — defines
  the *orchestrator* and the intent / handler model.

See also: OVOS-INTENT-4 (*Intent and Entity Registration Bus
Contract*) — the wire format pipeline plugins MAY consume to learn
what intents skills have registered. Consumption is plugin-
discretionary; this specification does not require it.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and
**MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **pipeline plugin** abstraction (§3) — the only thing the
  orchestrator iterates;
- the **match contract** (§4) — the only thing a plugin exposes;
- the **session fields owned by this specification** (§5):
  `session.pipeline` (positive whitelist + ordering),
  `session.blacklisted_pipelines`, `session.blacklisted_skills`,
  `session.blacklisted_intents` (negative filters);
- the **utterance lifecycle** (§6) — entry, iteration, dispatch,
  terminal events;
- the **dispatch** topic shape (§7) — `<skill_id>:<intent_name>`;
- the **handler-lifecycle trio** (§8) —
  `ovos.intent.handler.start` / `.complete` / `.error`;
- the **utterance-layer bus events** (§9) —
  the utterance entry topic `ovos.utterance.handle` (§9.1),
  `ovos.intent.matched`, `ovos.intent.unmatched`,
  `ovos.utterance.handled`, and the natural-language response topic
  `ovos.utterance.speak` (§9.6);
- **conformance** (§11).

It does **not** define:

- **what any pipeline plugin actually does** — plugins are black
  boxes identified by an opaque `pipeline_id`. The orchestrator's
  only contract with a plugin is the `match` operation of §4.
  Whether a plugin matches by template intents, keyword intents, a
  fine-tuned classifier, a chatbot, a language model, or anything
  else is the plugin's business.
- **what any handler does** — handlers are black boxes. Skills run
  their own; plugins that bundle handlers run theirs. The bus
  observes the handler-lifecycle trio (§8) and that is the full
  observable contract.
- **how plugins are loaded, discovered, configured, or
  instantiated** — a deployment concern.
- **how plugins consume registrations** — OVOS-INTENT-4 puts
  registrations on the bus; whether and how a given plugin
  subscribes is the plugin's own business.
- **the `session` lifecycle** — `session` is carried opaquely per
  OVOS-MSG-1 §4. The session fields this spec owns are listed in §5;
  other internal fields are owned by other specifications via the
  OVOS-SESSION-1 §2.2 registry mechanism.
- **per-plugin behavioural specs** — plugins have no behavioural
  contract beyond §4. A `converse` plugin, a `fallback` plugin, a
  persona plugin, a language-model plugin, a chatbot plugin: each
  defines itself.

---

## 2. The orchestrator and the pipeline plugin

The **orchestrator** (OVOS-INTENT-3 §6.1) is the logical role that
consumes the utterance-layer entry topic `ovos.utterance.handle`
(§9.1), iterates plugins per session, emits dispatch and terminal events,
and guarantees the universal end-marker `ovos.utterance.handled`.
The orchestrator is distinct from the **messagebus** (the transport
layer) and from any individual plugin.

The orchestrator MAY be implemented as a single process or as
multiple cooperating processes — a natural split along the audio
boundary runs an audio-input service (mic, STT), an utterance-
handling service (the pipeline and intent matching specified
here), and an audio-output service (TTS, playback) as separate
processes. From this specification's perspective those processes
together are "the orchestrator"; the split is a deployment /
containerization choice the spec accommodates but does not
prescribe. Pipeline plugins, the loaded-plugin set, and the match
contract of §4 live in the orchestrator process that implements
the utterance lifecycle (the utterance-handling service in the
split shape above).

The orchestrator is **stateless for named sessions** and holds
persistent state only for the reserved `session_id == "default"`
(OVOS-SESSION-1 §3.1). The full state-ownership model is owned by
OVOS-SESSION-2; consumers of this spec MAY take it as a
working assumption that each inbound utterance brings its own
session and the orchestrator does not maintain cross-utterance
state for named sessions.

A **pipeline plugin** is a third-party component identified by an
opaque `pipeline_id` — an arbitrary, deployment-unique string. The
orchestrator loads some number of plugins at startup; how it
discovers and instantiates them is a deployment concern. Each
plugin exposes one operation to the orchestrator (§4) and is
otherwise a black box.

---

## 3. Pipeline plugins

A pipeline plugin is identified by an opaque **`pipeline_id`** —
an arbitrary string. The orchestrator's loaded-plugin set is a
mapping `pipeline_id → plugin instance`; the orchestrator does
not interpret the `pipeline_id` string beyond using it as a key.

Constraints on `pipeline_id` strings:

- Non-empty.
- Bound by OVOS-MSG-1 §2.1.1: because `pipeline_id` appears as a
  component in colon-separated topic shapes (`<skill_id>:<intent_name>`
  in §7, per-pipeline introspection topics in §10), it **MUST NOT**
  contain `:`. The recommended form is ASCII letters / digits / `_` /
  `-` only.
- Unique within a deployment's loaded-plugin set.

A plugin **MAY** appear in a session's pipeline more than once
under different `pipeline_id`s if the plugin chooses to expose
multiple matching modes (for example, a strict mode and a
permissive mode). The orchestrator treats each `pipeline_id` as a
distinct stage.

### 3.1 Pipeline attribution

The orchestrator stamps `context["pipeline_id"]` on the dispatch
Message (§7.1) — this is the first point at which the matching
plugin's identity appears on the wire. From there it propagates
through all Messages the handler emits via MSG-1 derivation
semantics, making every downstream Message attributable to the
plugin that produced the match without any further action by the
plugin or the handler.

---

## 4. The match contract

A plugin exposes one operation to the orchestrator:

```
match(utterances, lang, session) → Match | None
```

Inputs:

- `utterances` — a **non-empty list of candidate strings**. The
  list typically originates from the entry topic (§9.1) and may
  have been modified by the utterance-transformer
  chain (OVOS-TRANSFORM-1 §3.2) before reaching the plugin. A
  plugin **MUST** accept this shape: a list of one or more
  candidate transcripts, in no particular order, all in the same
  language. A plugin is free to consider all candidates, only the
  first, or any subset; the orchestrator does not prescribe how
  candidates are weighted.
- `lang` — the BCP-47 content-language tag. When the entry-topic
  (§9.1) carried an authoritative `Message.data.lang`, the
  orchestrator passes it through. When it did not, the
  orchestrator **MUST** resolve the utterance language **once**
  per utterance, from the per-utterance evidence fields of
  OVOS-SESSION-1 §3.2 (user preference, lang-detect signals), and
  pass the resolved tag to **every** plugin's `match` call for
  that utterance. A single resolution point keeps the match round
  coherent: if each plugin re-derived language independently, the
  same utterance could be matched in different languages at
  different pipeline stages, and which language "wins" would be an
  accident of ordering. A plugin **MAY** refine the received tag
  (e.g. a multilingual matcher that detects a different content
  language) but **MUST NOT** re-derive it independently from
  session evidence, and **MUST** declare the language it actually
  matched in via `Match.lang`.
- `session` — the session carrier from `context.session` of the
  utterance Message (OVOS-MSG-1 §4, OVOS-SESSION-1).

Output: either `None` (decline) or a `Match` object with the
fields below.

### 4.1 The `Match` shape

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The `skill_id` of the handler to invoke. For a pipeline plugin that matches itself, this equals its `pipeline_id` (§7.0). |
| `intent_name` | string | yes | An opaque non-empty string that, together with `skill_id`, names the handler to invoke. For skill-owned matches this is the intent name the skill registered. For plugin-owned matches this is whatever label the plugin chose for this response. |
| `lang` | string | yes | The BCP-47 language tag the match was performed against. The plugin **MUST** set this — it is the plugin's explicit responsibility to declare what language its match is in. A plugin that received a `lang` parameter and matched in that language returns it here; a plugin that determined the language by other means (multilingual matcher, hard-coded engine, content-language detection) sets it to whatever language the match was performed in. |
| `slots` | object (string→string) | yes | The slot map (§4.3). MAY be empty. |
| `utterance` | string | yes | The specific candidate string from the input list that won the match. A plugin that does not track which candidate won **MUST** populate this with the first element of the input list as a fallback; the orchestrator forwards this value verbatim as `data.utterance` in the dispatch payload (§7.1) and **MUST NOT** substitute another value. |
| `updated_session` | object | no | A replacement `session` snapshot the plugin produced during `match` (§4.2). When present, the orchestrator MUST use this snapshot — in place of the inbound utterance's session — for the dispatch and every downstream stage. When absent, the inbound session is carried unchanged. This is the **only** mechanism by which a plugin's match-phase session mutations reach downstream consumers; in-place mutations of the inbound session object are not visible past the plugin boundary. |

The orchestrator interprets a non-`None` return as a definitive
claim. It does not score, rank, or rerank matches across plugins —
**first match wins** (§6). A plugin that wants to express
uncertainty must return `None` and let a later plugin claim.

### 4.2 The match contract is the single obligation

The plugin's `match` operation has **one obligation**: return a
`Match` (§4.1) or `null`. The orchestrator does not constrain
anything else about what `match` does internally — emitting bus
Messages during `match` is **allowed** (a plugin that polls
other components, calls out to a model server, asks the user a
disambiguation question, or runs any other matching strategy
that requires bus communication is conformant), and side effects
on plugin-internal state are the plugin's own business.

**Session mutation via `Match.updated_session`.** A plugin MAY
mutate session state as part of producing a Match — for
example, an intent plugin setting `session.intent_context` on
the match dispatch, a converse plugin setting
`session.response_mode` because the matched intent enables a
follow-up wait window, or any plugin reordering
`session.pipeline` for subsequent utterances on this session.
The mutation is communicated to the orchestrator via the
`updated_session` field on the returned `Match` (§4.1): the
plugin populates `updated_session` with the new session
snapshot it wants downstream consumers to see, and the
orchestrator MUST use that snapshot for the dispatch and every
subsequent stage of the utterance lifecycle (§6). When
`updated_session` is absent, the inbound utterance's session is
carried unchanged.

The `updated_session` pathway is **only effective for a claiming
match**. A plugin that returns `null` (declines) does not return
any `Match`, and therefore any session mutation it performed
during its match call is **discarded** at the plugin boundary:
the orchestrator continues iteration with the inbound session
snapshot, untouched. This is what makes match-phase mutation
safe under §6.2 first-match-wins iteration — a declined
plugin's exploratory mutations never reach later plugins or
downstream stages.

The orchestrator-side pattern is uniform:

```
match = plugin.match(utterances, lang, session)
if match is not None:
    session = match.updated_session or session
    # dispatch and downstream stages use this session
```

A plugin that mutates the inbound session object **in place**
during `match` without populating `updated_session` is
non-conformant — the in-place mutation may or may not be visible
to the orchestrator depending on object identity, and the field
is the only guaranteed-visible match-phase channel. Plugins that
need to mutate session state from `match` **MUST** do so via a
fresh snapshot returned in `updated_session`, not via in-place
mutation.

This rule applies only to `match`-phase mutations. Session
mutation from **handlers** (under §7 dispatch), from
**transformers** (OVOS-TRANSFORM-1 §3), and from the **direct
session-mutation pathway** of OVOS-MSG-1 (which CONTEXT-1 §5.3
and CONVERSE-1 §3.2 build on) is governed by those specs and is
unaffected by §4.2.

### 4.3 The slot map

`Match.slots` is a `{string: string}` mapping (the same shape
OVOS-INTENT-3 §7 defines for template / keyword intent slots).

For skill-owned matches against intents the plugin previously
consumed from OVOS-INTENT-4 registrations, the slot map keys
are the slot names (template intents) or vocabulary names (keyword
intents) of the matched intent.

For plugin-owned matches, the slot map is whatever the plugin
chooses to surface. It **MAY** be empty.

The orchestrator does not interpret the slot map; it forwards
it to the dispatched handler.

### 4.4 Match-phase timeout and latency discipline

The `match` operation is logically synchronous from the orchestrator's
perspective — the orchestrator calls `match` and waits for the return
value. Because §4.2 permits a plugin to communicate over the bus during
`match` (poll a model server, ask the user a disambiguation question,
etc.), the call can block for an unbounded time.

The orchestrator **SHOULD** bound each `match` invocation by a
deployment-defined time. The **RECOMMENDED** default is **10 s**.
When a bound is applied, it **MUST** be at least as large as any
collection ceiling a stage runs internally (e.g. the
OVOS-COMMON-QUERY-1 §2.1 collection window) — a match-phase timeout
shorter than a stage's own internal wait guarantees that stage is
killed mid-collection on every utterance it handles. If a plugin
has not returned within the bound,
the orchestrator **MUST** treat the call as if the plugin had raised an
exception — log the timeout, skip to the next plugin per §6.2, and
continue normally. Any partial mutation performed by the plugin during the
timed-out call is discarded (there is no `Match` to carry an
`updated_session`; the inbound session is unchanged). No bus event is
emitted for the timeout at this stage.

The timeout bound and whether it counts toward the §6.2 circuit-breaker
are deployer-configurable.

**Latency discipline.** In voice-assistant deployments, match-phase
latency directly determines response latency — the pipeline is
sequential and the user is waiting. Plugins **SHOULD** therefore return
from `match` as quickly as possible and defer all long-running work to
the handler phase. A plugin that can determine it will claim an
utterance without fully processing it **SHOULD** return a `Match`
immediately and begin expensive processing (model inference, network
calls, disambiguation) inside the handler, not inside `match`.

A language-model plugin is the canonical example: it typically knows it
will consume any utterance that reaches it and can return a `Match`
immediately; the actual generation belongs in the handler. The match
phase is a routing decision, not a processing phase.

The orchestrator **SHOULD** surface match-phase duration as an
observable metric so deployers can identify plugins that violate this
discipline.

---

## 5. Session fields owned by this specification

This specification claims four session fields per OVOS-SESSION-1
§2.2: one **positive** ordering field (§5.1 `pipeline`) and three
**negative** filtering fields (§5.2 `blacklisted_pipelines`, §5.3
`blacklisted_skills`, §5.4 `blacklisted_intents`). All four are
session-scoped, propagate with the session under OVOS-SESSION-1 §4,
and follow the deployment-default-fallback absence rule of
OVOS-SESSION-1 §2.1: an omitted, empty, or absent field resolves at
consumption to the deployment-configured default.

### 5.1 `session.pipeline`

An ordered array of `pipeline_id` strings expressing the **session
origin's preference** for which plugins to run and in what order.
It is a preference, not an authorization: the orchestrator narrows
the requested list to what is loaded (below) and what policy permits
(§5.5).

Any session — local, remote, layer-2-attached, programmatic — MAY
populate `session.pipeline` to request a specific ordering. The
orchestrator does not interpret who set it; the field is a
preference channel.

Example:

```json
{
  "session": {
    "pipeline": [
      "template-high",
      "keyword-high",
      "template-medium",
      "keyword-medium",
      "common-qa",
      "persona-high",
      "fallback-low"
    ]
  }
}
```

For each utterance, the orchestrator iterates `session.pipeline`
in order, calling `match` on each corresponding plugin (§6.2).

If a `pipeline_id` in `session.pipeline` does not correspond to
any loaded plugin, the orchestrator **MUST** skip it and **SHOULD**
log a warning. It **MUST NOT** abort the utterance over an unknown
identifier and **MUST NOT** fall back to the deployment default
merely because one identifier is unknown — the remaining known
identifiers are the effective ordered set.

If `session.pipeline` is absent or empty (per OVOS-SESSION-1 §2.1),
the orchestrator falls back to the **default-session pipeline**: the
pipeline configured for the reserved `session_id == "default"`
session (OVOS-SESSION-1 §3.1). The default-session pipeline is owned
and maintained by the orchestrator and represents what the
deployment runs when no preference is expressed. If the default
session itself has no `pipeline` configured, the utterance proceeds
to no-match (`ovos.intent.unmatched`, §9.3).

Different sessions may carry different `pipeline`. This is how a
session origin expresses different preferences for different
participants — for example, a remote-peer session may request a
restricted pipeline tailored to that participant's needs. Whether
that preference is honoured is a policy decision (§5.5).

### 5.2 `session.blacklisted_pipelines`

An unordered array of `pipeline_id` strings the orchestrator
**MUST NOT** invoke for this session.

`blacklisted_pipelines` is the **policy channel** for pipeline
selection. Where `session.pipeline` (§5.1) is the session origin's
preference, `blacklisted_pipelines` is enforcement: a plugin listed
here **MUST NOT** be invoked for this session **even if the same
`pipeline_id` is requested in `session.pipeline`**. Policy overrides
preference (§5.5).

Filtering is **orchestrator-only**: when the orchestrator iterates
its effective pipeline (per §5.5), it **MUST** skip any
`pipeline_id` listed here as if it were not loaded. No `match` call
is made; no bus event is emitted for the skip. The filtering is
observable only as a non-invocation.

Unknown `pipeline_id`s in `blacklisted_pipelines` are harmless and
**MUST NOT** cause the utterance to abort — they simply match
nothing.

An empty array (`[]`) is wire-equivalent to omission: both fall
back to the deployment default per OVOS-SESSION-1 §2.1. A
producer with no pipelines to deny **SHOULD** omit the field
rather than emit `[]`, per the wire-weight guidance of
OVOS-SESSION-1 §3.4.

### 5.3 `session.blacklisted_skills`

An unordered array of `skill_id` strings (OVOS-INTENT-3) whose
intents **MUST NOT** be matched for this session.

The contract is **two-tier**:

1. A pipeline plugin **SHOULD NOT** return a `Match` whose
   `skill_id` (§7.1) is a `skill_id` listed here. A plugin's
   internal handling of would-match-but-blacklisted candidates is
   **not specified** — it MAY skip the candidate before scoring,
   suppress its score below a match threshold, route to a
   plugin-internal default-handler, or anything else — as long as
   the returned `Match` does not name a blacklisted skill.
2. A pipeline plugin that does not implement filtering is **not
   conformant** with this field. The orchestrator **MUST** therefore
   act as backstop: after a plugin returns a candidate `Match`, the
   orchestrator **MUST** check `Match.skill_id` against
   `blacklisted_skills` and, if listed, **MUST** treat the match as
   if the plugin had declined — continue iteration to the next
   plugin per §6.2. No bus event is emitted for backstop filtering;
   it is observable only as a non-match.

Empty-array semantics match §5.2: `[]` is wire-equivalent to
omission. A producer with no skills to deny **SHOULD** omit the
field.

### 5.4 `session.blacklisted_intents`

An unordered array of fully-qualified `<skill_id>:<intent_name>`
strings (the dispatch-topic shape of §7) whose specific intents
**MUST NOT** be matched for this session.

The contract is identical in shape to §5.3 (two-tier:
plugin-SHOULD + orchestrator-MUST-backstop), with the comparison
performed against the candidate `Match`'s dispatch identity
`<Match.skill_id>:<Match.intent_name>`.

The bare `intent_name` form is **not** accepted in this field.
`intent_name` is only unique within an owner, so a bare entry would
silently denylist every same-named intent across every skill and
every pipeline plugin in the deployment — a sharp footgun. A
producer **MUST** emit fully-qualified entries; a consumer **MAY**
reject malformed (non-colon-bearing) entries or **MAY** ignore them
silently, but **MUST NOT** broaden a bare entry to all owners.

Entries are **language-agnostic.** OVOS-INTENT-4 §3.2 keys intent
identity on the triple `(skill_id, intent_name, lang)`, so a single
intent registered for `en-US` and `de-DE` is two separate
registrations. A `blacklisted_intents` entry
`<skill_id>:<intent_name>` denies both — there is no per-language
denylist. A deployment that needs language-scoped denial expresses
it through a session whose `lang` already narrows the set of
matchable registrations.

Empty-array semantics match §5.2: `[]` is wire-equivalent to
omission. SHOULD-omit when there is nothing to deny.

### 5.5 Composition: preference, availability, policy

The four fields layer in a fixed order: a **preference** stage
(§5.1), an **availability** stage (the loaded-plugin set), and a
**policy** stage (§5.2 / §5.3 / §5.4). Each later stage may narrow
the result of the earlier ones; no later stage adds anything an
earlier stage rejected.

The orchestrator computes the **effective pipeline** for an
utterance:

1. **Preference.** Start from `session.pipeline` if set and
   non-empty; otherwise start from the default-session pipeline
   (§5.1).
2. **Availability.** Drop any `pipeline_id` that does not
   correspond to a plugin loaded by the orchestrator. Unknown
   identifiers do not abort the utterance and do not trigger
   fallback to the default-session pipeline — the remaining known
   identifiers are the effective ordered set (§5.1).
3. **Policy.** Drop any `pipeline_id` listed in
   `session.blacklisted_pipelines`, even if it was explicitly
   requested in step 1. Policy overrides preference.

The result is the ordered list of `pipeline_id`s the orchestrator
iterates for this utterance.

`session.blacklisted_skills` and `session.blacklisted_intents` are
**not** applied at this stage. They are per-candidate policy filters
applied during iteration against each `Match` a plugin returns
(§5.3, §5.4). The two-tier shape (plugin SHOULD, orchestrator MUST
backstop) ensures policy enforcement regardless of plugin
conformance.

The intended separation of concerns is sharp:

- **Any session origin — including the participant on the user
  side of the bus — MAY request a preferred pipeline via
  `session.pipeline`.** This is a request channel, available to
  every emitter without authorization.
- **Only policy** (the denylists, typically populated by the
  orchestrator owner or by a layer-2 substrate that owns the
  session, see §5.6) can refuse a request. Policy is enforcement;
  preference is request. The two fields are layered, not
  alternatives.

If every requested `pipeline_id` is dropped by availability or
policy, the effective pipeline is empty and the utterance proceeds
directly to no-match (`ovos.intent.unmatched`, §9.3). The
orchestrator **MUST NOT** silently fall back to the default-session
pipeline in this case — falling back would let a policy-rejected
preference pull in a different ordering the origin never asked for
and policy never approved.

### 5.6 Use under layer-2 substrates (informative)

The §5.5 layering — preference from any origin, enforcement from
policy — is precisely what a layer-2 substrate (per OVOS-MSG-1
§3.4 / §4.4) needs to express **granular per-peer permissions** in
a multi-tenant deployment, without inventing a separate
authorization channel.

The intended split:

- A **client** (the participant on the user side of the bus —
  local device, remote peer, satellite, programmatic caller) sets
  `session.pipeline` to request what it would *like* to run.
  Clients are not trusted to grant themselves capabilities; they
  are only stating a preference.
- A **layer-2 substrate** that owns the session (typically because
  it attached the per-peer session at connection time) populates
  `session.blacklisted_pipelines`, `session.blacklisted_skills`,
  and `session.blacklisted_intents` from the peer's permission
  grant. These ride on every derived Message through OVOS-SESSION-1
  §4 propagation, so no per-hop re-authorization is needed and no
  orchestrator-side change is required to add authorization.

The orchestrator enforces the intersection: §5.5 step 3 drops
disallowed pipelines from the request; §5.3 / §5.4 drop disallowed
matches per candidate. A client that requests a forbidden plugin or
intent simply gets no result for that part of its request — its
preference is silently narrowed, exactly as if the plugin were not
loaded.

This specification reserves no fields for layer-2 authorization
beyond the three denylists; the broader authorization model
(identity verification, peer-to-grant binding, revocation,
auditing) is the layer-2 substrate's concern, not PIPELINE-1's.

---

## 6. The utterance lifecycle

Every utterance flows through the same lifecycle, regardless of
which plugin (if any) claims it. The lifecycle is **guaranteed
to terminate** with exactly one `ovos.utterance.handled` event
(§9.5).

### 6.1 The flow

```
ovos.utterance.handle                    ← entry (§9.1)
   │
   ├─ session retrieval; effective pipeline composed (§5.5)
   │  (preference → availability → policy)
   │
   ├─ utterance-transformer chain runs   ← TRANSFORM-1 §3.2
   ├─ metadata-transformer chain runs    ← TRANSFORM-1 §3.3
   │
   ├─ for pipeline_id in effective pipeline:
   │     plugin = loaded_plugins[pipeline_id]     # skip if not loaded
   │     match = plugin.match(utterances, lang, session)
   │     if match is None:
   │         continue   # any plugin-side updated_session is discarded
   │
   │     orchestrator-backstop denylist check (§5.3/§5.4)
   │     orchestrator-backstop required_slots check (§6.2)
   │     if filtered:  continue
   │
   │     session = match.updated_session or session   # §4.1, §4.2
   │
   │     ┌── post-match-pre-dispatch window ──────────────┐
   │     │ engine-side context promotion (CONTEXT-1 §5.1) │
   │     │ intent-transformer chain runs (TRANSFORM-1     │
   │     │   §3.4) — may modify Match.slots, MUST NOT  │
   │     │   change skill_id / intent_name                 │
   │     └────────────────────────────────────────────────┘
   │
   │     ovos.intent.matched                  (§9.2)
   │     dispatch on <match.skill_id>:<match.intent_name>  (§7)
   │     (handler runs; emits lifecycle trio §8)
   │     ovos.utterance.speak (×0..N)          (§9.6)
   │     ovos.utterance.handled               (§9.5)
   │     break
   │     [output layer — outside this spec's scope]
   │     (dialog-transformer chain ← TRANSFORM-1 §3.5)
   │     (tts-transformer chain   ← TRANSFORM-1 §3.6)
   │
   ├─ if no plugin matched (or all matches filtered):
   │     ovos.intent.unmatched                  (§9.3)
   │     ovos.utterance.handled                   (§9.5)
   │
   └─ post-match decrement turns_remaining--   ← CONTEXT-1 §4
      (runs after the match round whether or not any intent
       matched; entries freshly written this round — CONTEXT-1
       §4.1 — are exempt)
```

The flow diagram shows where companion-spec chains plug into this
specification's iteration loop. The **audio-transformer chain**
(TRANSFORM-1 §3.1) runs entirely in the audio-input service before
the entry topic is emitted and is therefore not visible here. The
**utterance** and **metadata** transformer chains run after entry
and before iteration, against the candidate utterance list. The
**post-match-pre-dispatch window** is where
CONTEXT-1 §5.1 sanctions engine-side `session.intent_context`
mutation and where TRANSFORM-1 §3.4 inserts the intent-transformer
chain over the chosen `Match`. **`ovos.utterance.handled` is emitted at handler completion** —
immediately after `ovos.intent.handler.complete` (or `.error`).
The **dialog-transformer** and **TTS-transformer** chains
(TRANSFORM-1 §3.5 / §3.6) run in the output layer after
`ovos.utterance.handled`, just before TTS rendering; they are
outside this specification's scope and are not a synchronization
barrier for the end-marker. Audio output is fully decoupled from
the pipeline: a chat-only deployment receives the same utterance
lifecycle and the same end-marker as an audio deployment.

Pseudocode is informative; normative rules are in §§4–9.

### 6.2 First-match-wins iteration

For each utterance, the orchestrator **MUST**:

- run the utterance-transformer and metadata-transformer chains
  (OVOS-TRANSFORM-1 §3.2, §3.3) before pipeline iteration begins;
- if the utterance-transformer chain returns an **empty
  utterance list**, skip pipeline iteration entirely and proceed
  directly to `ovos.intent.unmatched` (§9.3) — `match()` is
  contractually defined over a non-empty list (§4) and the
  orchestrator **MUST NOT** invoke any plugin with an empty
  list. (If the empty list arrived together with cancellation
  context per OVOS-TRANSFORM-1 §8.1, the cancellation terminal
  path of §8.2 there takes precedence over no-match here.)
- iterate `session.pipeline` in order;
- for each `pipeline_id`, call `match` on the corresponding loaded
  plugin (skipping unknown identifiers, §5);
- stop at the **first plugin** that returns a non-`None` `Match`;
- if no plugin returns a `Match`, emit `ovos.intent.unmatched`
  (§9.3).

**Evaluation order is the arbitration model.** The orchestrator
deliberately does not compare confidence across plugins: a plugin
positioned earlier in `session.pipeline` gets **first refusal** on
every utterance, and the first claim wins. This is what makes
**stateful interception** possible. A plugin's decision to claim may
depend not only on the utterance but on **session state** — a converse
plugin claims only when there is an active handler or an open
`response_mode` (a skill in response mode awaiting a reply); a persona plugin
only while a persona is active; a media plugin claims *resume* only
while it holds paused media; a stop plugin only when there is something
to stop. The same utterance ("yes", "next", "stop", "resume")
therefore routes to a different handler depending on the session, and
only ordering can guarantee that the stateful interceptor sees it
*before* the general intent engines that would otherwise match the bare
words; a ranked model could let a higher-scoring general match steal a
turn that belongs to an active handler.

A **selective** plugin — one with a strict false-positive budget that
expects to decline most utterances — is correspondingly expected to be
**conservative**: claim only when both the utterance and its state
warrant it, return `None` otherwise, and trust its position rather than
compete on a score. Cross-plugin ranking is not merely omitted:
heterogeneous engines (a keyword matcher, a neural classifier, a
language model) share no calibrated score space, and a state-derived
certainty ("I hold paused media, so *resume* is mine") is not a
quantity a text-similarity score can outbid. Deployers express policy
by **ordering** `session.pipeline` (§5.1); each plugin decides its own
claim from the utterance and the session it was handed (§4.1, §4.2).
The two concerns stay separate.

A plugin that raises an exception during `match` is treated as if
it returned `None`. The orchestrator **MUST** continue to the next
plugin and **SHOULD** log the exception. A single plugin's bug
does not fail the whole utterance.

**Repeated-exception circuit-breaker.** An orchestrator **SHOULD**
drop a plugin from the effective pipeline after a deployer-tunable
consecutive-exception threshold. A dropped plugin behaves as if
absent; recovery is a deployment concern. The threshold and scope
(per-session or process-wide) are deployer-configurable.

**Orchestrator backstop for `required_slots`.** After a plugin
returns a `Match`, the orchestrator **MUST** verify that the match's
slot map contains every slot listed in the intent's `required_slots`
(INTENT-3 §5.3). The orchestrator obtains this information from the
same registration data the plugin consumed — in-process, this is
available from the plugin's compiled state or from the orchestrator's
own manifest (INTENT-4 §10). If any required slot is absent, the
orchestrator **MUST** treat the match as if the plugin had declined
and continue iteration to the next plugin. This check operates
after the `blacklisted_skills` / `blacklisted_intents` backstop
(§5.3, §5.4) and uses the same observable semantics: no bus event
is emitted; it is observable only as a non-match.

The primary obligation to enforce `required_slots` still lies with
the engine during `match()`. The orchestrator backstop is a
second line of defense against engine bugs or plugins that do not
implement the rule.

### 6.3 Plugins do not see each other's matches

A plugin receives the same utterance every other plugin in the
pipeline received; it has no access to what an earlier plugin
tried or why it declined. Cross-plugin coordination belongs in
`session` (OVOS-MSG-1 §4) or in plugin-side out-of-band state
keyed on `session.session_id` (per OVOS-MSG-1 §5.4 —
"no central correlation, no central state").

### 6.4 Terminal events

Every utterance terminates in exactly one of three ways, each
followed by the universal end-marker `ovos.utterance.handled`:

| Outcome | Sequence of utterance-layer events |
|---------|------------------------------------|
| Matched by a plugin | `ovos.intent.matched` → dispatch + (handler trio §8) → `ovos.utterance.speak` ×0..N → `ovos.utterance.handled` |
| No plugin matched | `ovos.intent.unmatched` → `ovos.utterance.handled` |
| Cancelled by a transformer | `ovos.utterance.cancelled` → `ovos.utterance.handled` (see OVOS-TRANSFORM-1 §8.2) |

If a dispatched handler emits `ovos.intent.handler.error` (§8)
instead of `.complete`, the orchestrator still emits
`ovos.utterance.handled` afterwards. The "every utterance
terminates with `ovos.utterance.handled`" invariant holds across
all paths.

### 6.5 Long-running handlers and nested utterance lifecycles

Handlers are long-running by design. A handler **MAY** block
for an unbounded duration — for example, to run a voice game,
a multi-step interaction, or any flow that asks the user one or
more follow-up questions. This is not a timeout condition and
**MUST NOT** be treated as one.

When a handler asks the user a question and waits for the reply
(entering response mode, OVOS-CONVERSE-1 §5), the
following happens on the bus:

```
ovos.utterance.handle          (original utterance)
  ovos.intent.handler.start    (outer handler)
  ovos.utterance.speak         (handler's question to the user)
  [outer handler blocks]
    ovos.utterance.handle      (user's reply)
      ovos.intent.matched
      ovos.intent.handler.start    (inner :response dispatch)
      ovos.intent.handler.complete
    ovos.utterance.handled     (user's reply — inner lifecycle ends)
  [outer handler unblocks, continues]
  ovos.intent.handler.complete (outer handler)
ovos.utterance.handled         (original utterance — outer lifecycle ends)
```

The inner utterance is a complete, independent lifecycle:
it enters on `ovos.utterance.handle`, is matched and dispatched
by the converse plugin on `<skill_id>:response`
(OVOS-CONVERSE-1 §5), and terminates with its own
`ovos.utterance.handled`. The outer lifecycle's
`ovos.utterance.handled` does not fire until the outer handler
returns, which may be after arbitrarily many inner lifecycles.

**The "exactly one `ovos.utterance.handled` per
`ovos.utterance.handle`" invariant (§6.4) applies independently
to each entry message.** It says nothing about ordering between
concurrent or nested lifecycles; interleaved handler trios and
end-markers are conformant and expected.

**The orchestrator MUST remain able to accept and process new
`ovos.utterance.handle` messages while a handler is running.**
An orchestrator that blocks the utterance-entry subscription for
the duration of a handler invocation will deadlock the first time
any handler waits for a user reply. Concurrent utterance processing
is a structural requirement, not an optimisation.

The same liveness requirement applies within the match phase: the
orchestrator **MUST** continue servicing its bus subscriptions —
including poll replies destined for an in-flight plugin (a stop
plugin's pongs, a converse or fallback poll's responses) — while a
`match` call is in flight. An orchestrator whose bus loop blocks on
the synchronous `match` return deadlocks every plugin whose match
strategy involves a bus round-trip (§4.2).

The session is the correlation key for nested lifecycles: the
inner utterance carries the same `session_id` with
`session.response_mode` populated (OVOS-CONVERSE-1 §5), which
is what the converse plugin reads to route the reply to the
waiting handler. No additional correlation field is defined by
this specification.

---

## 7. Dispatch

When a plugin's `match` returns a non-`None` `Match`, the
orchestrator dispatches the matched handler by emitting a Message
on the topic:

```
<skill_id>:<intent_name>
```

where `<skill_id>` is `Match.skill_id` and `<intent_name>` is
`Match.intent_name`. Both segments are bound by OVOS-MSG-1 §2.1.1 — neither may
contain `:` — so the single `:` split is unambiguous.

### 7.0 `Match.skill_id` is the handler's identity

`Match.skill_id` is the `skill_id` of the component that will
handle the dispatch. The orchestrator does not distinguish between
a skill whose intents were registered via OVOS-INTENT-4 and a
pipeline plugin that matched itself — both are reached by the
same `<skill_id>:<intent_name>` dispatch topic, and the
dispatched handler has the same obligations as any skill
(OVOS-INTENT-4 §3.1).

A pipeline plugin that returns matches where `skill_id` equals its
own `pipeline_id` is simply a component whose `skill_id` and
`pipeline_id` happen to be the same identifier. It skips the
OVOS-INTENT-4 registration step because it consumes no external
intent registry — its `match` implementation decides directly
whether to claim the utterance. There is no architectural
difference; the dispatch path is identical.

### 7.1 Routing and payload

The dispatch Message's `context` (OVOS-MSG-1 §4):

- `session` is propagated from the originating utterance;
- `source` and `destination` follow the single-flip routing model
  (OVOS-MSG-1 §5.2) — the orchestrator derives the dispatch via
  `reply`, so `destination` is the original utterance emitter and
  `source` is the orchestrator;
- **`context["skill_id"]` stamping.** The orchestrator **MUST**
  stamp `context["skill_id"] = <skill_id>` on every dispatch.
  MSG-1 derivation semantics carry this value forward into every
  Message the handler emits, satisfying OVOS-INTENT-4 §3.1 by
  construction.
- **`context["pipeline_id"]` stamping.** The orchestrator **MUST**
  stamp `context["pipeline_id"]` on every dispatch with the
  `pipeline_id` of the plugin that produced the match (§3.1). When
  the match is self-addressed (`skill_id == pipeline_id`, §7.0),
  both context keys carry the same identifier.
- **`session.active_handlers` push.** The orchestrator **MUST**
  push `{skill_id: <skill_id>, activated_at: <orchestrator-stamped
  Unix timestamp in seconds>}` onto `session.active_handlers`,
  evicting any prior entry with the same `skill_id`. The list is
  a recency record keyed by `activated_at` — consumers determine
  "most recently activated" by comparing timestamps, not by list
  position. The push is
  **suppressed** only for dispatches on reserved intent_names
  listed in §7.3 — a reserved-name dispatch represents a
  continuation of an already-active skill's participation or its
  termination, not a fresh activation. The orchestrator applies
  the polymorphism rule (§7.0) uniformly and does not otherwise
  distinguish skill from pipeline-plugin dispatches: suppression
  **MUST** be keyed on the Match's `intent_name` appearing in the
  §7.3 reserved-name registry — never on the producing
  `pipeline_id`. The push is
  applied after `Match.updated_session` is committed: a plugin
  that mutates `active_handlers` via `updated_session` (e.g.,
  STOP-1's global stop wiping the list) sees the stamp applied
  on top, so the dispatched skill_id always lands at the head
  unless the intent_name is reserved.

The dispatch Message's `data`:

```json
{
  "lang": "en-US",
  "utterance": "play the beatles",
  "slots": { "query": "the beatles" }
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `lang` | string | yes | The content language of the match, taken directly from `Match.lang`. A `Match` with no `lang` is malformed; the orchestrator **MUST** treat it as if the plugin declined and continue iteration. |
| `utterance` | string | yes | The candidate string that won the match. |
| `slots` | object (string→string) | yes | The slot map (§4.3). MAY be empty. |

`skill_id` and `intent_name` are not repeated in the payload — they are the topic's `<skill_id>:<intent_name>` prefix and suffix. A handler that needs them splits the topic on `:`.

### 7.2 Subscription discipline

Each handler subscribes to exactly its own
`<skill_id>:<intent_name>` topic. A skill subscribes to topics
under its own `skill_id`; a plugin that bundles its own handlers
subscribes to topics under its own `pipeline_id`. Because each
topic is unique to one handler, the bus delivers the dispatch
only to the intended consumer.

A consumer that receives a dispatch on a topic it should not be
listening to (a configuration bug) **MUST NOT** run the handler
and **SHOULD** log the discrepancy. The orchestrator does not
police subscriptions.

### 7.3 Reserved intent_names

Other normative specifications **MAY** reserve specific
`intent_name` values for matches produced by a particular
pipeline plugin role. A reserved intent_name is one that:

- skills and pipelines **MUST NOT** register under OVOS-INTENT-4;
  a registration naming a reserved intent_name is malformed and
  every consumer (including the orchestrator's manifest) treats it
  under the OVOS-INTENT-4 §5.3 malformed-payload rules — log at
  WARN, do not index;
- a pipeline plugin **MAY** emit as the `intent_name` of a
  returned `Match` to signal "this match was produced by the
  role that reserves the name"; the dispatch then proceeds
  normally per §7, addressed to `<skill_id>:<reserved_name>`,
  and the handler subscribed to that topic does whatever the
  reserving specification defines.

A reservation is a **namespace lease**, not a dispatch
modification. Dispatches on reserved intent_names fire §7.1
context stamping, §7.2 routing, and §8 handler-trio identically
to ordinary dispatches. The one exception is the
`session.active_handlers` push defined in §7.1, which is
suppressed on reserved-name dispatches — a reserved name
represents a continuation or termination of an already-active
skill's participation, not a fresh activation. Stamping
suppression is keyed on the Match's reserved `intent_name` (this
registry), never on the producing `pipeline_id`. The reserving
specification gets exclusive use of the name across the
deployment's skill set; it gets no other privilege.

Reserved intent_names:

| Reserved intent_name | Reserving spec | Meaning of a Match bearing this name |
|----------------------|----------------|--------------------------------------|
| `converse` | OVOS-CONVERSE-1 §4 | a converse plugin's claim that `<skill_id>` (an active handler) wants this utterance — the orchestrator dispatches `<skill_id>:converse` and the owner's converse handler runs |
| `response` | OVOS-CONVERSE-1 §5 | a converse plugin's signal that `<skill_id>` (the response-mode holder) is to receive the awaited utterance — the orchestrator dispatches `<skill_id>:response` and the owner's response handler runs |
| `stop` | OVOS-STOP-1 §4 | a stop plugin's claim that `<skill_id>` (an active handler) should cease activity — the orchestrator dispatches `<skill_id>:stop` and the owner's stop handler runs |
| `fallback` | OVOS-FALLBACK-1 §6.3 | a fallback plugin's claim that `<skill_id>` (a registered fallback handler) is willing to handle the utterance — the orchestrator dispatches `<skill_id>:fallback` and the handler runs |
| `common_query` | OVOS-COMMON-QUERY-1 §3 | a common-query plugin's self-addressed match (`Match.skill_id` is the plugin's own `pipeline_id`) — the orchestrator dispatches `<pipeline_id>:common_query` and the plugin's bundled handler speaks the answer it selected during `match` |

This specification fixes only the registry mechanism (reservation
listing); the per-name semantics are owned by the reserving
specification. Other specifications MAY reserve further names by
adding rows to this table in a revision of this specification.

A plain skill (§7.0) subscribes to a reserved-name dispatch topic
via framework convention rather than OVOS-INTENT-4 registration —
the reserved name is not registrable. The normal skill path
(INTENT-4-registered intents) and the reserved-name path share the
same `<skill_id>:<intent_name>` dispatch shape; no dispatch
mechanics change.

---

## 8. Handler-lifecycle messages

The handler — whether a skill or a plugin-bundled handler — is a
black box. **Third-party handler code carries no obligation under
this specification.** The handler-lifecycle trio is emitted by the
**orchestrator** that invokes the handler, wrapping the invocation:
`start` before the call, then `complete` on normal return or `error`
on exception. The handler itself does not emit anything.

The three broadcast notification topics are the
**handler-lifecycle trio**:

| Topic | Meaning |
|-------|---------|
| `ovos.intent.handler.start` | The orchestrator is about to invoke the handler. |
| `ovos.intent.handler.complete` | The handler returned normally. |
| `ovos.intent.handler.error` | The handler raised. |

Each trio Message is produced via OVOS-MSG-1 §5.1 `forward` from the
originating dispatch Message — `context` (including `session`) is
preserved unchanged. The trio is broadcast so any observer (loggers,
transcript viewers, analytics, fallback chains) can subscribe.

### 8.1 Order and obligations

For each accepted dispatch, the **orchestrator MUST** emit:

- `ovos.intent.handler.start` immediately before invoking the
  handler;
- **exactly one** of `ovos.intent.handler.complete` (on normal
  return) or `ovos.intent.handler.error` (on exception) immediately
  after the invocation returns or raises.

A dispatch produces exactly one `start` and exactly one terminal
event. The orchestrator owns the trio in full; no third-party code
is required to participate.

### 8.2 Payload

Each lifecycle message's `data`:

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music"
}
```

`ovos.intent.handler.error` adds an `exception` field:

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "exception": "RuntimeError: Spotify is not configured"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The `skill_id` of the handler that was dispatched. |
| `intent_name` | string | yes | The intent the handler was dispatched for. |
| `exception` | string | `error` only | Human-readable description of the failure raised by the handler. |

Implementations **MAY** include additional fields but consumers
**MUST NOT** require them.

### 8.3 Handler timeout

The orchestrator **MAY** bound handler execution by a
deployment-defined time. If the handler has not returned within the
bound, the orchestrator **MUST** emit `ovos.intent.handler.error`
with an `exception` field indicating timeout, then **MUST** proceed
to emit `ovos.utterance.handled` (§9.5).

The orchestrator **MUST NOT** re-emit the dispatch Message for the
same match. Re-dispatch is not defined by this specification.

---

## 9. Utterance-layer messages

This specification formalizes the following utterance-layer bus events.
All travel in standard OVOS-MSG-1 envelopes; routing follows the
single-flip model of OVOS-MSG-1 §5.2.

### 9.1 The utterance-layer entry point — `ovos.utterance.handle`

The orchestrator subscribes to **`ovos.utterance.handle`**, the
**utterance-layer entry-point topic** produced by any component
that wants to feed an utterance into the assistant — a listener,
a chat bridge, a CLI, a test harness, a remote-peer client.
Receiving on this topic kicks off the lifecycle of §6.

Payload shape:

```json
{
  "utterances": ["turn off the lights"],
  "lang": "en-US"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of strings | yes | One or more candidate utterance strings. |
| `lang` | string | no | BCP-47 language tag of the utterance. **Present only when the producer authoritatively knows the content language** (e.g. a chat client emitting text it locally typed in `de-DE`, or an audio service emitting text from an STT decoder run in `en-US`). When absent, the content language is **not authoritatively known**; producers **MUST NOT** synthesize a value on the wire. On receipt, the orchestrator **MUST** resolve the language once from OVOS-SESSION-1 §3.2 evidence fields and pass the resolved tag to every plugin's `match` call (§4) — a single resolution point keeps all stages matching in the same language; plugins MAY refine but MUST NOT re-derive independently. |

`ovos.utterance.handle` is the only entry topic name this
specification recognizes. A conformant orchestrator subscribes to
this topic; a conformant producer emits to it.

### 9.2 `ovos.intent.matched`

Emitted by the orchestrator after a plugin's `match` returns
non-`None`, before the dispatch (§7) goes out. Broadcast (no
`destination`).

Payload:

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "utterance": "play the beatles",
  "slots": { "query": "the beatles" },
  "pipeline_id": "template-high"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The handler's `skill_id`. |
| `intent_name` | string | yes | The matched intent name. |
| `lang`, `utterance`, `slots` | as §7.1 | — | Same semantics as the dispatch payload. |
| `pipeline_id` | string | yes | The `pipeline_id` of the plugin that produced the match. |

`ovos.intent.matched` is a **notification**, not a dispatch.
Consumers **MUST NOT** treat receipt as permission or instruction
to run a handler — handler invocation happens via the dispatch
topic (§7).

### 9.3 `ovos.intent.unmatched`

Emitted by the orchestrator when pipeline iteration completed
with no plugin claiming the utterance. Broadcast.

```json
{
  "utterances": ["turn off the lights"],
  "lang": "en-US"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of strings | no | The candidate utterance list that no plugin matched, as it stood after the utterance-transformer chain. Included for observability; consumers **MUST NOT** re-submit it without explicit user intent. |
| `lang` | string | no | BCP-47 tag from the entry-topic Message (§9.1), if it was present. Absent when the entry-topic carried no `lang`. |

Both fields are optional. An observer that receives no fields still
knows no plugin matched — the topic name alone is normative.

This message **MUST** be followed immediately by
`ovos.utterance.handled` (§9.5).

This is the **intent-layer failure** signal. It is distinct from
a handler-layer error (§8): `ovos.intent.unmatched` means "no
plugin claimed"; `ovos.intent.handler.error` means "a handler
ran and raised."

### 9.4 The dispatch topic

`<skill_id>:<intent_name>` — see §7.

### 9.5 `ovos.utterance.handled`

The **universal end-marker** for an utterance. Emitted by the
orchestrator on every terminal path — cancellation, no-match,
matched-and-handler-completed, matched-and-handler-errored,
matched-and-handler-timed-out.

Broadcast. Payload **MAY** be empty.

A conformant orchestrator **MUST** emit exactly one
`ovos.utterance.handled` per entry-topic Message (§9.1).
Multiple emissions for one utterance are malformed; zero is
malformed.

### 9.6 `ovos.utterance.speak` — natural-language response

`ovos.utterance.speak` is the **natural-language output exit point** of
the pipeline — the symmetric counterpart to the `ovos.utterance.handle`
entry point (§9.1). Together they define the natural-language I/O
boundary of the voice assistant: human speech (or text) arrives on the
entry topic; the assistant's natural-language response departs on this
topic.

A handler emits `ovos.utterance.speak` to deliver a natural-language
response string for the assistant to convey to the user. What the
deployment does with the Message downstream — TTS rendering, audio
queueing, playback, chat display — is out of scope for this
specification and is defined by the output-path companion specification.
A deployment with no audio output (a text-only chat bridge, a test
harness) receives the same `ovos.utterance.speak` Message as an
audio-capable deployment.

**Payload:**

```json
{
  "utterance": "It is currently 22 degrees and sunny.",
  "lang": "en-US"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterance` | string | yes | The natural-language response string. |
| `lang` | string | no | BCP-47 tag of the response language. When absent, the output stage resolves language from the session per OVOS-SESSION-1 §3.2. |
| `listen` | bool | no | When `true`, the handler expects a follow-up utterance after this response is delivered; the output stage re-opens the user input channel once delivery completes. Absent or `false` means no follow-up is expected. The output-side behaviour this triggers is defined by the output-path companion specification. |

**Derivation and session propagation.** A handler **MUST** derive each
`ovos.utterance.speak` emission from the dispatch Message (§7) it
received, per MSG-1 §5 derivation semantics. This carries
`context.session` and `context.skill_id` forward automatically —
the output layer (dialog-transformer chain OVOS-TRANSFORM-1 §3.5,
TTS, delivery) can read the session and attribute the response
without additional wire fields. An `ovos.utterance.speak`
Message that does not derive from a dispatch is non-conformant.

**Multiplicity and ordering.** A handler **MAY** emit zero or more
`ovos.utterance.speak` Messages. Zero is permitted — a handler that
acts silently (playing a sound, toggling a device, queuing media) is
conformant. When a handler emits multiple, the order of emission is the
intended delivery order; the output stage **SHOULD** preserve it.

**Broadcast.** `ovos.utterance.speak` carries no `destination` — it is
broadcast. Any output component subscribed to the topic may consume it.

---

## 10. Per-pipeline introspection

Each pipeline plugin owns the set of intents it currently has
loaded. To let consumers (UIs, developer tools, debug viewers,
other plugins) discover that set at runtime, this specification
defines a pull-query / scatter-response pattern keyed on
`pipeline_id`.

A pipeline plugin with bundled handlers **SHOULD** publish the set
of `intent_name` values it owns through the query topic below.
Observers and introspection tools rely on this index to enumerate
every handler in the deployment; without it, plugin-owned handlers
are invisible to deployment-wide tooling that walks OVOS-INTENT-4
only. This is **not** OVOS-INTENT-4 registration — it is a
one-way declaration of "these are the intent_names I dispatch on."

### 10.1 Query and response topics

| Topic | Direction | Carries |
|-------|-----------|---------|
| `ovos.pipeline.<pipeline_id>.intents.list` | request | empty payload (or filters, see §10.3) |
| `ovos.pipeline.<pipeline_id>.intents.list.response` | reply | the plugin's currently-loaded intent set |

A consumer that wants the loaded intents of a specific pipeline
**MUST** emit on the per-`pipeline_id` topic above. There is **no
aggregate query** — a consumer that wants the intent set of every
loaded plugin emits one query per `pipeline_id` it cares about and
aggregates the responses itself.

The `pipeline_id` in the topic is the same identifier carried by
`session.pipeline` (§5) and by `context["pipeline_id"]` on any observed dispatch (§3.1); a consumer that has already observed
a `pipeline_id` from any of these sources can query it directly.

### 10.2 Response payload

The plugin **MUST** emit the response derived via `reply`
(OVOS-MSG-1 §5.2), so that routing metadata is preserved and
the response reaches the requester through any layer-2
transport. The response carries the currently-loaded intent
set:

```json
{
  "pipeline_id": "template-high",
  "intents": [
    {
      "intent_name": "play_music",
      "skill_id": "music.skill",
      "lang": "en-US"
    },
    {
      "intent_name": "stop_music",
      "skill_id": "music.skill",
      "lang": "en-US"
    }
  ]
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `pipeline_id` | string | yes | The responding plugin's id. |
| `intents` | array | yes | Currently-loaded intents (possibly empty). |
| `intents[].intent_name` | string | yes | Intent identifier. |
| `intents[].skill_id` | string | yes | The `skill_id` of the handler. For a self-matching plugin, equals its `pipeline_id`. |
| `intents[].lang` | string | yes | The language the intent is registered for. |

A plugin **MAY** include additional per-intent fields (engine
metadata, confidence thresholds, sample templates) but consumers
**MUST NOT** require them.

### 10.3 Filters

The request payload **MAY** carry filters:

```json
{ "lang": "en-US", "skill_id": "music.skill" }
```

When a filter is present, the plugin **SHOULD** restrict its
response to intents matching every filter field. Unknown filter
keys are ignored (forward-compatible).

### 10.4 Pull-query is the source of truth

Pipeline plugins **MAY** broadcast load-time announcements (e.g.
when a skill registers new intents the plugin recompiles), but
consumers that need accurate state **MUST** query
`ovos.pipeline.<pipeline_id>.intents.list` and **MUST NOT** assume
that any prior broadcast reached them. The bus is asynchronous,
has no delivery guarantees, and a consumer that started after a
load event missed the announcement.

A plugin **MUST** respond to every query it observes for its own
`pipeline_id`. A consumer that receives no response within a
deployment-defined timeout **MAY** retry; persistent silence
indicates the plugin is not loaded.

**Under a split orchestrator** (§2), a pipeline plugin is loaded
into exactly one orchestrator process — typically the
utterance-handling process that owns the match round of §6. That
process answers the per-`pipeline_id` query for plugins it hosts.
Sibling processes do not respond on its behalf. A query is
broadcast; the consumer accepts the single response that arrives
from the hosting process.

---

## 11. Conformance

### A **deployment** **SHOULD**:

- load at least one pipeline plugin that consumes OVOS-INTENT-4
  registrations when skills emitting keyword or template intents are
  present; without such a plugin those intents never match.

### An **orchestrator** **MUST**:

- subscribe to the utterance-layer entry topic
  `ovos.utterance.handle` (§9.1);
- run every received utterance through the lifecycle of §6
  exactly once;
- emit `ovos.utterance.handled` (§9.5) exactly once per
  utterance, regardless of which terminal path was taken;
- iterate `session.pipeline` in order (§6.2) and stop at
  the first plugin returning a non-`None` `Match`;
- skip unknown `pipeline_id`s without failing the utterance (§5);
- emit `ovos.intent.unmatched` when no plugin claimed (§9.3);
- emit `ovos.intent.matched` (§9.2) on every successful claim,
  before the dispatch;
- dispatch on `<match.skill_id>:<match.intent_name>` per §7;
- handle a plugin exception by logging and continuing to the
  next plugin (§6.2), not by failing the utterance;
- emit the handler-lifecycle trio (§8) wrapping every handler
  invocation: `start` before the call, then exactly one of
  `complete` (on normal return) or `error` (on exception or
  timeout, §8.3) after;
- remain able to accept and process new `ovos.utterance.handle`
  messages while a handler is running (§6.5).

### A **pipeline plugin** **MUST**:

- expose a `match(utterances, lang, session) → Match | None` operation
  (§4);
- when claiming, return a `Match` with `skill_id`, `intent_name`,
  and `lang` per §4 — never a partial or speculative claim;
- bear a `pipeline_id` distinct from any other loaded plugin's
  id (§3);
- **respond** to every `ovos.pipeline.<own_pipeline_id>.intents.list`
  query with a §10.2 response payload describing its currently-loaded
  intent set (§10.4) — pull-query is the source of truth that
  consumers rely on.

### A **handler** (skill or plugin-bundled)

Handlers carry **no normative obligation** under this
specification. The orchestrator owns the handler-lifecycle trio
(§8) and the dispatch envelope (§7). A handler is an opaque
callable; the spec binds the orchestrator that invokes it, not the
handler itself.

---

## See also

- *Bus Message Specification* (OVOS-MSG-1) — the envelope, the
  single-flip routing model, the shared topic-component identifier
  rule (§2.1.1), the `session` carrier that holds `pipeline`.
- *Session Specification* (OVOS-SESSION-1) — the wire shape of
  `session`, the registry mechanism under which this specification
  claims the `pipeline` field, and the deployment-default fallback
  rule for omitted / empty `session.pipeline`.
- *Intent and Entity Registration Bus Contract* (OVOS-INTENT-4) —
  the registration wire format plugins consume (when they choose
  to).
- *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  concept and the orchestrator role.
