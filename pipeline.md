# Utterance Lifecycle and Pipeline Specification

**Spec ID:** OVOS-PIPELINE-1 · **Version:** 2 · **Status:** Draft

This document defines the **utterance lifecycle** — the path an
utterance takes from the moment it enters the assistant to the moment
the assistant is done with it — and the **pipeline plugin**
abstraction the **orchestrator** runs to decide what to do with each
utterance.

It is the orchestrator-side companion to OVOS-INTENT-3 and
OVOS-INTENT-4. Those specifications define what an intent *is* and
how a skill puts an intent on the bus; this one defines what the
orchestrator does with utterances and the contract every pipeline
plugin conforms to.

It builds on three companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  routing keys, session carrier, and derivations every Message
  defined here travels in;
- the *Intent Definition Specification* (OVOS-INTENT-3) — defines
  the *orchestrator* and the intent / handler model;
- the *Intent and Entity Registration Bus Contract* (OVOS-INTENT-4)
  — the wire format pipeline plugins consume to learn what intents
  to match (when they choose to).

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
- the **dispatch** topic shape (§7) — `<owner_id>:<intent_name>`;
- the **handler-lifecycle trio** (§8) —
  `ovos.intent.handler.start` / `.complete` / `.error`;
- the **utterance-layer bus events** (§9) —
  the utterance entry topic (§9.1, name deferred),
  `ovos.intent.matched`, `complete_intent_failure`,
  `ovos.utterance.handled`;
- **conformance** (§10).

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
  OVOS-SESSION-1 §2.1 registry mechanism.
- **per-plugin behavioural specs** — plugins have no behavioural
  contract beyond §4. A `converse` plugin, a `fallback` plugin, a
  persona plugin, a language-model plugin, a chatbot plugin: each
  defines itself.

---

## 2. The orchestrator and the pipeline plugin

The **orchestrator** (OVOS-INTENT-3 §6.1) is the logical role that
consumes the utterance-layer entry topic (§9.1; topic name
deferred to a future spec), iterates plugins per session, emits
dispatch and terminal events, and guarantees the universal
end-marker `ovos.utterance.handled`.
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

A **pipeline plugin** is a third-party component identified by an
opaque `pipeline_id` — an arbitrary, deployment-unique string. The
orchestrator loads some number of plugins at startup; how it
discovers and instantiates them is a deployment concern. Each
plugin exposes one operation to the orchestrator (§4) and is
otherwise a black box.

From the orchestrator's perspective, "plugin" and "skill" are
indistinguishable as handler owners. Both are black-box third-party
components. The only difference is where the handler lives:

- a skill's handler is reached via a `<skill_id>:<intent_name>`
  dispatch topic — the skill registered the intent (OVOS-INTENT-4)
  and owns the handler;
- a plugin's bundled handler is reached via a
  `<pipeline_id>:<intent_name>` dispatch topic — the plugin matched
  the utterance and owns the handler itself.

From outside either case, the assistant responded. The user does
not know or care which component answered.

Plugins are diverse by design. A deployment may load plugins that
consume OVOS-INTENT-4 registrations and match against keyword or
template intents, plugins that consume no registrations and match
by their own internal rules (such as language-model-backed
personas), plugins that always claim with a fallback response, or
anything else. The contract is just `match`.

A deployment running skills that emit OVOS-INTENT-4 keyword or
template intent registrations **SHOULD** load at least one plugin
that consumes those registrations; otherwise those intents will
never match. Whether to load such a plugin is a deployment choice,
not a spec-level requirement.

---

## 3. Pipeline plugins

A pipeline plugin is identified by an opaque **`pipeline_id`** —
an arbitrary string. The orchestrator's loaded-plugin set is a
mapping `pipeline_id → plugin instance`; the orchestrator does
not interpret the `pipeline_id` string beyond using it as a key.

Constraints on `pipeline_id` strings:

