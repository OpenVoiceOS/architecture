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

*Informative.* Strictness belongs on the producer side. A
consumer-side reject on an unknown key would let one bad emitter
sever valid traffic for every consumer, and would make every
additive envelope extension a breaking change.

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
   `<skill_id>:<intent_name>` using the `reply` derivation (§5.2),
   which swaps the pair. The Message is now going *assistant → user*
   and is addressed to the emitter.
3. The handler runs and announces its outcome using the `forward`
   derivation (§5.1), preserving the dispatch's `context`.

The pair alternates rather than being rewritten, so the emitter's
identifier stays in it for the whole exchange.

**A worked round-trip.** The swap of §5.2 keeps the **peer**
named in the routing pair for the whole chain. Take a converse poll
(OVOS-CONVERSE-1 §4.2) where the utterance came from a satellite the
bridge stamped `sat-7`:

| Step | Derivation | `source` | `destination` |
|------|------------|----------|---------------|
| utterance | origination | `sat-7` | absent |
| `<skill_id>.converse.ping` | `reply` of the utterance | `sat-7` (unchanged — no destination to swap in) | `sat-7` |
| `<skill_id>.converse.pong` | `reply` of the ping | `sat-7` | `sat-7` |
| `ovos.utterance.speak` | `reply` of the dispatch | `sat-7` | `sat-7` |

`sat-7` rides the whole chain, which is the point: the final
user-facing Message is addressed to the satellite, and the bridge
routes it home on that value alone. A derivation that replaced
`source` with each producer's own identifier would drop `sat-7`
after one hop and the reply would never reach the user.

The pong is addressed to `sat-7` rather than to the converse plugin,
and the plugin still receives it — delivery is by **topic
subscription**, and `destination` is informational (§3.4). Stopping
an internal poll from being relayed out to the satellite is a bridge
concern — the bridge does not relay internal-coordination topics —
not something the routing keys arbitrate.

### 3.2 `source`

`source` — string — an opaque identifier set by the emitter on
origination. `forward` (§5.1) preserves it. `reply` (§5.2) swaps it
with `destination`, so on a replied Message `source` names the
**previous hop's addressee**, not the component that produced the
Message.

A consumer therefore **MUST NOT** read `source` as the identity of
the producer. Producer identity is not carried by this envelope. What
the pair does carry, hop after hop, is the **peer** on the external
side of the boundary (§3.1): the swap keeps that identifier alive
along the whole chain, which is what lets a layer-2 system route a
terminal Message back to the participant that started the
exchange.

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

### 4.2 The utterance identifier

`utterance_id` — opaque string — rides inside `Message.context`
beside `session` and names one **interaction lifecycle**: one
utterance entering the system, one out-of-band query, one
UI-originated command. Everything derived from that lifecycle — the
transformer passes, the pipeline contest and its polls, the pongs,
the dispatch, the terminal events — carries the same value, because
propagation across the derivations of §5 is the same **MUST** that
governs `session` (§4.1).

The component that **originates** a lifecycle stamps `utterance_id`,
exactly once: the client or listener emitting the utterance-entry
Message, the requester opening an out-of-band contest, the UI
emitting a command. The orchestrator **MUST** stamp it at lifecycle
entry when the source did not. A component **MUST NOT** overwrite a
`utterance_id` already present — downstream regeneration would
detach every already-derived Message from its lifecycle.

The value is opaque: consumers compare it for equality and do
nothing else. It **MUST** be unique per lifecycle within the
deployment (a UUID is RECOMMENDED; no format is normative). Two
Messages carry the same `utterance_id` **iff** they belong to the
same lifecycle — which is the entire correlation rule: a poll
answer whose `utterance_id` differs from the poll's answers some
other question. Specifications built on this one (OVOS-FALLBACK-1,
OVOS-COMMON-QUERY-1, OVOS-TRANSFORM-1 attribution) correlate by
this field and define nothing of their own.

### 4.3 The layer-2 picture

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
- `context` = a copy of `C` with `source` and `destination`
  **swapped**:

  1. If `C.destination` is set, the new `source` is `C.destination`
     — its **first element** when `C.destination` is an array.
  2. If `C.source` is set, the new `destination` is `C.source`.
  3. A key absent from `C` stays absent. On a broadcast (no
     `destination`) `source` is therefore carried through unchanged
     and only `destination` is written.
  4. All other `context` keys, including `session` (§4), are
     preserved unchanged. As with `forward`, if the source Message
     has no `session`, the derivation **MAY** populate a default
     session on the result (§4.1).

`reply` is the basis of any "send back to the asker" Message, and it
is a **transform on the Message alone** — it reads nothing but `C`.
That is why the new `source` comes from `C.destination` and not from
the replying component: the derivation has no access to the
component's identity.

The swap is what makes the routing pair survive a chain. Each hop
alternates the two values rather than replacing them, so the peer
identifier the exchange started with is still in the pair at the
last hop (§3.1). A component that maintains no addressing at all
still conforms: with neither key set, `reply` changes nothing.

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
- `utterance_id` (§4.2), stamped once at the lifecycle source and
  propagated the same way, so an asker can narrow an answer to the
  exact interaction that asked. This is still not central
  correlation: no host assigns it, no host tracks it, and no
  Message-level in-reply-to chain exists — equality comparison by
  whoever cares is all there is.

Whether to correlate at all, and how, is entirely the
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

"Treat as malformed" means the same thing everywhere this
specification and its companions use it. A consumer that treats a
Message as malformed:

- **MUST** drop it — no handler runs, no state changes, nothing is
  derived from it, and it is not repaired by guessing;
- **MUST NOT** let the fault tear down its transport or its
  subscriptions. One bad Message is a producer fault, not a
  transport fault;
- **SHOULD** log it with enough detail to identify the producer.

A consumer emits no error Message in reply unless the specification
defining the topic prescribes one; a malformed Message often carries
no usable routing keys to answer on.

---

## 7. Conformance

### A **producer** of Messages **MUST**:

- give `type` a non-empty string value matching §2.1, and emit no
  top-level keys beyond `type`, `data`, `context` (§2);
- when present, give `data` and `context` JSON-object values
  (possibly empty); they **MAY** be omitted when empty (§2);
- when deriving a Message from another (`forward` / `reply` /
  `response`), follow §5 — in particular, swap `source` and
  `destination` on a `reply`, taking the first element when
  `destination` is an array (§5.2);
- propagate `session` from a source Message onto every Message
  derived from it (§4.1, §5.1–§5.2), mutating only session fields it
  owns and only at the boundaries of OVOS-SESSION-2 §2.6;
- emit serialization conformant to §6.

A producer **SHOULD**:

- set `source` to its own identifier **on origination**, when one is
  assigned (§3.2);
- set `destination` when the Message is targeted at a known consumer
  (§3.3);
- when deriving a Message that answers another, use the `.response`
  suffix convention of §5.3 where it applies, so observers can
  recognize the answer.

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
opaque layer-2 substrate of §3.4 / §4.3.

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
