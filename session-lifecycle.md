# Session Lifecycle and State Ownership Specification

**Spec ID:** OVOS-SESSION-2 · **Version:** 1 · **Status:** Draft

This document defines **who owns session state**, **when it may be
mutated**, **how it propagates between client and assistant**, and
**how a conversation resumes** after arbitrary elapsed time or across
an orchestrator restart.

It is the lifecycle complement to OVOS-SESSION-1, which defines the
wire shape of the `session` carrier and explicitly defers lifecycle to
this specification. Where SESSION-1 fixes *what `session` looks like
on the bus*, this specification fixes *who is allowed to mutate it,
when, and how its state survives across utterances*.

The central principle is **statelessness with one named exception**: the
orchestrator holds no authoritative session state for any session except
`session_id == "default"` (SESSION-1 §3.1), which the orchestrator
fully owns. Every other session is **client-owned**. This makes
conversations resumable after arbitrary elapsed time, lets an
orchestrator restart without losing client continuity, and lets
multiple orchestrators serve the same session without coordination.

Dependencies:

- **OVOS-MSG-1** — the Message envelope, `forward` / `reply` /
  `response` derivations, and the asynchronous bus substrate.
- **OVOS-SESSION-1** — the `session` JSON shape, field registry,
  `session_id == "default"` reservation, and omission-not-null rule.
- **OVOS-PIPELINE-1** — utterance lifecycle, `Match.updated_session`
  channel, and the universal end-marker `ovos.utterance.handled`.
- **OVOS-TRANSFORM-1** — the six transformer hooks that are normative
  session-mutation boundaries per §2.6.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **state-ownership model** (§2) — who holds session state, when
  it mutates, and what components SHOULD project vs hold internally;
- **client-side merge rules** (§3) — how a client tracks session
  updates from assistant-emitted Messages;
- **resumption semantics** (§4) — what makes a conversation resumable
  across arbitrary elapsed time or orchestrator restart;
- the **default-session ownership rule** (§5) — the one exception to
  statelessness;
- **conformance** (§6) for the four roles: bus, orchestrator,
  component, client.

This specification does **not** define:

- **the wire shape of `session`** — owned by OVOS-SESSION-1;
- **semantics of any individual session field** — owned by the
  field's claiming specification;
- **persistence format** for client-held session state;
- **session authentication or authorization** — a layer-2 concern
  (OVOS-MSG-1 §3.4);
- **cross-client session sharing** — routing policy is a layer-2
  responsibility;
- **session migration between orchestrators** — handled implicitly by
  the §2.2 stateless rule;
- **lifecycle observability events** (`ovos.session.start` / `.end`
  or similar) — not required for correctness here.

---

## 2. The state-ownership model

### 2.1 The bus is stateless transport

The message bus **MUST NOT** interpret, persist, or mutate
`Message.context.session`. A dropped, delayed, or duplicated Message
has no effect on any party's session state beyond what that party reads
off the Message.

### 2.2 The orchestrator is stateless for named sessions

For every `session_id` other than `"default"`, the orchestrator
**MUST NOT** maintain authoritative session state across utterances.
Each inbound Message brings its own session snapshot; the orchestrator
processes it during the utterance lifecycle (PIPELINE-1 §6), mutating
in place only at the permitted boundaries of §2.6, and emits the result
on its response Messages. Between utterances on a named session the
orchestrator holds no state for that session.

The orchestrator **MAY** maintain a transient per-utterance cache (the
inbound session, the active Match, etc.); such caches are
utterance-scoped and **MUST NOT** be relied upon as durable across
utterances.

A consequence: any orchestrator in a deployment can serve any inbound
Message on any named session — no coordination required, no shared
state to consult.

### 2.3 The orchestrator owns `session_id == "default"`

The orchestrator **MUST** maintain persistent in-process state for
`session_id == "default"` — the **default-session store**. This is the
one exception to §2.2. Behaviour rules for this store are in §5.

