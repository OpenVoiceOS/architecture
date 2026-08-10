# Bus Message Specification

**Spec ID:** OVOS-MSG-1 · **Version:** 2 · **Status:** Draft

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
  (§5) — `forward` and `reply`, the `response` shorthand, and the
  explicit absence
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
- the two normative Message derivations `forward` and `reply`, which
  propagate or rewrite the routing and session keys above, and the
  `response` shorthand defined in terms of `reply` (§5);
- the explicit non-prescription of any central correlation model —
  messages are fully asynchronous, askers correlate their own
  request/response chains if they need to (§5.4);
- serialization rules (§6);
- conformance (§7).

It does **not** define:

- *which* message topics exist — that is the domain of other
  specifications and of each component's own contract. A component
  that names topics under its own contract, without a specification
  defining them, is bound by the topic syntax of §2.1 and uses the
  dotted form only (§2.1.1);
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

A Message is a **JSON object** with no top-level keys other than
these:

| Key | Type | Required | Meaning |
|------|------|----------|---------|
| `type` | string | yes | The message topic. |
| `data` | object | no | The message payload. An absent `data` is equivalent to an empty object (`{}`). |
| `context` | object | no | Assistant metadata about the message — routing keys (§3) and the session carrier (§4). An absent `context` is equivalent to an empty object (`{}`). |

Producers **MAY** omit `data` and/or `context` when they would be
empty; consumers **MUST** treat an absent `data` or `context` as
equivalent to `{}`. Producers **MUST NOT** emit any other top-level
key.

A consumer that receives a Message carrying top-level keys it does
not know **MUST NOT** reject the Message on that ground alone, and
**MUST** ignore those keys. A Message is malformed only when it
cannot be parsed as a JSON object (§6) or when a key defined here
carries a value of the wrong type — a non-string or empty `type`, a
`data` or `context` that is not a JSON object. Malformed Messages
are handled per §6.1.

*Informative.* The asymmetry is deliberate: strictness belongs on
the producer side, where the defect originates; a consumer-side
reject on an unknown key would let a single non-conformant emitter
sever otherwise-valid traffic for every consumer on the bus, and
would make every additive envelope extension a breaking change.

### 2.1 `type`

`type` is a non-empty string identifying the message topic. It **MUST**
match the syntax:

- ASCII letters, digits, `.`, `:`, `_`, `-`;
- no whitespace;
- no empty segment: a topic **MUST NOT** begin or end with `.` or
  `:`, and **MUST NOT** contain two adjacent separators (`..`, `::`,
  `.:`, `:.`);
- lowercase **RECOMMENDED** for new topics.

Dots segment a topic into a readable hierarchy —
`assistant.intent.register.keyword`, `XXX.response`. The dot has no
normative semantics in the envelope; hierarchy depth and segment
meaning are conventions of the specifications that define topics.
The colon, by contrast, **is** normatively reserved — §2.1.1 defines
the rule, which every topic-defining specification inherits.

#### 2.1.1 The topic convention: colon vs. dot

Two topic shapes exist on the bus, distinguished by one character:

1. **Dispatch topics** contain a `:` and are assembled at runtime
   from identifiers — the canonical shape is
   `<skill_id>:<intent_name>`, the per-intent handler-dispatch
   topic. The `:` **is the marker** that a topic addresses a
   specific registered handler rather than naming an event. A
   dispatch topic **MUST** contain exactly one `:`, so that a
   consumer can split it into its two identifier roles without
   ambiguity. Only a specification in this family **MAY** define a
   colon-bearing topic shape, and it **MUST** define the identifier
   role on each side of the `:`. A component **MUST NOT** invent a
   colon-bearing topic under its own contract.
2. **All other topics** — events, requests, responses, lifecycle
   signals — use the dotted form `<x>.<y>.<verb>` (any depth) and
   **MUST NOT** contain `:`. This is the only form available to a
   component naming topics under its own contract (§1).

Consequently a consumer **MAY** classify any topic by a single test:
a `:` anywhere in `type` means a dispatch-shaped topic per the
specification that defined that shape; no `:` means an ordinary
dotted topic.

**Decomposed and assembled shapes.** A topic shape built from
identifiers is either:

- **decomposed** — a consumer is expected to recover the component
  identifiers by splitting the received topic string. The dispatch
  shape `<skill_id>:<intent_name>` is decomposed: the handler reads
  `skill_id` and `intent_name` back out of the topic; or
- **assembled** — the identifier is used only to *build* a topic
  string that is then matched by exact subscription. No consumer
  splits it, because the identifier is already known to every
  participant, carried in `data`, or both. The addressed poll shape
  `<skill_id>.converse.ping` (OVOS-CONVERSE-1 §4.2) is assembled:
  the owner subscribes to the one topic built from its own
  `skill_id`, and the same value is repeated in `data.skill_id`.

A topic-defining specification **MUST** state which of the two its
shape is. Absent such a statement the shape is decomposed.

**Separator hygiene.** In a **decomposed** shape, an identifier used
as a component **MUST NOT** contain the character(s) the shape uses
structurally:

- in `<A>:<B>`, neither A nor B contains `:`;
- in `<A>.<B>`, neither A nor B contains `.`;
- shapes combining both separators impose both constraints on the
  components they delimit.

An **assembled** shape imposes no such constraint: an identifier
containing the shape's separator (a `skill_id` such as `wiki.test`
in `wiki.test.converse.ping`) is conformant, because the resulting
topic is never split. The identifier must still yield a topic
matching §2.1.

Each topic-defining specification declares only what its own
separators require of its own identifiers; a character is
constrained only where it is structural in a decomposed shape.

**Recommended identifier form.** When defining a new identifier
intended for use as a topic component, prefer values that contain only
ASCII letters, digits, `_`, and `-`. This avoids accidental collision
with any separator a current or future topic shape may choose.

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
so on). A consumer that does not understand a `context` key **MUST**
ignore it, and **MUST NOT** reject a Message because of the presence,
absence, or value of a key it does not own.

A key's **owning specification** — the specification that defines the
key and its value semantics — **MAY** prescribe how a consumer of
that key handles a malformed value, up to and including dropping the
Message. That prescription binds only consumers acting on that key:
OVOS-SESSION-1 §2.5 defines such a rule for a `session` value that is
not a JSON object, where the carrier itself is unusable. Rejection is
therefore always the owning specification's deliberate choice about
its own key, never a consumer's reaction to metadata it does not
understand.

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
   core is the consumer.
2. The assistant core classifies the utterance and matches an intent.
   It dispatches the handler on the per-intent topic
   `<skill_id>:<intent_name>` using the `reply` derivation (§5.2):
   `destination` becomes the emitter, and `source` becomes the
   orchestrator's own identifier. The Message is now going
   *assistant → user*.
3. The handler runs and announces its outcome using the `forward`
   derivation (§5.1), preserving the dispatch's `context`. Observers
   still see the emitter as `destination` and the orchestrator as
   `source`.

At each step the pair `(source, destination)` answers one question
unambiguously: *which side of the boundary is talking, and to
whom?*

**A worked round-trip.** The rule of §5.2 — a reply names its own
producer as `source` — is what keeps an addressed poll from bouncing
back at the asker. Take a converse poll (OVOS-CONVERSE-1 §4.2) for a
remote skill reached through a bridge, with the utterance emitted by
a satellite the bridge has stamped `sat-7`:

| Step | Producer | Derivation | `source` | `destination` |
|------|----------|------------|----------|---------------|
| utterance | satellite, via bridge | origination | `sat-7` | absent (broadcast) |
| `<skill_id>.converse.ping` | converse plugin | `reply` of the utterance | `converse-plugin` | `sat-7` |
| `<skill_id>.converse.pong` | polled skill | `reply` of the ping | `<skill_id>` | `converse-plugin` |

The pong is addressed to the converse plugin, which asked, and not
to `sat-7`, which did not — because the ping's producer wrote its
own identifier into `source`, and the pong reverses that pair. Had
the ping instead copied the emitter's identifier forward, the pong
would have come back addressed to `sat-7`, and the bridge would have
relayed an internal poll response out to the satellite while the
converse plugin waited for an answer that never arrived.

### 3.2 `source`

`source` — string — opaque identifier of the **producer** of the
Message. The emitter sets it on origination; the `reply` derivation
(§5.2) rewrites it to the identifier of the component producing the
reply, so a replied Message always names its own producer.
`forward` (§5.1) preserves it, deliberately: a forwarded Message is
a relay of someone else's Message and keeps naming the original
producer.

