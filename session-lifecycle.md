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

It builds on five companion specifications:

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
- the *Transformer Plugin Specification* (OVOS-TRANSFORM-1) —
  defines six transformer hooks (audio, utterance, metadata,
  intent, dialog, TTS) that are normative session-mutation
  boundaries per §2.6;
- the *Intent Context Specification* (OVOS-CONTEXT-1) and the
  *Active Handlers and Interactive Response Specification*
  (OVOS-CONVERSE-1) — both elect the §2.4 SHOULD-project
  pathway for their cross-utterance state (intent-context
  entries, active-handler list, response-mode wait window
  respectively), making it resumption-safe by construction.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **state-ownership model** (§2) — who holds session state,
  what is permitted to mutate it, and when components SHOULD
  project their cross-utterance state into session-resident
  fields vs hold it internally with best-effort resumption
  semantics;
- the **client-side merge rules** (§3) — how a client tracks
  session updates from assistant-emitted Messages, keyed on
  `session_id` alone;
- the **resumption semantics** (§4) — what makes a conversation
  resumable across arbitrary elapsed time or orchestrator
  restart;
- the **default-session ownership rule** (§5) — the one
  exception to statelessness, codifying current ovos-core
  `SessionManager.default_session` behaviour;
- **conformance** (§6) for the four roles (bus, orchestrator,
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
  is out of scope. A layer-2 system that routes Messages to
  specific clients (using MSG-1 `source` / `destination`)
  can disambiguate which client owns a given `session_id`, but
  that routing policy is a layer-2 responsibility;
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
means "interact with the device-local session." The orchestrator
**MUST** maintain persistent in-process state for this single
session, keyed under `"default"` — the authoritative
default-session store.

This is the one exception to §2.2. The local device is a
client of the orchestrator that runs in the same process tree
as the orchestrator itself; making the orchestrator hold its
state is the simplest representation of that physical
co-location. It also formalises the existing ovos-core pattern
of `SessionManager.default_session` (see APPENDIX §5).

Behaviour rules for the default-session store are in §5.

### 2.4 Project state into session when practical; plugin-internal state is permitted

A component (a pipeline plugin, a transformer, any other
participant) that holds `session_id`-keyed state **across**
utterances **SHOULD** project that state into a session-resident
field it owns (claimed under SESSION-1 §2.1) when projection is
practical. Projection flows through the pipeline plugin's
`Match.updated_session` channel (PIPELINE-1 §4.2) or through
in-place mutation at transformer / handler boundaries (§2.6).
Projected state is **resumption-safe by construction** — it
travels with the session, survives orchestrator restart, and
moves transparently across multi-orchestrator deployments.

A component **MAY** instead hold authoritative cross-utterance
state internally when projection is impractical. Realistic
examples:

- a **language-model plugin** holding a multi-turn conversation
  transcript that is too large to ride on every session-carrying
  Message;
- a **media pipeline plugin** holding playback positions, queued
  playlists, or user-favourite catalogues backed by external
  service APIs;
- a **personalization component** holding learned preferences,
  trained classifiers, or any state tied to local model
  artefacts;
- any plugin whose state is intrinsically tied to external
  resources (sockets, processes, files, accounts) that cannot
  be serialised into a JSON session field meaningfully.

A component that takes this path:

- **owns its state lifecycle in full** — persistence (or not),
  expiry, eviction, multi-orchestrator coordination if the
  deployment has multiple orchestrators, and any privacy or
  access-control concerns the state raises;
- offers **best-effort resumption with no normative guarantee**.
  A user resuming "unpause the music" months later may or may
  not get a useful reaction — the plugin may have evicted the
  playback state, the underlying media process may no longer
  exist, the user's playlist may have changed, or the plugin
  may have persisted the state and handle the resume cleanly.
  The spec does not bind the outcome of any plugin-internal
  resumption attempt;
- MUST NOT expect other components or clients to know its
  state exists or to compensate for its absence.

The CONVERSE-1 converse plugin (§5 there) is one example of a
plugin that chooses to project all its cross-utterance state —
the response-mode wait window is small, simple, and naturally
session-coupled, so the SHOULD-project path is the obvious fit.
LLM, media, and personalization plugins typically pick the
plugin-internal path. Both are conformant; the choice is per
plugin.

Transient **in-utterance** caches (helper structures built
during a single `match` call, batched lookups within a
transformer chain) are always permitted regardless of projection
choice — they are utterance-scoped, discarded at
end-of-utterance, never cross-utterance state.

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

Trust and authorization are layer-2 concerns (§1); this spec
places no constraint on what `session_id` or `session` value a
client sends.

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
session is received by the client and merged per §3), but
**MUST NOT** be expected to affect the utterance during which
it was emitted.

