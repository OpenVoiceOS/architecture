# Bus Message Specification

**Spec ID:** OVOS-MSG-1 · **Version:** 1 · **Status:** Draft

This document defines the **bus message** — the single unit of
communication exchanged between components of a voice-assistant
runtime. It covers:

- the on-the-wire **JSON envelope** (§2) — `type`, `data`, `context`;
- the **routing keys** inside `context` (§3) — `source` and
  `destination`, which mark the assistant-core / handler-code boundary;
- the **session carrier** inside `context` (§4) — the `session`
  object, whose wire shape is defined by OVOS-SESSION-1; this
  specification fixes only its existence and propagation rule;
- the **derivations** that produce a new Message from an existing one
  (§5) — `forward`, `reply`, `response`, and the explicit absence
  of any central correlation mechanism (messages are fully async);
- **serialization** rules (§6);
- **conformance** (§7).

It is implementation-agnostic: any process, in any language, on any
transport, can produce and consume conformant Messages. It is the
foundation other bus specifications build on.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and
**MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the JSON envelope of a Message (§2);
- the routing keys `source` and `destination` and their role marking
  the assistant-core / handler-code boundary (§3);
- the session carrier `session` and its propagation behaviour
  (§4); its wire shape, field set, and field semantics are owned
  by **OVOS-SESSION-1**;
- the three normative Message derivations `forward`, `reply`, and
  `response`, which propagate or rewrite the routing and session keys
  above (§5);
- the explicit non-prescription of any central correlation model —
  messages are fully asynchronous, askers correlate their own
  request/response chains if they need to (§5.4);
- serialization rules (§6);
- conformance (§7).

It does **not** define:

- *which* message topics exist — that is the domain of other
  specifications and of each component's own
  contract;
- the shape of a Message's `data` payload — fixed per-topic by the
  specification that defines the topic;
- the **internal shape** of `session` — fields, semantics,
  defaults, registry — owned by **OVOS-SESSION-1**;
- the **session lifecycle** — when a session begins, ends,
  expires, how it is resumed. `session` is a carrier; lifecycle is
  out of scope for this specification;
- any **central correlation mechanism** — no per-message identifier,
  no in-reply-to chain, no host-managed request/response bookkeeping.
  The bus is fully asynchronous; askers that need correlation handle
  it themselves using the raw material this spec provides (§5.4);
- any **state tracking** — components that need per-conversation
  state track it themselves keyed on the session identifier (per
  OVOS-SESSION-1), or arrange it out of band. Multi-turn
  conversation, intent context, cross-skill state, and similar
  concerns are deferred to other specifications;
- how identifiers are **assigned** or **resolved** — `source`,
  `destination`, and the `session` content are opaque to this
  specification;
- the **transport** carrying the message — websocket, in-process queue,
  Unix socket, or anything else are all permitted;
- encryption, authentication, authorization, delivery guarantees,
  ordering guarantees, and retry behaviour.

A Message is the unit of payload; a Message bus is the transport. This
specification is about the former only.

---

## 2. The envelope

A Message is a **JSON object** with exactly these top-level keys:

| Key | Type | Required | Meaning |
|------|------|----------|---------|
| `type` | string | yes | The message topic. |
| `data` | object | no | The message payload. An absent `data` is equivalent to an empty object (`{}`). |
| `context` | object | no | Assistant metadata about the message — routing keys (§3) and the session carrier (§4). An absent `context` is equivalent to an empty object (`{}`). |

Producers **MAY** omit `data` and/or `context` when they would be
empty; consumers **MUST** treat an absent `data` or `context` as
equivalent to `{}`. Producers **MUST NOT** emit any other top-level
key. A consumer that receives a Message with unknown top-level keys
**SHOULD** treat it as malformed but **MAY** instead ignore the
unknown keys and process the envelope normally — a hard reject
would let a single over-eager emitter (or a transitional serializer
adding a diagnostic key) sever otherwise-valid traffic across every
strict consumer on the bus. The strictness belongs on the producer
side.

### 2.1 `type`

`type` is a non-empty string identifying the message topic. It **MUST**
match the syntax:

- ASCII letters, digits, `.`, `:`, `_`, `-`;
- no whitespace;
- lowercase RECOMMENDED for new topics.

Dot- and colon-separated segments are common in topics —
`assistant.intent.register.keyword`, `XXX.response` — and have no
normative semantics here; segmenting is a convention used by the
specifications that define topics, not a feature of the envelope.

