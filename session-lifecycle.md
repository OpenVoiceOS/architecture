# Session Lifecycle and State Ownership Specification

**Spec ID:** OVOS-SESSION-2 · **Version:** 1 · **Status:** Draft

This document defines **who owns session state**, **when it is
mutated**, **how it propagates between client and assistant**, and
**how a conversation resumes** after arbitrary elapsed time or
across an orchestrator restart.

It is the lifecycle complement to OVOS-SESSION-1, which defines
the wire shape of the `session` carrier and explicitly defers
lifecycle (SESSION-1 §1 / §6 non-goals). Where SESSION-1 fixes
*what `session` looks like on the bus*, this specification fixes
*who is allowed to mutate it, when, and how its state survives
across utterances*.

The central principle is **statelessness with one named
exception**: the orchestrator and the message bus hold no
authoritative session state for any session except the reserved
`session_id == "default"` (SESSION-1 §3.1), which the orchestrator
fully owns. Every other session is **client-owned**: a participant
on the user side of the bus boundary holds the authoritative state
for its own `session_id` and persists it however it chooses. This
arrangement makes conversations resumable after arbitrary elapsed
time, lets an orchestrator restart without losing client-side
continuity, and lets multiple orchestrators in a deployment serve
the same session without coordination.

It builds on four companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  routing keys, `forward` / `reply` / `response` derivations,
  and the asynchronous nature of the bus this spec relies on;
- the *Session Carrier Wire Shape Specification* (OVOS-SESSION-1) —
  the JSON shape of `session`, the field registry, the
  `session_id == "default"` reservation, and the
  omission-not-`null` rule;
- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the per-utterance lifecycle, the
  `Match.updated_session` channel that match-phase session
  mutations travel on, and the universal end-marker
  `ovos.utterance.handled`;
- the *Intent Context Specification* (OVOS-CONTEXT-1) and the
  *Active Handlers and Interactive Response Specification*
  (OVOS-CONVERSE-1) — consumers of this spec's projection
  mandate (§2.4); they hold session-keyed state and therefore
  fall under the rule that all such state lives in
  session-resident fields.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **state-ownership model** (§2) — who holds session state,
  what is permitted to mutate it, and what the projection
  mandate requires of components that maintain cross-utterance
  state;
- the **per-utterance round** (§3) — the flow of a single
  utterance from client emission to assistant response;
- the **client-side merge rules** (§4) — how a client tracks
  session updates from assistant-emitted Messages, keyed on
  `session_id` alone;
- the **resumption semantics** (§5) — what makes a conversation
  resumable across arbitrary elapsed time or orchestrator
  restart;
- the **default-session ownership rule** (§6) — the one
  exception to statelessness, codifying current ovos-core
  `SessionManager.default_session` behaviour;
- **conformance** (§7) for the four roles (bus, orchestrator,
  component, client).

This specification does **not** define:

- **the wire shape of `session`** — owned by OVOS-SESSION-1;
- **the semantics of any individual session field** — owned by
  the field's claiming specification;
- **persistence format** for client-held session state — every
  client chooses its own storage (in-process memory, local
  database, encrypted blob, etc.);
- **session authentication or authorization** — a layer-2
  concern built on top of OVOS-MSG-1 §3.4. A client that sends
  *any* `session_id` it wants is conformant; trust boundaries
  are someone else's spec;
- **cross-client session sharing** — two clients holding the
  same `session_id` would race on session state; coordination
  is out of scope;
- **session migration between orchestrators** — handled
  implicitly by the §2.2 stateless rule (any orchestrator can
  serve any named session because no orchestrator holds state
  for it);
- **lifecycle observability events** (`ovos.session.start` /
  `.end` or similar) — deferred to a future observability
  specification if needed; not required for correctness here.

---

## 2. The state-ownership model

### 2.1 The bus is stateless transport

The message bus (OVOS-MSG-1) holds no session state. It
delivers Messages and does not interpret their `session`
carrier. A Message dropped, delayed, or duplicated by the bus
has no effect on any party's session state beyond what that
party reads off the Message.

This is structural: OVOS-MSG-1 §3 defines the bus as a
publish/subscribe substrate with no per-session machinery.
Stateless transport is what makes the rest of this spec
possible.

### 2.2 The orchestrator is stateless for named sessions