---

## 3. Client-side merge rules

These rules are intentionally minimal and permissive. The spec
fixes what is *available* for a client to merge from; the
client decides what to *use*.

### 3.1 Session_id is the only key

A client **MAY** update its local session tracking from any
Message it observes carrying a `session_id` matching its own.
`session_id` uniquely identifies the channel; it is the only
key that matters for client-side session merging. No other
matching predicate is normative.

### 3.2 Every assistant-emitted Message carries an updated session

Per PIPELINE-1 §4.2 and §5, every assistant-emitted Message
carries a valid session at its emission point. A client that
adopts any one such Message's session has a snapshot consistent
with the assistant's view at that point in the round.

Adopting the **latest received** session is the simplest client
policy. More elaborate policies (field-by-field merge, selecting
by emitter identity) are also conformant; the spec does not
prescribe.

### 3.3 `ovos.utterance.handled` is the canonical convergence point

When a client wants a single canonical "round is over" snapshot,
the PIPELINE-1 §9 universal end-marker `ovos.utterance.handled`
is the recommended adoption point: emitted exactly once per
utterance on every terminal path, carrying the assistant's final
session for the round. A client may also adopt incrementally per
§3.1, or combine both; all are conformant.

---

## 4. Resumption semantics

### 4.1 Resumption is implicit

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

### 4.2 What is resumption-safe

Resumption-safe state is every field in the SESSION-1 §3
registry, plus the projected state of any component that
elected §2.4's SHOULD-project pathway — resumption-safe by
construction since it lives in session-resident fields.

Resumption is **field-by-field**: omitted fields resolve to
deployment defaults at the consumer (SESSION-1 §2.1). A client
that resumes without `intent_context` enters with a fresh
context but retains every other field.

### 4.3 Plugin-internal state — best-effort resumption

State held internally by a component per §2.4's MAY-internal
pathway is governed by the holding component's own design. The
spec defines no protocol for plugin-internal state; the
plugin chooses what to persist, what to evict, and what
"resume" means for its own state. A client cannot expect
parity across components:

- a chat-history-holding LLM plugin may resume a months-old
  conversation seamlessly because it persisted the transcript;
- the same client trying to resume "unpause the music" may find
  the media plugin's playback state long gone, because the
  plugin evicted it after a deployer-configured TTL or because
  the underlying media process restarted;
- a personalization component may retain learned preferences
  forever, may drop them on restart, or may rebuild them on
  demand — entirely up to the plugin.

This is not a defect of the spec; it is the cost of allowing
plugins to hold state too large or too coupled to external
resources to project into session. Plugins that want resumption
parity with the projection pathway can adopt projection (§2.4);
plugins that prefer internal state accept best-effort
resumption.

A transient in-utterance cache (§2.4) is, by definition, gone
at end-of-utterance and therefore trivially absent from any
future round; resumption neither preserves nor needs it.

---

## 5. The default-session ownership rule

### 5.1 Persistent orchestrator-held state

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
  §5.2, in-handler mutations) propagate into the store
  through the standard derivation chain.