- Non-empty.
- Bound by OVOS-MSG-1 §2.1.1 (identifiers used as topic
  components): no `:`, no `.`, no whitespace, ASCII letters /
  digits / `_` / `-` only. The constraint is necessary because
  `pipeline_id` may appear as the owner in the dispatch topic shape
  `<owner_id>:<intent_name>` (§7) and per-pipeline introspection
  topics (§10) build on the same identifier.
- Unique within a deployment's loaded-plugin set.

A plugin **MAY** appear in a session's pipeline more than once
under different `pipeline_id`s if the plugin chooses to expose
multiple matching modes (for example, a strict mode and a
permissive mode). The orchestrator treats each `pipeline_id` as a
distinct stage.

### 3.1 Plugin self-identification on emission

A pipeline plugin **MUST** set `Message.context["pipeline_id"]` on
**every Message it emits to the bus**, to its own `pipeline_id`.
This is the plugin-side analogue of the skill rule in
OVOS-INTENT-4 §3.1: it makes plugin-originated traffic
attributable to its emitter without parsing topic names or `data`
payloads.

The rule binds independent of the topic. It applies to bus events
a plugin emits on its own initiative (background telemetry,
diagnostics, plugin-defined topics), and — crucially for downstream
specs — to events on shared topics owned by other specifications.
For example, OVOS-CONTEXT-1 §5.2 reads `context["pipeline_id"]` to
attribute a plugin-emitted `ovos.context.set` to its owning
plugin; without this rule that attribution has no wire-level
source.

**Stamp rule.** A plugin **MUST** set
`Message.context["pipeline_id"]` to its own identity on **every
Message it places on the bus** and on every Message it **modifies
in place** before that Message proceeds. "Places on the bus"
covers all the ways a plugin can cause a Message to appear on the
bus:

- a fresh emission (the plugin constructs and emits a new
  Message);
- a derivation it performs and then emits — `Message.forward(...)`,
  `Message.reply(...)`, or `Message.response(...)` from a prior
  Message the plugin received or held (OVOS-MSG-1 §5). The
  derivation mechanism is irrelevant to the stamp rule: the
  plugin is the origin of the *resulting* Message-on-wire even
  when context propagated from upstream, and the resulting
  Message **MUST** carry `context["pipeline_id"]` set to the
  plugin's id, overwriting whatever inherited
  `context["pipeline_id"]` may have been there from an earlier
  pipeline plugin in a multi-plugin chain.

Modify-in-place covers the case where the plugin mutates an
existing Message (its `context`, its `data`, the session it
carries — for example, a CONTEXT-1 §5.3 direct mutation) without
itself causing a fresh emission; the next emitter still propagates
the modified Message, and `context["pipeline_id"]` must reflect
the plugin that mutated.

The combined effect: whenever the plugin's hands have been on a
Message that subsequently appears on the bus, `context["pipeline_id"]`
reflects it.

**Coexistence with other identity keys.** When a plugin emits via
`Message.forward` / `.reply` / `.response` (OVOS-MSG-1 §5) from a
prior Message that already carries `context["skill_id"]` from an
upstream dispatch (or any of the six `<type>_transformer_ids` keys
of OVOS-TRANSFORM-1 §1.3 from upstream transformer stages), the
inherited keys are preserved by the derivation rule and **not**
stripped. Each names a different component in the chain that
produced the Message; the plugin additionally stamps its own
`context["pipeline_id"]`. Attribution consumers
(OVOS-CONTEXT-1 §5.2, audit / telemetry observers) apply a
lifecycle-position precedence — see OVOS-CONTEXT-1 §5.2 — to pick
a single owner when they need one.

`Message.context["pipeline_id"]` is the plugin's **self-attribution**.
Mirroring the `context["skill_id"]` / `data["skill_id"]` distinction
of OVOS-INTENT-4 §3.1, a topic's `data` schema may also carry
`pipeline_id` as the **subject** of the message (the plugin a
query is filtered against, the plugin being described, etc.); a
consumer reading `data.pipeline_id` is reading a subject, not a
self-attribution.

