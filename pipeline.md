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
- the **`session.pipeline`** field (§5) — how a session
  chooses which plugins and in what order;
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
  OVOS-MSG-1 §4. `session.pipeline` is one internal field
  this spec prescribes (§5); other internal fields are deferred to
  a future session specification.
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

## 5. `session.pipeline`

This specification claims **`pipeline`** as a session field per
OVOS-SESSION-1 §2.1: an array of `pipeline_id` strings, ordered,
session-scoped, propagating with the session, with the
deployment-default-fallback absence rule (an omitted, empty, or
absent `session.pipeline` resolves to the deployment-configured
default at consumption, per OVOS-SESSION-1 §2.5).

Example:

```json
{
  "session": {
    "pipeline": [
      "padatious-high",
      "adapt-high",
      "padatious-medium",
      "adapt-medium",
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
the orchestrator falls back to the deployment-configured default
pipeline. If no default is configured, the utterance proceeds to
no-match (`complete_intent_failure`, §9.3).

Different sessions may carry different `pipeline`. This is how a
deployment provides different behaviour to different participants —
for example, a remote-peer session may carry a restricted pipeline
that excludes destructive plugins.

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
  `source` is the orchestrator.

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
| `lang` | string | no | BCP-47 language tag of the utterance. If absent, the orchestrator falls back per OVOS-SESSION-1 §3.2. |

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
  "pipeline_id": "padatious-high"
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
  "pipeline_id": "padatious-high",
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