### 2.4 Project state into session when practical; plugin-internal state is permitted

A component that holds `session_id`-keyed state **across** utterances
**SHOULD** project that state into a session-resident field it claims
under SESSION-1 §2.2. Projection flows through `Match.updated_session`
(PIPELINE-1 §4.2) for pipeline plugins, or through in-place mutation
at transformer and handler boundaries (§2.6). Projected state is
**resumption-safe by construction** — it travels with the session,
survives orchestrator restart, and is transparent to multi-orchestrator
deployments.

A component **MAY** instead hold authoritative cross-utterance state
internally when projection is impractical. A component that takes this
path:

- **MUST** own its state lifecycle in full — persistence (or not),
  expiry, eviction, and any coordination the deployment requires;
- **MUST NOT** expect other components or clients to know its state
  exists or to compensate for its absence;
- offers **best-effort resumption with no normative guarantee** — the
  spec does not bind the outcome of any plugin-internal resumption
  attempt.

Transient in-utterance caches (helper structures built and discarded
within a single `match` call or transformer invocation) are always
permitted regardless of projection choice.

### 2.5 Clients own their named sessions

A client that uses a named `session_id` (anything other than
`"default"`) **MUST** be its own authoritative store for that session's
state. Persistence format and lifetime are entirely the client's
choice. The spec places no constraint on what `session_id` or `session`
value a client sends; trust boundaries are a layer-2 concern.

### 2.6 When session mutates in place

In-place session mutation during an utterance lifecycle is permitted
only at these boundaries:

- **transformer boundaries** — any of OVOS-TRANSFORM-1's six hooks
  (audio, utterance, metadata, intent, dialog, TTS);
- **pipeline boundaries** — a pipeline plugin's `match` returning a
  non-null `Match.updated_session` per PIPELINE-1 §4.2; the
  orchestrator **MUST** apply it as `session = match.updated_session`
  immediately;
- **handler boundaries** — a dispatched handler **MAY** mutate
  session in place; its emissions via `forward` / `reply` / `response`
  (OVOS-MSG-1 §5) carry the mutated session forward.

Bus events emitted outside these boundaries **MUST NOT** be relied upon
to mutate session state during the current utterance. A bus-emitted
Message carrying a mutated session **MAY** affect subsequent utterances
on that session (merged by the client per §3), but **MUST NOT** be
expected to affect the utterance during which it was emitted.

---

## 3. Client-side merge rules

### 3.1 `session_id` is the only key

A client **MAY** update its local session tracking from any Message it
observes carrying a `session_id` matching its own. No other matching
predicate is normative.

### 3.2 Every assistant-emitted Message carries an updated session

Per PIPELINE-1 §4.2 and §5, every assistant-emitted Message carries a
valid session snapshot at the point of emission. A client that adopts
any one such Message's session has a snapshot consistent with the
assistant's view at that point in the round. Adopting the latest
received session is the simplest conformant policy.

### 3.3 `ovos.utterance.handled` is the canonical convergence point

`ovos.utterance.handled` (PIPELINE-1 §9) is emitted exactly once per
utterance on every terminal path. A client that wants a single
"round is over" snapshot **SHOULD** adopt the session carried by this
end-marker. A client **MAY** also merge incrementally per §3.1; both
are conformant.

---

## 4. Resumption semantics

### 4.1 Resumption is implicit

A client **MAY** re-emit a previously-used `session_id` with its
locally-held session state at any time. There is no session-resume
handshake on the wire: the inbound Message's session IS the resume.
The orchestrator processes it per §2.2 — stateless, without knowledge
of prior utterances.

### 4.2 What is resumption-safe

Resumption-safe state is every field in the SESSION-1 §3 registry, plus
the projected state of any component that elected §2.4's
SHOULD-project pathway (by construction, since it lives in
session-resident fields).

Resumption is field-by-field: omitted fields resolve to deployment
defaults at the consumer (SESSION-1 §2.1). A client that resumes
without, for example, `intent_context` enters with a fresh context but
retains every other present field.