For every `session_id` other than the reserved `"default"`
(SESSION-1 §3.1), the orchestrator **MUST NOT** maintain
authoritative session state across utterances. Each inbound
Message carrying such a `session_id` brings its own session
snapshot, which the orchestrator processes during the
utterance lifecycle (PIPELINE-1 §6) — mutating in place only
at transformer and pipeline boundaries (§2.6 below) — and
emits forward on its response Messages. Between utterances on
a named session, the orchestrator holds no state for that
session.

The orchestrator **MAY** maintain a transient per-utterance
cache (the inbound session it is currently processing, the
Match it has produced, etc.); such caches are utterance-scoped
and discarded at end-of-utterance. They are **not**
cross-utterance state and **MUST NOT** be relied upon by any
component as durable.

A consequence: any orchestrator in a deployment can serve any
inbound Message on any named session. No coordination is
required because no orchestrator holds state another would
need to consult. Cross-orchestrator load-balancing, failover,
and restart are all transparent at the session layer.

### 2.3 The orchestrator owns `session_id == "default"`

The reserved value `session_id == "default"` (SESSION-1 §3.1)
marks a Message as originating from the local device. The
orchestrator **MUST** maintain persistent in-process state for
this single session, keyed under `"default"` — the
authoritative default-session store.

This is the one exception to §2.2. The local device is a
client of the orchestrator that runs in the same process tree
as the orchestrator itself; making the orchestrator hold its
state is the simplest representation of that physical
co-location. It also formalises the existing ovos-core pattern
of `SessionManager.default_session` (see APPENDIX §5).

Behaviour rules for the default-session store are in §6.

### 2.4 The projection mandate

A component (a pipeline plugin, a transformer, any other
participant) that holds `session_id`-keyed state **across**
utterances **MUST** project that state into a session-resident
field it owns (claimed under SESSION-1 §2.1). The projection
flows through the pipeline plugin's `Match.updated_session`
channel (PIPELINE-1 §4.2): the component writes its state into
the field on every match it produces, the orchestrator carries
the updated session forward, and the next utterance arrives
with the state already populated for the component to read.

The projection mandate is what makes resumption (§5) work
uniformly across all components. A component that holds
authoritative state outside the session carrier is, by
definition, not resumption-safe — its state evaporates on
orchestrator restart and is invisible to other orchestrators
in a multi-orchestrator deployment.

Transient **in-utterance** caches are permitted (a plugin may
build helper structures for its match call, a transformer may
batch context lookups), but cross-utterance state **MUST** be
projected. The distinction is lifecycle-scoped: anything that
needs to outlive the current `ovos.utterance.handled`
(PIPELINE-1 §9) must be in `session`.

### 2.5 Clients own their named sessions

A **client** is any participant on the user side of the bus
boundary — the local device for the default session, a remote
peer over a layer-2 substrate for any other session. A client
that uses a named `session_id` (anything other than
`"default"`) **MUST** be its own authoritative store for that
session's state. Persistence format and lifetime are entirely
the client's choice: in-process memory for the duration of a
process, a SQLite file across restarts, an encrypted blob in
the user's cloud, anything else.

The client is free to send any `session_id` with any
`session` value at any time (§1: authentication is out of
scope). A client emitting Messages on a session it has never
seen before, on a session belonging to a different participant
it has decided to impersonate, or on a session with a
fabricated `session_id` is all wire-conformant — the
orchestrator processes them identically. Trust is a layer-2
concern.

### 2.6 When session mutates in place

In-place session mutations during an utterance lifecycle
happen only at these boundaries:

- **transformer boundaries** — any of OVOS-TRANSFORM-1's six
  hooks (audio, utterance, metadata, intent, dialog, TTS);
- **pipeline boundaries** — a pipeline plugin's `match` may
  return a `Match.updated_session` per PIPELINE-1 §4.2; the
  orchestrator MUST apply it as `session = match.updated_session
  or session` immediately on a non-null match;
- **handler boundaries** — a dispatched handler (skill or
  plugin-bundled handler per PIPELINE-1 §7.0) MAY mutate
  session in-place; its emissions via `forward` / `reply` /
  `response` (OVOS-MSG-1 §5) carry the mutated session
  forward.

Bus events emitted *outside* these boundaries — the
asynchronous, normal-event-handler kind that any component may
emit at any time — **MUST NOT** be expected to mutate session
state in the current utterance. The bus is asynchronous and
not part of the utterance lifecycle (§2.1).