#### 2.1.1 Identifiers used as topic components

Some specifications define topics whose `type` string is assembled from
named identifiers at runtime — for example `<skill_id>:<intent_name>`
or `<component_type>.<component_id>`. For such a topic to be
unambiguously parseable, the identifiers it uses as components **MUST
NOT** contain the separator character(s) the topic uses structurally:

- a topic shaped `<A>:<B>` requires A and B to not contain `:`;
- a topic shaped `<A>.<B>` requires A and B to not contain `.`;
- a topic shaped `<A>.<B>:<C>` requires A and B to not contain `.`
  and B and C to not contain `:`.

Each specification declares only what its own separator requires.
No separator character is globally forbidden in all identifiers;
identifiers used in topics that do not use that character as a
structural separator may contain it freely.

**Recommended identifier form.** When defining a new identifier
intended for use as a topic component, prefer values that contain only
ASCII letters, digits, `_`, and `-`. This avoids accidental collision
with any separator a current or future topic shape may choose.

**Colon convention.** The `:` character is reserved for use by formal
specifications as a structural separator in topic shapes. Informally
defined or application-specific topics **SHOULD** avoid `:` in their
topic name so the convention remains unambiguous.

### 2.2 `data`

`data` is a JSON object. It **MAY** be empty (`{}`). Its keys, value
types, and required fields are fixed by the specification that defines
`type`. This specification places no further constraint on `data`'s
contents.

Producers **MUST NOT** rely on a particular serialization order of
`data` keys; consumers **MUST NOT** reject a Message because of key
order.

### 2.3 `context`

`context` is a JSON object carrying **assistant metadata** about the
Message — routing keys (§3), the session carrier (§4), and any other
metadata higher-level systems layer onto the envelope. It **MAY** be
empty (`{}`).

The distinction between `data` and `context` is intentional:

- `data` belongs to the topic — its shape changes from one topic to
  another.
- `context` is topic-independent — it travels with the Message
  regardless of topic and is available to consumers that do not know
  the topic.

Other specifications **MAY** define additional `context` keys for their
own purposes (GUI routing, security context, tracing identifiers, and
so on). A consumer **MUST NOT** reject a Message because of the
presence, absence, or value of any `context` key; a consumer that does
not understand a `context` key **MUST** ignore it.

---

## 3. Routing keys — the assistant-core / handler-code boundary

`source` and `destination` exist primarily to mark the boundary
between the **assistant core** and **third-party handler code** (or
any other external participant on the bus). Together they tell every
observer which direction a Message is travelling across that boundary
at any given moment.

### 3.1 The boundary, illustrated (informative)

A typical end-to-end flow, showing how the routing pair flips as the
Message crosses the boundary:

1. An **emitter** (a microphone service, a chat UI, a remote client,
   a test harness) sends an utterance Message and sets `source` to
   itself. The Message is going *user → assistant*; the assistant
   core is the consumer. **A -> B**
2. The assistant core classifies the utterance and matches an intent.
   It dispatches the handler on the per-intent topic
   `<skill_id>:<intent_name>` via `.reply`. The Message is now going
   *assistant → user*. **B -> A**
3. The handler runs and announces its outcome, preserving the
   dispatch's `context` via `.forward`. Observers still see the
   emitter as `destination`. **B -> A**

At each step the pair `(source, destination)` answers one question
unambiguously: *which side of the boundary is talking, and to
whom?*

### 3.2 `source`

`source` — string — opaque identifier of the **producer** of the
Message. The emitter sets it on origination; the `reply` derivation
(§5.2) rewrites it on each hop so it always names the current sender.
`forward` (§5.1) preserves it.

### 3.3 `destination`

`destination` — string OR array of strings — opaque identifier(s) of
the intended consumer(s). Absence (or an empty array) means
**broadcast** — every subscriber to the topic is an intended consumer.

The bus is not an authorization boundary: a consumer whose identifier
is not in `destination` may still observe the Message. `source` and
`destination` are **informational metadata** under this specification —
they tell observers who is talking and to whom, but enforcing
addressing (refusing to deliver, refusing to act, authenticating peers)
is the job of layer-2 systems built on top (§3.4), not of the Message
envelope.

### 3.4 Opacity and layer-2 extensions