The merge semantics for inbound default-session Messages follow
SESSION-1 §2.1's omission rule: **omitted inbound fields leave
the stored field unchanged** (the stored value is the
orchestrator's last authoritative value for that field); a
**present inbound field replaces the stored value** for that
field. This is the natural complement to the stateless-named-
session rule: the default-session store fills the role the
client plays for named sessions.

### 5.2 Restart semantics

The default-session store is **process-local**. An orchestrator
restart discards it; the default session reverts to deployment
defaults (the empty session, with every field falling back per
SESSION-1 §2.1). Components keyed on the default session lose
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

### 5.3 Component reliance on default-session continuity

Components **MAY** rely on default-session continuity within a
single deployment lifetime: a §2.4-projected field (e.g.
`session.response_mode`, `session.intent_context`) is reliably
preserved across utterances because the orchestrator's store
holds it. For named sessions the same field is preserved only
as long as the client holds it locally — best-effort on remote
peers.

### 5.4 Default-session sync to clients

The orchestrator **MAY** emit the default-session state as a
diagnostic on a deployer-defined topic, so that interested
observers can track default-session evolution without processing
every response Message. No normative topic name or consumer is
defined here; this is deployment policy.

---

## 6. Conformance

### 6.1 Bus

The message bus **MUST** be stateless with respect to session.
It **MUST NOT** interpret, mutate, persist, or special-case
`Message.context.session` for any reason. Delivery is the bus's
contract; session is opaque to it.

### 6.2 Orchestrator

An orchestrator that claims conformance to this specification
**MUST**:

- treat every named session (`session_id != "default"`) as
  stateless per §2.2 — no cross-utterance state held outside
  what the inbound Message brings;
- hold the default session as persistent in-process state per
  §5, with the merge / derive / restart semantics of §5.1 /
  §5.2;
- apply in-place session mutations only at the boundaries of
  §2.6 (transformer, pipeline-match, handler);
- propagate session forward unchanged on every Message
  derivation per OVOS-MSG-1 §5 and SESSION-1 §4, except where
  the §2.6 boundaries dictate mutation;
- emit the universal end-marker `ovos.utterance.handled`
  carrying the final round session (PIPELINE-1 §9), as the
  client-side convergence point of §3.3.

An orchestrator **MUST NOT** require any client to declare
session-start / session-end / session-id-allocation events
before processing an inbound Message. Clients send what they
send; the orchestrator processes what arrives.

### 6.3 Component

A component that holds `session_id`-keyed state across
utterances **SHOULD**:

- project that state into a session-resident field it claims
  under SESSION-1 §2.1 (per §2.4), via the appropriate
  in-utterance pathway — `Match.updated_session` for pipeline
  plugins per PIPELINE-1 §4.2, direct mutation for
  transformers and handlers per §2.6;
- on every inbound Message, read its state from `session`
  rather than from a cross-utterance internal store.

A component **MAY** instead hold cross-utterance state
internally per §2.4 when projection is impractical, in which
case it MUST take full responsibility for state lifecycle and
accept best-effort resumption (§4.3).

A component **MUST NOT** rely on bus events (the asynchronous
kind that fire outside the utterance lifecycle) to mutate
session state in the current utterance (§2.6). It MAY emit such
events to communicate with other components; their effect on
session, if any, lands on subsequent utterances.

### 6.4 Client

A **client** (any participant on the user side of the bus
boundary that uses a named `session_id`) **MUST**:

- hold its own authoritative session state for the
  `session_id` values it uses, per §2.5;
- include that state in `Message.context.session` on every
  inbound Message it emits.

A client **MAY** update its local session per §3, choose any
persistence format and lifetime, and re-emit a previously-used
`session_id` at any time (§4).

A client **MUST NOT**:

- expect the orchestrator to remember any session state for it
  between rounds — every round MUST be self-sufficient via the
  inbound session.

### 6.5 Default-session client

The local device, which uses `session_id == "default"`, is a
special-case client. Because the orchestrator owns the default
session per §5, the local device **MAY** omit `Message.context.session`
or emit `session: {}` (SESSION-1 §3.1's equivalent forms) and
rely on the orchestrator's stored state. This is the only place
this spec recognizes a client that does not carry its own
authoritative state — and only because the orchestrator's
default-session store *is* that state for the local device.

---

## 7. Non-goals

See §1 for the full list of non-goals. This section adds one
clarification: **default-session persistence across orchestrator
restart** is not defined here. §5.2 makes restart-loss
explicit and intentional; persistence is deployer policy if
desired.