A bus-emitted Message that carries a mutated session **MAY**
affect subsequent utterances on that session (its updated
session is received by the client and merged per §4), but
**MUST NOT** be expected to affect the utterance during which
it was emitted.

---

## 3. The per-utterance round

A **round** is the full processing of one inbound utterance,
from client emission to assistant response. Every round follows
the same shape:

### 3.1 Client emission

The client emits an inbound Message — typically
`ovos.utterance.handle` (PIPELINE-1 §9) carrying an utterance
to process, but the rule applies to any inbound Message kind.
The Message carries the client's current local session in
`Message.context.session` (OVOS-MSG-1 §4, SESSION-1 §2).

The session the client emits is its **authoritative local
state** at emission time. It includes every session-resident
field the client has accumulated from prior rounds, including
fields populated by components under the §2.4 projection
mandate that the client has merged per §4.

### 3.2 Assistant processing

The assistant runs the OVOS-PIPELINE-1 §6 utterance lifecycle:

- transformer chains (audio, utterance, metadata) run before
  pipeline iteration;
- pipeline plugins iterate per `session.pipeline`;
- the first plugin to return a non-null `Match` wins;
- `session = match.updated_session or session` is applied
  immediately on a non-null match (PIPELINE-1 §4.2);
- post-match transformer chain (intent) runs;
- the orchestrator dispatches `<owner_id>:<intent_name>`
  (PIPELINE-1 §7);
- the dispatched handler runs and may mutate session in-place
  per §2.6;
- post-handler transformer chains (dialog, TTS) run on the
  handler's emissions;
- the orchestrator emits the universal end-marker
  `ovos.utterance.handled` (PIPELINE-1 §9) with the final
  session.

### 3.3 Assistant emissions

During and after the lifecycle, the assistant emits zero or
more response Messages: `ovos.intent.matched`, the
handler-lifecycle trio (`ovos.intent.handler.start` /
`.complete` / `.error`), any number of handler-derived speak /
forward / reply Messages, dialog-transformer outputs,
TTS-stage Messages, and the terminal `ovos.utterance.handled`.

Each of these Messages carries `Message.context.session` at the
emission point. By construction:

- pipeline-plugin emissions during match carry the inbound
  session (the plugin has not yet returned a `Match`);
- post-match emissions carry the post-`Match.updated_session`
  snapshot;
- handler-derived emissions carry the dispatch session as
  mutated by the handler;
- the end-marker carries the final session for the round.

### 3.4 Client reception

The client observes the response Messages and updates its local
session per §4 — keyed on its own `session_id`, the only filter
the spec recognises. After the round ends, the client's local
session reflects the post-utterance state and is used as the
inbound session for its next emission.

---

## 4. Client-side merge rules

These rules are intentionally minimal and permissive. The spec
fixes what is *available* for a client to merge from; the
client decides what to *use*.

### 4.1 Session_id is the only key

A client **MAY** update its local session tracking from any
Message it observes carrying a `session_id` matching its own.
`session_id` uniquely identifies the channel; it is the only
key that matters for client-side session merging. No other
matching predicate is normative.

A client that does not update its session from observed
Messages is also conformant (it discards continuity, but the
wire contract does not require any client to track state). This
spec only specifies what is *available* on the wire; clients
are free to ignore it.

### 4.2 Every assistant-emitted Message carries an updated session

By the rules of §3.2 and PIPELINE-1 §4.2 / §5, every
assistant-emitted Message carries a valid session at its
emission point. A client that adopts any one such Message's
session has, by definition, a snapshot consistent with the
assistant's view at that point in the round.

Adopting the **latest received** session is the simplest
client policy and is conformant. More elaborate policies
(field-by-field merge across multiple observed Messages,
selecting by emitter identity, etc.) are also conformant; the
spec does not prescribe.

### 4.3 `ovos.utterance.handled` is the canonical convergence point

When a client wants a single canonical "this round is over"
snapshot, the PIPELINE-1 §9 universal end-marker
`ovos.utterance.handled` is the recommended adoption point: it
is emitted exactly once per utterance on every terminal path,
and the session it carries is the assistant's final state for
the round.

A client may:

- adopt only the end-marker's session (simplest model;
  intermediate updates ignored until the round ends);
- adopt incrementally per §4.1 throughout the round;
- combine both — incremental tracking with end-marker
  canonical override.