`source` and `destination` are **opaque strings** from the perspective
of this specification. A consumer **MUST NOT** parse or ascribe
structure to their values beyond string equality. How identifiers are
minted (UUID, hostname-derived, etc.) is a deployment
concern.

Because the pair cleanly identifies *who is on the external side of
the boundary*, it is the natural attachment point for **layer-2
systems** that build authentication, authorization, multi-tenant
routing, or remote participation on top of the bus. A typical
layer-2 pattern populates `source` / `destination` with peer
identifiers so a satellite device or a remote client is addressable
on the same bus as a local handler, without the assistant core
itself learning about peers.

---

## 4. The session carrier

`session` — JSON object — carrier of the **conversational session**
the Message belongs to. A session ties together a sequence of
user-and-assistant exchanges that should be considered one unit of
conversation (one wake-word interaction, one chat thread, one
remote-client connection, one device interaction window — the
boundaries are deployment-defined).

The wire shape of `session`, its field set, and the semantics of every
field it carries are defined by **OVOS-SESSION-1**. This specification
owns only the fact that `session` rides inside `Message.context` and
the propagation rule that applies to it across the derivations of §5.

### 4.1 Propagation

Producers **SHOULD** set `session` on every Message that arises
within a session; consumers **SHOULD** propagate it onto Messages
derived from it (`forward`, `reply`, `response`) unchanged. Whatever
value the source Message carried for `session` — including an absent
or empty `session` — is preserved by propagation; a downstream
decision keyed on any session field continues to fire for every
derived Message in the same chain.

A producer **MUST NOT** modify a `session` already present on the
source Message during propagation. Propagation preserves the existing
session unchanged (§5.1).

### 4.2 The layer-2 picture

`session` combined with `source`/`destination` (§3) is what makes this
specification a **substrate for higher-level systems**: `source` and
`destination` mark the boundary on a per-Message basis; `session`
identifies which external participant the Message belongs to.

Together these mechanisms are sufficient to layer authentication,
authorization, and remote-participant routing above the bus without
modifying the bus envelope. A typical layer-2 system built this way
identifies peers via `source` / `destination` and per-peer sessions
via the `session` carrier.

---

## 5. Message derivations

Many topics participate in request/response chains, or relay Messages
across components. To make those chains **wire-portable** —
independent of any one implementation — this specification defines
three normative derivations that produce a new Message from an
existing one, propagating or rewriting the routing keys of §3 and the
session carrier of §4.

An implementation **MAY** offer the derivations under any names; what
matters is that the resulting Message has the shape described.

Given a source Message `M = { type: T, data: D, context: C }`:

### 5.1 `forward(T', D')`

Produces a new Message:

- `type` = `T'`,
- `data` = `D'`,
- `context` = `C` (preserved unchanged, including `source`,
  `destination`, and `session`).

Used to relay a Message under a new topic while preserving every
routing and session field. The forwarder does **not** become the new
`source` — the original producer remains named.

If the source Message has no `session`, the derivation **MAY**
materialize a default session on the result per OVOS-SESSION-1;
it **MUST NOT** modify a `session` already present.

### 5.2 `reply(T', D')`

Produces a new Message:

- `type` = `T'`,
- `data` = `D'`,
- `context` = a copy of `C` with the routing keys of §3 **reversed**
  so the new Message is addressed back to `M`'s producer:

  1. If `C.source` is set, the new context's `destination` is set to
     `C.source`.
  2. If `C.destination` is set:
     - and is a single string, the new context's `source` is set to
       `C.destination`;
     - and is an array of strings, the new context's `source` **MAY**
       be set to the identifier of the component producing the reply
       (typically one of the array entries). Selecting the **first
       element** is RECOMMENDED, so that independently written
       components converge on the same deterministic choice. The
       choice remains implementation-defined; consumers **MUST
       NOT** rely on a particular member being chosen.
  3. All other `context` keys, including `session` (§4), are
     preserved unchanged. As with `forward`, if the source Message
     has no `session`, the derivation **MAY** populate a default
     session on the result (§4.1).

`reply` is the basis of any "send back to the asker" Message. A
producer that does not maintain `source`/`destination` at all **MAY**
treat `reply` as equivalent to `forward` — the reply will be
broadcast, which is the only well-defined behaviour absent addressing
information.

### 5.3 `response(D')`

Equivalent to `reply(T + ".response", D')`. A `response` is a `reply`
whose topic is the source topic suffixed with `.response`. Topics
defined in other specifications **MAY** rely on the `.response`
suffix convention to mark a Message as the answer to a prior one.