#### Orchestrator-side enforcement

The orchestrator (or any component that loads pipeline plugins)
**SHOULD** intercept / decorate the plugin's emit pathway at load
time so non-compliant plugin code cannot emit a Message that lacks
or misstates `context["pipeline_id"]`. This places the discipline
on the plugin-loading infrastructure rather than on every plugin
author, mirroring the skill-loader enforcement of OVOS-INTENT-4
§3.1.

A consumer that needs to attribute a plugin-emitted Message
**MUST** read `context["pipeline_id"]` — it **MUST NOT** infer the
plugin from `source`, `data` fields, or topic name. A Message
without `context["pipeline_id"]` arriving on a topic that requires
plugin attribution (per the topic's owning spec) is malformed at
that topic's layer; the topic's spec defines the rejection
behaviour.

---

## 4. The match contract

A plugin exposes one operation to the orchestrator:

```
match(utterance, session) → Match | None
```

Inputs:

- `utterance` — a **non-empty list of candidate strings**. The
  list typically originates from the entry topic (§9.1) and may
  have been modified by the utterance-transformer
  chain (OVOS-TRANSFORM-1 §3.2) before reaching the plugin. A
  plugin **MUST** accept this shape: a list of one or more
  candidate transcripts, in no particular order, all in the same
  language. A plugin is free to consider all candidates, only the
  first, or any subset; the orchestrator does not prescribe how
  candidates are weighted.
- `session` — the session carrier from `context.session` of the
  utterance Message (OVOS-MSG-1 §4, OVOS-SESSION-1).

Output: either `None` (decline) or a `Match` object with the
fields below.

### 4.1 The `Match` shape

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `owner_id` | string | yes | The `skill_id` of the skill that owns the handler, **or** the `pipeline_id` of the plugin itself if the plugin bundles its own handler. |
| `intent_name` | string | yes | An opaque non-empty string that, together with `owner_id`, names the handler to invoke. For skill-owned matches this is the intent name the skill registered. For plugin-owned matches this is whatever label the plugin chose for this response. |
| `captures` | object (string→string) | yes | The capture map (§4.3). MAY be empty. |
| `utterance` | string | no | The specific candidate string from the input list that won the match. |

The orchestrator interprets a non-`None` return as a definitive
claim. It does not score, rank, or rerank matches across plugins —
**first match wins** (§6). A plugin that wants to express
uncertainty must return `None` and let a later plugin claim.

### 4.2 Plugins are side-effect-free during `match`

A plugin **MUST NOT** emit Messages from inside its `match`
operation. `match` is a pure decide-or-decline call. Any side
effects — activating a skill, updating internal session state,
storing conversational history — happen after the orchestrator
has confirmed the claim wins, in the plugin's own handler
(reached via the dispatch topic of §7) or in some other plugin-
internal step the spec does not constrain.

This is the difference between "matching" and "running": the
orchestrator may call `match` on several plugins before one
claims; plugins that took side effects from declined matches
would corrupt each other's state.

### 4.3 The capture map

`Match.captures` is a `{string: string}` mapping (the same shape
OVOS-INTENT-3 §7 defines for template / keyword intent slots).

For skill-owned matches against intents the plugin previously
consumed from OVOS-INTENT-4 registrations, the capture map keys
are the slot names (template intents) or vocabulary names (keyword
intents) of the matched intent.

For plugin-owned matches, the capture map is whatever the plugin
chooses to surface. It **MAY** be empty.

The orchestrator does not interpret the capture map; it forwards
it to the dispatched handler.

---

## 5. Session fields owned by this specification

This specification claims four session fields per OVOS-SESSION-1
§2.1: one **positive** ordering field (§5.1 `pipeline`) and three
**negative** filtering fields (§5.2 `blacklisted_pipelines`, §5.3
`blacklisted_skills`, §5.4 `blacklisted_intents`). All four are
session-scoped, propagate with the session under OVOS-SESSION-1 §4,
and follow the deployment-default-fallback absence rule of
OVOS-SESSION-1 §2.5: an omitted, empty, or absent field resolves at
consumption to the deployment-configured default.

§5.5 fixes how the positive and negative fields compose when both
are set; §5.6 is an informative note on layer-2 authorization use.

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

If `session.pipeline` is absent or empty (per OVOS-SESSION-1 §2.5),
the orchestrator falls back to the **default-session pipeline**: the
pipeline configured for the reserved `session_id == "default"`
session (OVOS-SESSION-1 §3.1). The default-session pipeline is owned
and maintained by the orchestrator and represents what the
deployment runs when no preference is expressed. If the default
session itself has no `pipeline` configured, the utterance proceeds
to no-match (`complete_intent_failure`, §9.3).

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
back to the deployment default per OVOS-SESSION-1 §2.5. A
producer with no pipelines to deny **SHOULD** omit the field
rather than emit `[]`, per the wire-weight guidance of
OVOS-SESSION-1 §3.4.

### 5.3 `session.blacklisted_skills`

An unordered array of `skill_id` strings (OVOS-INTENT-3) whose
intents **MUST NOT** be matched for this session.

The contract is **two-tier**:

1. A pipeline plugin **SHOULD NOT** return a `Match` whose
   `owner_id` (§7.1) is a `skill_id` listed here. A plugin's
   internal handling of would-match-but-blacklisted candidates is
   **not specified** — it MAY skip the candidate before scoring,
   suppress its score below a match threshold, route to a
   plugin-internal default-handler, or anything else — as long as
   the returned `Match` does not name a blacklisted skill.
2. A pipeline plugin that does not implement filtering is **not
   conformant** with this field. The orchestrator **MUST** therefore
   act as backstop: after a plugin returns a candidate `Match`, the
   orchestrator **MUST** check `Match.owner_id` against
   `blacklisted_skills` and, if listed, **MUST** treat the match as
   if the plugin had declined — continue iteration to the next
   plugin per §6.2. No bus event is emitted for backstop filtering;
   it is observable only as a non-match.

Empty-array semantics match §5.2: `[]` is wire-equivalent to
omission. A producer with no skills to deny **SHOULD** omit the
field.

### 5.4 `session.blacklisted_intents`

An unordered array of fully-qualified `<owner_id>:<intent_name>`
strings (the dispatch-topic shape of §7) whose specific intents
**MUST NOT** be matched for this session.

The contract is identical in shape to §5.3 (two-tier:
plugin-SHOULD + orchestrator-MUST-backstop), with the comparison
performed against the candidate `Match`'s dispatch identity
`<Match.owner_id>:<Match.intent_name>`.

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
`<owner_id>:<intent_name>` denies both — there is no per-language
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
directly to no-match (`complete_intent_failure`, §9.3). The
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
entry topic                              ← entry (§9.1)
   │
   ├─ session retrieval; pipeline read from session (§5)
   │
   ├─ for pipeline_id in session.pipeline:
   │     plugin = loaded_plugins[pipeline_id]     # skip if not loaded
   │     match = plugin.match(utterance, session)
   │     if match is not None:
   │         ovos.intent.matched                  (§9.2)
   │         dispatch on <match.owner_id>:<match.intent_name>  (§7)
   │         (handler runs; emits lifecycle trio §8)
   │         ovos.utterance.handled               (§9.5)
   │         break
   │
   └─ if no plugin matched:
         complete_intent_failure                  (§9.3)
         ovos.utterance.handled                   (§9.5)
```

### 6.2 First-match-wins iteration

For each utterance, the orchestrator **MUST**:

- iterate `session.pipeline` in order;
- for each `pipeline_id`, call `match` on the corresponding loaded
  plugin (skipping unknown identifiers, §5);
- stop at the **first plugin** that returns a non-`None` `Match`;
- if no plugin returns a `Match`, emit `complete_intent_failure`
  (§9.3).

A plugin that raises an exception during `match` is treated as if
it returned `None`. The orchestrator **MUST** continue to the next
plugin and **SHOULD** log the exception. A single plugin's bug
does not fail the whole utterance.

### 6.3 Plugins do not see each other's matches

A plugin receives the same utterance every other plugin in the
pipeline received; it has no access to what an earlier plugin
tried or why it declined. Cross-plugin coordination belongs in
`session` (OVOS-MSG-1 §4) or in plugin-side out-of-band state
keyed on `session.session_id` (per OVOS-MSG-1 §5.4 —
"no central correlation, no central state").

### 6.4 Terminal events

Every utterance terminates in exactly one of two ways, each
followed by the universal end-marker `ovos.utterance.handled`:

| Outcome | Sequence of utterance-layer events |
|---------|------------------------------------|
| Matched by a plugin | `ovos.intent.matched` → dispatch + (handler trio §8) → `ovos.utterance.handled` |
| No plugin matched | `complete_intent_failure` → `ovos.utterance.handled` |

If a dispatched handler emits `ovos.intent.handler.error` (§8)
instead of `.complete`, the orchestrator still emits
`ovos.utterance.handled` afterwards. The "every utterance
terminates with `ovos.utterance.handled`" invariant holds across
all paths.

---

## 7. Dispatch

When a plugin's `match` returns a non-`None` `Match`, the
orchestrator dispatches the matched handler by emitting a Message
on the topic:

```
<owner_id>:<intent_name>
```

where `<owner_id>` is `Match.owner_id` (a `skill_id` or a
`pipeline_id`) and `<intent_name>` is `Match.intent_name`.

The `:` between the two segments is the only one in the topic:
`skill_id`, `pipeline_id`, and `intent_name` are bound by
OVOS-MSG-1 §2.1.1 (no `:`, no `.`, no whitespace), so the split is
unambiguous.

### 7.1 Routing and payload

The dispatch Message's `context` (OVOS-MSG-1 §4):

- `session` is propagated from the originating utterance;
- `source` and `destination` follow the single-flip routing model
  (OVOS-MSG-1 §5.2) — the orchestrator derives the dispatch via
  `reply`, so `destination` is the original utterance emitter and
  `source` is the orchestrator;
- when `<owner_id>` is a `skill_id`, the orchestrator **MUST**
  stamp `context["skill_id"]` to that `skill_id`. This carries
  the skill's identity forward into every Message the handler
  emits via `forward` (OVOS-MSG-1 §5.1) — the skill inherits the
  context, satisfying OVOS-INTENT-4 §3.1 by construction. When
  `<owner_id>` is a `pipeline_id` (plugin-bundled handler), the
  orchestrator **MUST NOT** stamp `context["skill_id"]` —
  plugin-bundled handlers identify themselves via `pipeline_id`,
  not `skill_id`.

Any Message the skill subsequently emits **MUST** carry
`context["skill_id"]` matching the `<owner_id>` of the dispatch
that invoked it (OVOS-INTENT-4 §3.1). Because the orchestrator
stamps the dispatch context and skills derive their emissions
from it via `forward` / `reply`, this match is automatic — a
skill that emits a Message whose `context["skill_id"]` differs
from the dispatch is non-conformant, and the orchestrator
**SHOULD** detect and log such drift if it is in a position to
do so (loader-side interception per OVOS-INTENT-4 §3.1).

The dispatch Message's `data`:

```json
{
  "owner_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "utterance": "play the beatles",
  "captures": { "query": "the beatles" }
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `owner_id` | string | yes | The `Match.owner_id` — the topic's prefix, repeated for the handler's convenience. |
| `intent_name` | string | yes | The `Match.intent_name` — the topic's suffix. |
| `lang` | string | yes | The language the utterance was recognized in (`data.lang`, OVOS-MSG-1 §4.2). |
| `utterance` | string | yes | The candidate string that won the match. |
| `captures` | object (string→string) | yes | The capture map (§4.3). MAY be empty. |

### 7.2 Subscription discipline

Each handler subscribes to exactly its own
`<owner_id>:<intent_name>` topic. A skill subscribes to topics
under its own `skill_id`; a plugin that bundles its own handlers
subscribes to topics under its own `pipeline_id`. Because each
topic is unique to one handler, the bus delivers the dispatch
only to the intended consumer.

A consumer that receives a dispatch on a topic it should not be
listening to (a configuration bug) **MUST NOT** run the handler
and **SHOULD** log the discrepancy. The orchestrator does not
police subscriptions.

### 7.3 In-process equivalence

When the handler-owning component (skill or plugin) runs in the
same process as the orchestrator, the orchestrator **MAY** invoke
the handler directly without serializing the dispatch Message
over a transport — provided every external observer sees the
same `<owner_id>:<intent_name>` dispatch and the same
handler-lifecycle trio (§8) it would have seen for an
out-of-process handler. This uniformity is what makes a
deployment portable across in-process and out-of-process handler
arrangements.

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
  "owner_id": "music.skill",
  "intent_name": "play_music"
}
```

`ovos.intent.handler.error` adds an `exception` field:

```json
{
  "owner_id": "music.skill",
  "intent_name": "play_music",
  "exception": "RuntimeError: Spotify is not configured"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `owner_id` | string | yes | The handler-owning component's id (skill_id or pipeline_id). |
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

This specification formalizes five utterance-layer bus events.
All travel in standard OVOS-MSG-1 envelopes; routing follows the
single-flip model of OVOS-MSG-1 §5.2.

### 9.1 The utterance-layer entry point

The orchestrator subscribes to an **utterance-layer entry-point
topic** produced by any component that wants to feed an utterance
into the assistant — a listener, a chat bridge, a CLI, a test
harness, a remote-peer client. Receiving on this topic kicks off
the lifecycle of §6.

The **topic name itself is not prescribed by this
specification**; it is the subject of a separate spec covering
audio-input ↔ assistant-core wire contracts. Current deployments
use **`recognizer_loop:utterance`** as the entry topic; a
conformant orchestrator MAY subscribe to that name for
compatibility while the entry-point spec is in flight.

Payload shape on the entry topic (current convention):

```json
{
  "utterances": ["turn off the lights"],
  "lang": "en-US"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of strings | yes | One or more candidate utterance strings. |
| `lang` | string | no | BCP-47 language tag of the utterance. If absent, the orchestrator **MUST** fall back to `session.lang` (OVOS-SESSION-1 §3.2.1) — the session-scoped user-preference signal — and if that too is absent, to the deployment default. The consolidation guidance of OVOS-SESSION-1 §3.2.7 is informative for downstream stages but does not apply to this entry-point field, which has only the two normative sources named here. |

What **is** normative in this specification is the *behaviour
after entry*: every utterance the orchestrator accepts proceeds
through §6, terminates with exactly one `ovos.utterance.handled`
(§9.5), and carries the universal lifecycle obligations of
§§7–8. The entry topic's exact name and payload shape are not.

### 9.2 `ovos.intent.matched`

Emitted by the orchestrator after a plugin's `match` returns
non-`None`, before the dispatch (§7) goes out. Broadcast (no
`destination`).

Payload mirrors the dispatch payload (§7.1) plus the matching
plugin's id:

```json
{
  "owner_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "utterance": "play the beatles",
  "captures": { "query": "the beatles" },
  "pipeline_id": "template-high"
}
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `owner_id`, `intent_name`, `lang`, `utterance`, `captures` | as §7.1 | yes | Same fields as the dispatch payload. |
| `pipeline_id` | string | yes | The `pipeline_id` of the plugin that produced the match. |

`ovos.intent.matched` is a **notification**, not a dispatch.
Consumers **MUST NOT** treat receipt as permission or instruction
to run a handler — handler invocation happens via the dispatch
topic (§7).

### 9.3 `complete_intent_failure`

Emitted by the orchestrator when pipeline iteration completed
with no plugin claiming the utterance. Broadcast. Payload
**MAY** carry the original utterance data for observability.

This message **MUST** be followed immediately by
`ovos.utterance.handled` (§9.5).

This is the **intent-layer failure** signal. It is distinct from
a handler-layer error (§8): `complete_intent_failure` means "no
plugin claimed"; `ovos.intent.handler.error` means "a handler
ran and raised."

### 9.4 The dispatch topic

`<owner_id>:<intent_name>` — see §7.

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

---

## 10. Per-pipeline introspection

Each pipeline plugin owns the set of intents it currently has
loaded. To let consumers (UIs, developer tools, debug viewers,
other plugins) discover that set at runtime, this specification
defines a pull-query / scatter-response pattern keyed on
`pipeline_id`.

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
`session.pipeline` (§5) and by `Match.owner_id` when the plugin
owns its own handler (§7); a consumer that has already observed
a `pipeline_id` from any of these sources can query it directly.

### 10.2 Response payload

The plugin **MUST** reply with the currently-loaded intent set:

```json
{
  "pipeline_id": "template-high",
  "intents": [
    {
      "intent_name": "play_music",
      "owner_id": "music.skill",
      "lang": "en-US"
    },
    {
      "intent_name": "stop_music",
      "owner_id": "music.skill",
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
| `intents[].owner_id` | string | yes | The owning component (`skill_id` or `pipeline_id` when plugin-owned). |
| `intents[].lang` | string | yes | The language the intent is registered for. |

A plugin **MAY** include additional per-intent fields (engine
metadata, confidence thresholds, sample templates) but consumers
**MUST NOT** require them.

### 10.3 Filters

The request payload **MAY** carry filters:

```json
{ "lang": "en-US", "owner_id": "music.skill" }
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

### An **orchestrator** **MUST**:

- subscribe to the utterance-layer entry topic (§9.1) — name
  deferred to a future spec; current deployments use
  `recognizer_loop:utterance`;
- run every received utterance through the lifecycle of §6
  exactly once;
- emit `ovos.utterance.handled` (§9.5) exactly once per
  utterance, regardless of which terminal path was taken;
- iterate `session.pipeline` in order (§6.2) and stop at
  the first plugin returning a non-`None` `Match`;
- skip unknown `pipeline_id`s without failing the utterance (§5);
- emit `complete_intent_failure` when no plugin claimed (§9.3);
- emit `ovos.intent.matched` (§9.2) on every successful claim,
  before the dispatch;
- dispatch on `<match.owner_id>:<match.intent_name>` per §7;
- handle a plugin exception by logging and continuing to the
  next plugin (§6.2), not by failing the utterance;
- emit the handler-lifecycle trio (§8) wrapping every handler
  invocation: `start` before the call, then exactly one of
  `complete` (on normal return) or `error` (on exception or
  timeout, §8.3) after.

### A **pipeline plugin** **MUST**:

- expose a `match(utterance, session) → Match | None` operation
  (§4);
- be **side-effect-free during `match`** (§4.2) — no Messages
  emitted, no state changed beyond what is needed to decide;
- when claiming, return a `Match` with `owner_id` and
  `intent_name` per §4 — never a partial or speculative claim;
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

### Non-goals

The following are explicitly outside this specification: plugin
loading and discovery; any pre-pipeline utterance transformation
or cancellation chain; ASR n-best ranking semantics within
plugins; per-plugin behavioural specs; the `session` object's
wire shape and field set (owned by OVOS-SESSION-1).

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