### 3.3 `destination`

`destination` — string OR array of strings — opaque identifier(s) of
the intended consumer(s). Absence (or an empty array) means
**broadcast** — every subscriber to the topic is an intended consumer.

A producer **MUST NOT** emit an empty string as `destination`, or as
a member of a `destination` array; no identifier is ever the empty
string. A consumer that receives one **MUST** treat that value as
absent — an empty-string `destination` is a broadcast, and an
empty-string array member is ignored. The same holds for `source`.

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
structure to their values beyond string equality. Where `destination`
is an array, the test is per-member string equality: a consumer is an
intended consumer when its own identifier equals any member. How
identifiers are minted (UUID, hostname-derived, etc.) is a deployment
concern.

**Layer-2 systems.** A *layer-2 system* is a system that composes the
mechanisms of this specification without extending the envelope: it
reads and writes `source`, `destination`, and the fields of `session`
that it owns, and derives whatever behaviour it needs — authentication,
authorization, multi-tenant routing, remote participation — from those
values alone. It adds no top-level key, no derivation, and no
correlation mechanism. The **assistant core** — the components that
classify an utterance, select a handler, and dispatch it — is
unaware of any layer-2 system above it, and remains conformant to
this specification whether one is present or not.

Because the routing pair cleanly identifies *who is on the external
side of the boundary*, it is the natural attachment point for such
systems. A typical layer-2 pattern populates `source` /
`destination` with peer identifiers so a satellite device or a
remote client is addressable on the same bus as a local handler,
without the assistant core itself learning about peers.

**Identifiers are not credentials.** `source` and `destination` are
routing keys, not authentication: a Message asserts its `source`, and
nothing in this envelope proves the assertion. Accordingly:

- a producer **MUST NOT** set `source` to an identifier that was not
  assigned to it, and **MUST** omit `source` when it has no assigned
  identifier;
- a component that admits Messages into the bus from outside its
  trust domain — a bridge, a gateway, any transport terminator —
  **MUST** overwrite `source` on every inbound Message with the
  identifier it has assigned to that peer, discarding whatever the
  peer supplied. It **MUST NOT** trust a peer-supplied `source`, on
  the first Message or on any later one;
- a consumer **MUST NOT** treat a matching `source` as proof of
  identity, and **MUST NOT** derive an authorization decision from
  `source` or `destination` alone. Authorization, where a deployment
  needs it, is a layer-2 concern built on evidence this envelope does
  not carry.

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
within a session. A component that derives a Message from another
(`forward`, `reply`, `response`) **MUST** propagate `session` onto
the derived Message. Whatever value the source Message carried for
`session` — including an empty `session`, and including an absence
that no derivation chose to materialize (§5.1) — is carried through;
a downstream decision keyed on any session field continues to fire
for every derived Message in the same chain.

Propagation is **MUST**, not **SHOULD**, because the specifications
built on this one treat an unpropagated session as a defect rather
than a variation: a derived Message that drops the session detaches
its chain from the conversation it belongs to, and every
session-keyed decision downstream of it silently changes answer.

**Session mutation.** A producer **MUST NOT** modify a `session`
already present on the source Message during propagation, with one
exception: a component acting at one of the mutation boundaries
enumerated in **OVOS-SESSION-2 §2.6** — transformer, pipeline, and
handler boundaries — **MAY** mutate the session fields **it owns**,
and the derived Message then carries the mutated session forward.
Fields the component does not own **MUST** be carried through
unchanged, whether or not the component understands them. Outside
those boundaries, propagation preserves the existing session
unchanged (§5.1).

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
two normative derivations that produce a new Message from an existing
one, propagating or rewriting the routing keys of §3 and the session
carrier of §4, plus the `response` shorthand defined in terms of
`reply`.

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
materialize a default session on the result per OVOS-SESSION-1. A
`session` already present is carried unchanged, subject to the
owned-field exception of §4.1.

### 5.2 `reply(T', D')`

Produces a new Message:

- `type` = `T'`,
- `data` = `D'`,
- `context` = a copy of `C` with the routing keys of §3 rewritten so
  the new Message names its own producer and is addressed back to
  `M`'s producer:

  1. **`destination`.** If `C.source` is set, the new context's
     `destination` is set to `C.source`. If `C.source` is absent,
     the new context's `destination` is omitted — the reply is a
     broadcast, which is the only well-defined behaviour when there
     is no asker to name.
  2. **`source`.** The new context's `source` is set to the
     identifier of the **component producing the reply** — its own
     assigned identifier, not any value read out of `C`. This holds
     whatever shape `C.destination` had: a string, an array, or
     absent. A producer with no assigned identifier **MUST** omit
     `source`, and **MUST NOT** copy `C.destination` or `C.source`
     into it.
  3. All other `context` keys, including `session` (§4), are
     preserved unchanged. As with `forward`, if the source Message
     has no `session`, the derivation **MAY** populate a default
     session on the result (§4.1).

`reply` is the basis of any "send back to the asker" Message. Its
`source` rule is what makes an addressed round-trip terminate at the
component that opened it: because each hop names itself, the answer
to a reply is addressed to the component that asked, not to whoever
asked *that* component (§3.1). Every request/response round-trip in
the specifications built on this one — the converse and fallback
polls, the common-query contest, the handler dispatch — relies on
that property. A producer that maintains no identifier at all still
conforms: its replies carry no `source`, and the component it
answered addresses its own next Message by broadcast.

*Informative.* A single rule replaces the older reversal: `source`
comes from the producer, never from `C.destination`. Copying
`C.destination` gave the same answer only in the case where the
replying component was the sole addressee, and gave a wrong or
undefined answer everywhere else — on a broadcast the producer had
no value to copy, and on a multi-addressee Message it had several.

### 5.3 `response(D')`

Equivalent to `reply(T + ".response", D')`. A `response` is a `reply`
whose topic is the source topic suffixed with `.response`. Topics
defined in other specifications **MAY** rely on the `.response`
suffix convention to mark a Message as the answer to a prior one.

The shorthand is defined only where the arithmetic is unambiguous:
`T` **MUST NOT** already end in `.response`, and **MUST NOT** contain
a `:`. A dispatch topic (§2.1.1) has no `.response` counterpart, and
suffixing an answer topic again produces
`<x>.response.response`, which no specification defines. Where either
condition fails, the answering component names the answering topic
explicitly and derives via `reply` instead.

An asking component **MUST NOT** assume that the answer to a Message
on topic `T` arrives on `T + ".response"` unless the specification
that defines `T` says so. Several specifications name their answering
topic directly — `<skill_id>.converse.pong` answers
`<skill_id>.converse.ping` (OVOS-CONVERSE-1 §4.2) — and the answering
topic is always whatever the defining specification states.

### 5.4 No central correlation

Messages on the bus are **fully asynchronous**. This specification
defines **no** central correlation mechanism: no per-message
identifier, no in-reply-to chain, no host-managed
request/response bookkeeping.

What the spec *does* provide is the raw material an asker can use
to do its own correlation, if it wants to:

- the answering topic, which the specification defining the request
  topic states (§5.3);
- `session` (§4), which is propagated across `reply` / `response` /
  `forward` (§5.1–§5.2), so an asker can narrow an incoming answer
  to the conversation it belongs to;
- `data` itself: an answering component **SHOULD** echo back a
  discriminating field from the request it answers, so the asker can
  pair answer to request without any host bookkeeping. The
  common-query contest does exactly this — a skill's answer echoes
  the opaque `query_id` it was asked about (OVOS-COMMON-QUERY-1
  §6.4) — and it is the pattern to follow for any topic where
  several requests may be outstanding at once.

Topic and session alone do **not** discriminate parallel requests:
two requests on one topic in one session produce two indistinguishable
answers. An echoed field is what separates them, and a specification
that expects parallel requests **SHOULD** name the field to echo.
Whether to correlate at all, and how, is otherwise entirely the
asker's responsibility. Each component (skills, pipeline plugins, external
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

### 6.1 Handling a malformed Message

"Treat as malformed" has one meaning throughout this specification
and every specification that cites it. A consumer that treats a
Message as malformed:

- **MUST NOT** act on it — no handler runs, no state changes, no
  derived Message is emitted from it;