### 5.4 No central correlation

Messages on the bus are **fully asynchronous**. This specification
defines **no** central correlation mechanism: no per-message
identifier, no in-reply-to chain, no host-managed
request/response bookkeeping.

What the spec *does* provide is the raw material an asker can use
to do its own correlation, if it wants to:

- the response is emitted on `<request_type>.response` (§5.3);
- `session` (§4) is preserved across `reply` / `response` /
  `forward` (§5.1–§5.2), so an asker can match an incoming
  `<request_type>.response` against an outstanding request in the
  same `session`.

Whether to do that, and how, is entirely the asker's
responsibility. Each component (skills, pipeline plugins, external
clients) tracks its own state as needed, keyed on the session
identifier (per OVOS-SESSION-1) when it cares about per-channel
continuity. Components that need richer discrimination than
topic + session — for example, multiple parallel requests on the
same topic in the same session — carry whatever they need in
`context` themselves or arrange it out of band.

A consequence: a Message on the bus is **self-contained**. Any
state a later consumer needs is either inside the Message (`data`
for topic-specific payload, `context` for cross-topic metadata,
`session` for per-channel carrier) or kept by some component out
of band — never recovered by a hidden host-side correlation
index. This keeps the bus async-friendly and is what makes the
layer-2 routing model viable.

---

## 6. Serialization

A Message is serialized as **UTF-8 JSON** per RFC 8259, with the
following constraints:

- No comments and no trailing commas (RFC 8259 already excludes both).
- Object key order is **not significant**. Producers and consumers
  **MUST NOT** rely on it.
- Numbers **MUST** be finite. `NaN`, `+Infinity`, `-Infinity` are
  forbidden.
- Strings are Unicode; producers **SHOULD** emit valid UTF-8 with no
  BOM.
- A serialized Message is a **single** top-level JSON object — not a
  JSON array, not a stream of objects. How multiple Messages are
  framed on a transport is a transport concern, not a Message
  concern.

A consumer that cannot parse a received payload as a JSON object
conforming to §2 **MUST** treat it as malformed and **MUST NOT**
silently coerce it.

---

## 7. Conformance

### A **producer** of Messages **MUST**:

- emit a top-level `type` matching §2.1, and no top-level keys
  beyond `type`, `data`, `context` (§2);
- give `type` a non-empty string value matching §2.1;
- when present, give `data` and `context` JSON-object values
  (possibly empty); they MAY be omitted when empty (§2);
- when deriving a Message from another (`forward` / `reply` /
  `response`), follow §5;
- emit serialization conformant to §6.

A producer **SHOULD**:

- set `source` to its own identifier when one is assigned (§3.2);
- set `destination` when the Message is targeted at a known consumer
  (§3.3);
- propagate `session` from a source Message to derived Messages
  unchanged (§4, §5.1–§5.2);
- when deriving a Message that answers another, use the `.response`
  suffix convention of §5.3 so observers can recognize the answer.

### A **consumer** of Messages **MUST**:

- reject a Message that violates §2 (wrong top-level keys, wrong
  types, missing or non-string `type`) as malformed;
- treat an absent `data` or `context` as equivalent to `{}` (§2);
- tolerate any `context` shape, including an empty object, and ignore
  `context` keys it does not understand (§2.3);
- treat the values of `source` and `destination` as opaque (§3.4);
  the contents of `session` are opaque to this specification —
  consumers consult OVOS-SESSION-1 for the field set and
  consumption semantics;
- not require any of `source`, `destination`, or `session` to be
  present — they are all optional, and a Message without them is
  well-formed.

A consumer **SHOULD**:

- propagate `session` (§4) onto Messages it derives from the received
  one;
- not rely on a particular order of `data` or `context` keys.

### Non-goals

The following are explicitly **outside** this specification and
**MUST NOT** be inferred from it: transport choice, encryption,
authentication, authorization, delivery guarantees, ordering
guarantees, retry behaviour, session lifecycle (start, end, expiry,
resumption), the internal shape of `session` (owned by
OVOS-SESSION-1), identifier assignment policy, and multi-tenant
routing semantics beyond the
opaque layer-2 substrate of §3.4 / §4.2.

---

## See also

- **OVOS-SESSION-1** — the wire shape of `session`, its field
  set, and consumption semantics.