All three are conformant. The choice is observability vs.
latency vs. complexity.

---

## 5. Resumption semantics

### 5.1 Resumption is implicit

A client **MAY** re-emit a previously-used `session_id` with
its locally-held session state at any time. There is no
"session resume" handshake on the wire: the inbound Message's
session IS the resume. The orchestrator processes it via the
stateless rule of §2.2 — it neither knows nor cares whether the
session has been seen before, was last seen seconds or years
ago, or was previously served by a different orchestrator.

Resumption works because the orchestrator carries no
cross-utterance state for the session (§2.2), and the client
carries the full state on the inbound Message (§2.5).

### 5.2 What is resumption-safe

Resumption-safe state is the union of:

- every field claimed under SESSION-1 §3's registry (language
  signals, `pipeline`, `intent_context`, `active_handlers`,
  `response_mode`, the transformer chains, the blacklists,
  `site_id`, and any future-claimed field);
- the projected state of every component bound by §2.4 — by
  construction, since the projection mandate requires
  cross-utterance state to live in session-resident fields.

Resumption is **field-by-field**: a client that drops or
replaces individual fields gets the corresponding fall-back
behaviour at the consumer (SESSION-1 §2.5: omitted fields
resolve to deployment defaults). A client that resumes a
session minus `intent_context` enters with a fresh declarative
state but retains the rest.

### 5.3 What is not resumption-safe

Anything not session-resident is not resumption-safe. This is
exactly the space §2.4 forbids components from holding:
authoritative cross-utterance state outside session is a
conformance violation. A conformant deployment has nothing
non-resumption-safe by construction.

A transient in-utterance cache (§2.4) is, by definition,
gone at end-of-utterance and therefore trivially absent from
any future round; resumption neither preserves nor needs it.

---

## 6. The default-session ownership rule

### 6.1 Persistent orchestrator-held state

The orchestrator **MUST** maintain persistent in-process state
for `session_id == "default"`, keyed under `"default"`. This is
the **default-session store**.

The default-session store is updated continuously during
orchestrator operation:

- every inbound Message bearing `session_id == "default"` (or
  equivalent — omitted session, empty session, explicit
  default, all per SESSION-1 §3.1) is merged into the store as
  part of the utterance lifecycle;
- every outbound Message on the default session derives from
  the store, so handlers and components see the current
  default state on the dispatch they receive;
- session mutations during the lifecycle (transformer
  boundaries §2.6, `Match.updated_session` per PIPELINE-1
  §4.2, in-handler mutations) propagate into the store
  through the standard derivation chain.

The merge semantics for inbound default-session Messages are
"last-write-wins per field": an inbound field value replaces
the stored value for that field. Omitted inbound fields leave
the stored field unchanged. This is symmetric with §4's
permissive client-side merge.

### 6.2 Restart semantics

The default-session store is **process-local**. An orchestrator
restart discards it; the default session reverts to deployment
defaults (the empty session, with every field falling back per
SESSION-1 §2.5). Components keyed on the default session lose
their state.

This is acceptable for the default session by design: the
default session represents the local device, which is typically
co-located with the orchestrator process. A restart of the
orchestrator is a restart of the device's voice stack;
discarding the default-session state matches user expectation
of a "fresh start" after restart.

Deployments that want default-session persistence across
restarts MAY implement orchestrator-side persistence (writing
the store to disk on shutdown, restoring on start). This is
deployment policy; the spec does not require it.

### 6.3 Component reliance on default-session continuity