- **MUST** drop it, and **MUST NOT** repair or coerce it into a
  conformant shape by guessing at the producer's intent;
- **MUST NOT** crash, and **MUST NOT** let the fault tear down its
  transport or its subscriptions. A malformed Message is a
  **per-message producer fault**, never a transport fault; a consumer
  that drops its bus connection over one bad Message can be held
  offline indefinitely by a single misbehaving producer;
- **SHOULD** log the violation, with enough detail to identify the
  producer, so that the defect is fixable;
- **MUST NOT** emit an error Message in reply, unless the
  specification defining the topic prescribes one. A malformed
  Message often carries no usable routing keys, and an unprompted
  error reply to a broadcast is itself a source of bus noise.

Dropping is silent to the bus and loud to the operator. That is the
intended asymmetry: the defect is reported where it can be fixed,
without a second Message that other consumers must now interpret.

---

## 7. Conformance

### A **producer** of Messages **MUST**:

- give `type` a non-empty string value matching §2.1, and emit no
  top-level keys beyond `type`, `data`, `context` (§2);
- when present, give `data` and `context` JSON-object values
  (possibly empty); they **MAY** be omitted when empty (§2);
- when deriving a Message from another (`forward` / `reply` /
  `response`), follow §5 — in particular, set the `source` of a
  `reply` to its own identifier, never to a value read out of the
  source Message's `context` (§5.2);
- propagate `session` from a source Message onto every Message
  derived from it (§4.1, §5.1–§5.2), mutating only session fields it
  owns and only at the boundaries of OVOS-SESSION-2 §2.6;
- omit `source` when it has no assigned identifier, and never claim
  an identifier assigned to another component (§3.4);
- overwrite `source` on every Message it admits from outside its
  trust domain, when it is a bridge, gateway, or other transport
  terminator (§3.4);
- emit serialization conformant to §6.

A producer **SHOULD**:

- set `source` to its own identifier when one is assigned (§3.2);
- set `destination` when the Message is targeted at a known consumer
  (§3.3);
- when deriving a Message that answers another, use the `.response`
  suffix convention of §5.3 where it applies, so observers can
  recognize the answer;
- echo a discriminating field from the request in any answer it
  produces on a topic that may carry parallel requests (§5.4).

### A **consumer** of Messages **MUST**:

- treat a Message that violates §2 as malformed — an unparseable
  payload (§6), a missing, non-string, or empty `type`, a `type` that
  does not match §2.1, or a `data` or `context` that is not a JSON
  object — and handle it per §6.1;
- ignore unknown top-level keys, and **MUST NOT** reject a Message on
  that ground alone (§2);
- treat an absent `data` or `context` as equivalent to `{}` (§2);
- tolerate any `context` shape, including an empty object, and ignore
  `context` keys it does not understand, without rejecting the
  Message over a key it does not own (§2.3);
- treat the values of `source` and `destination` as opaque, comparing
  them by string equality only, per member where `destination` is an
  array (§3.4); the contents of `session` are opaque to this
  specification — consumers consult OVOS-SESSION-1 for the field set
  and consumption semantics;
- not treat `source` as proof of identity, and not derive an
  authorization decision from the routing keys alone (§3.4);
- not require any of `source`, `destination`, or `session` to be
  present — they are all optional, and a Message without them is
  well-formed.

A consumer **SHOULD**:

- not rely on a particular order of `data` or `context` keys.

A consumer that owns a `context` key **MAY** prescribe, in the
specification defining that key, how a malformed value of that key is
handled — including dropping the Message (§2.3, OVOS-SESSION-1 §2.5).
A consumer that derives Messages from the ones it receives is a
producer of those derived Messages and is bound by the producer rules
above, including session propagation (§4.1).

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
  set, and consumption semantics; the malformed-carrier rule (§2.5)
  that this specification's §2.3 carve-out permits.
- **OVOS-SESSION-2** — the session mutation boundaries (§2.6) that
  bound the owned-field exception of §4.1.
- **OVOS-PIPELINE-1** — the dispatch topic shape
  `<skill_id>:<intent_name>` (§7.1), the decomposed shape of §2.1.1.
- **OVOS-BRIDGE-1** — a layer-2 system built on the routing keys of
  §3 and the session carrier of §4.