### 4.3 Plugin-internal state — best-effort resumption

State held internally by a component per §2.4's MAY-internal pathway
is governed by the holding component alone. The spec defines no
protocol for plugin-internal state resumption; the plugin chooses what
to persist and what "resume" means for its own state.

---

## 5. The default-session ownership rule

### 5.1 Persistent orchestrator-held state

The orchestrator **MUST** maintain a **default-session store** — persistent
in-process state keyed under `session_id == "default"`. The store:

- **merges inbound fields**: a present field on an arriving
  `session_id == "default"` Message replaces the stored value for that
  field; an omitted field leaves the stored value unchanged (SESSION-1
  §2.1 — the stored value is the last authoritative value);
- **sources outbound sessions**: every Message the orchestrator emits
  on the default session derives from the current store, so dispatched
  handlers see the most-recent default-session state;
- **reflects in-utterance mutations**: transformer, pipeline-match, and
  handler mutations (§2.6) propagate into the store through the
  standard derivation chain.

### 5.2 Restart semantics

The default-session store is process-local. An orchestrator restart
discards it; the default session reverts to deployment defaults.
Deployments that want persistence across restarts **MAY** implement
orchestrator-side persistence; the spec does not require it.

### 5.3 Component reliance on default-session continuity

Components **MAY** rely on default-session continuity within a single
deployment lifetime: a §2.4-projected field is preserved across
utterances because the orchestrator's store holds it. For named
sessions the same field is preserved only as long as the client holds
it locally — best-effort on remote peers.

---

## 6. Conformance

### 6.1 Bus

The bus **MUST** be stateless with respect to session. It **MUST NOT**
interpret, mutate, persist, or special-case `Message.context.session`.

### 6.2 Orchestrator

An orchestrator **MUST**:

- treat every named session (`session_id != "default"`) as stateless
  per §2.2 — no cross-utterance state held outside what the inbound
  Message brings;
- hold the default session as persistent in-process state per §5;
- apply in-place session mutations only at the boundaries of §2.6
  (transformer, pipeline-match, handler);
- propagate session forward unchanged on every Message derivation
  (OVOS-MSG-1 §5, SESSION-1 §4) except where §2.6 dictates mutation;
- emit `ovos.utterance.handled` carrying the final round session as the
  canonical convergence point (PIPELINE-1 §9, §3.3 above).

An orchestrator **MUST NOT** require any session-start, session-end, or
session-id-allocation handshake before processing an inbound Message.

### 6.3 Component

A component that holds `session_id`-keyed state across utterances
**SHOULD**:

- project that state into a session-resident field it claims under
  SESSION-1 §2.2, via `Match.updated_session` (pipeline plugins) or
  in-place mutation (transformers and handlers) per §2.6;
- read its state from `session` on each inbound Message rather than
  from a cross-utterance internal store.

A component **MAY** instead hold cross-utterance state internally per
§2.4 and **MUST** then take full responsibility for state lifecycle and
accept best-effort resumption (§4.3).

A component **MUST NOT** rely on bus events fired outside the utterance
lifecycle to mutate session state during the current utterance.

### 6.4 Client

A client using a named `session_id` **MUST**:

- hold its own authoritative session state for the `session_id` values
  it uses, per §2.5;
- include that state in `Message.context.session` on every inbound
  Message it emits;
- **NOT** expect the orchestrator to remember any session state between
  rounds — every round **MUST** be self-sufficient via the inbound
  session.

A client using `session_id == "default"` (the local device) **MAY**
omit `Message.context.session` or emit `session: {}` and rely on the
orchestrator's stored state — the only case this spec recognises where
a client need not carry its own authoritative state, because the
orchestrator's default-session store is that state.

A client **MAY** update its local session per §3, choose any
persistence format and lifetime, and re-emit a previously-used
`session_id` at any time (§4).