Components consuming the default session **MAY** rely on the
orchestrator's continuity within a single deployment lifetime.
A pipeline plugin that holds session-keyed state and projects
to a session-resident field per §2.4 finds that, for the
default session, the projected field is reliably preserved
across utterances (the orchestrator's store holds it). For a
named session, the same projection is preserved only as long
as the client holds the session locally.

CONVERSE-1's `session.response_mode` wait window, CONTEXT-1's
`session.intent_context` entries, and any future
session-projected state are therefore **reliable on the local
device** (default session) and **best-effort on remote peers**
(named sessions held by the client). For named sessions the
client's local persistence policy determines whether wait
windows actually survive a several-day gap.

### 6.4 Default-session sync to clients

The orchestrator **MAY** emit the default-session state as a
diagnostic on a deployer-defined topic (the existing
`ovos.session.sync` / `ovos.session.update_default` pattern in
current ovos-core is one such mechanism). This is informative;
no spec-level consumer is defined.

---

## 7. Conformance

### 7.1 Bus

The message bus **MUST** be stateless with respect to session.
It **MUST NOT** interpret, mutate, persist, or special-case
`Message.context.session` for any reason. Delivery is the bus's
contract; session is opaque to it.

### 7.2 Orchestrator

An orchestrator that claims conformance to this specification
**MUST**:

- treat every named session (`session_id != "default"`) as
  stateless per §2.2 — no cross-utterance state held outside
  what the inbound Message brings;
- hold the default session as persistent in-process state per
  §6, with the merge / derive / restart semantics of §6.1 /
  §6.2;
- apply in-place session mutations only at the boundaries of
  §2.6 (transformer, pipeline-match, handler);
- propagate session forward unchanged on every Message
  derivation per OVOS-MSG-1 §5 and SESSION-1 §4, except where
  the §2.6 boundaries dictate mutation;
- emit the universal end-marker `ovos.utterance.handled`
  carrying the final round session (PIPELINE-1 §9), as the
  client-side convergence point of §4.4.

An orchestrator **MUST NOT** require any client to declare
session-start / session-end / session-id-allocation events
before processing an inbound Message. Clients send what they
send; the orchestrator processes what arrives.

### 7.3 Component

A component (pipeline plugin, transformer, dispatched handler,
introspection observer) **MUST**:

- project any `session_id`-keyed state it holds across
  utterances into a session-resident field it claims under
  SESSION-1 §2.1 (the §2.4 mandate);
- write that state into the field via the appropriate
  in-utterance pathway — `Match.updated_session` for pipeline
  plugins per PIPELINE-1 §4.2, direct mutation for
  transformers and handlers per §2.6;
- treat transient in-utterance caches as utterance-scoped,
  discarding them at end-of-utterance;
- read its state from `session` on every inbound Message,
  rather than from a cross-utterance internal store.

A component **MUST NOT** rely on bus events (the asynchronous
kind that fire outside the utterance lifecycle) to mutate
session state in the current utterance (§2.6). It MAY emit such
events to communicate with other components; their effect on
session, if any, lands on subsequent utterances.

### 7.4 Client

A **client** (any participant on the user side of the bus
boundary that uses a named `session_id`) **MUST**:

- hold its own authoritative session state for the
  `session_id` values it uses, per §2.5;
- include that state in `Message.context.session` on every
  inbound Message it emits.

A client **MAY**:

- update its local session from any Message it observes per §4;
- choose any persistence format, lifetime, and lifecycle for
  its local session;
- re-emit a previously-used `session_id` at any time per §5;
- send any `session_id` and any `session` value, including
  fabricated ones (trust is a layer-2 concern, §2.5).

A client **MUST NOT**:

- expect the orchestrator to remember any session state for it
  between rounds — every round MUST be self-sufficient via the
  inbound session.

### 7.5 Default-session client

The local device, which uses `session_id == "default"`, is a
special-case client. Because the orchestrator owns the default
session per §6, the local device **MAY** omit `Message.context.session`
or emit `session: {}` (SESSION-1 §3.1's equivalent forms) and
rely on the orchestrator's stored state. This is the only place
this spec recognizes a client that does not carry its own
authoritative state — and only because the orchestrator's
default-session store *is* that state for the local device.

---

## 8. Non-goals

This specification deliberately does not:

- define a **session-store protocol** for client-side
  persistence — every client picks its own;
- define **session authentication or authorization** — layer-2
  on top of MSG-1 §3.4;
- define **cross-client session sharing or coordination** —
  two clients holding the same `session_id` race; out of scope;
- define **session migration between orchestrators** — handled
  implicitly by §2.2 statelessness; any orchestrator can serve
  any named session because none holds state for it;
- define **lifecycle observability events** (`ovos.session.start`
  / `.end`, etc.) — deferred to a future spec if needed; not
  required for correctness;
- define **per-field encryption or selective field exposure** —
  the session is one JSON object and is propagated as a whole;
- define **default-session persistence across orchestrator
  restart** — §6.2 makes restart-loss explicit; deployer
  policy if desired;
- replace **OVOS-SESSION-1** — that spec owns the wire shape
  and the field registry; this spec owns the lifecycle that
  rides on top.
