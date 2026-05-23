# Bus Message Specification

**Spec ID:** OVOS-MSG-1 · **Version:** 1 · **Status:** Draft

This document defines the **bus message** — the single unit of
communication exchanged between components of an OVOS-style voice
assistant. It covers:

- the on-the-wire **JSON envelope** (§2) — `type`, `data`, `context`;
- the **routing keys** inside `context` (§3) — `source` and
  `destination`, which mark the OVOS / handler-code boundary;
- the **session carrier** inside `context` (§4) — the `session` object,
  including `session_id` (with the reserved `"default"` value) and
  `lang`, and the absent-session default rule;
- the **derivations** that produce a new Message from an existing one
  (§5) — `forward`, `reply`, `response`, and the explicit absence
  of any central correlation mechanism (messages are fully async);
- **serialization** rules (§6);
- **conformance** (§7).

It is implementation-agnostic: any process, in any language, on any
transport, can produce and consume conformant Messages. Every key and
derivation defined here already exists in current OVOS code paths; this
specification only formalizes them. No new fields are introduced. 
It is the foundation other bus specifications build on.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and
**MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the JSON envelope of a Message (§2);
- the routing keys `source` and `destination` and their role marking
  the OVOS / handler-code boundary (§3);
- the session carrier `session`, including the normative internal
  fields `session_id` and `lang`, the absent-session default, and
  propagation (§4);
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
- the **session lifecycle** or the **internal shape** of `session`
  beyond `session_id` and `lang` — when a session begins, ends,
  expires, what additional preferences or state it carries. `session`
  is a carrier; lifecycle and full structure are deferred to a future
  session specification;
- any **central correlation mechanism** — no per-message identifier,
  no in-reply-to chain, no host-managed request/response bookkeeping.
  The bus is fully asynchronous; askers that need correlation handle
  it themselves using the raw material this spec provides (§5.4);
- any **state tracking** — components that need per-conversation
  state track it themselves keyed on `session.session_id`, or
  arrange it out of band. Multi-turn conversation, intent context,
  cross-skill state, and similar concerns are deferred to future
  specifications;
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
| `data` | object | yes | The message payload. |
| `context` | object | yes | Assistant metadata about the message — routing keys (§3) and the session carrier (§4). |

Other top-level keys **MUST NOT** appear; consumers **MUST** reject any
Message with unknown top-level keys.

### 2.1 `type`

`type` is a non-empty string identifying the message topic. It **MUST**
match the syntax:

- ASCII letters, digits, `.`, `:`, `_`, `-`;
- no whitespace;
- lowercase RECOMMENDED for new topics.

Dot- and colon-separated segments are common in OVOS topics —
`ovos.intent.register.keyword`, `XXX.response` — and have no normative
semantics here; segmenting is a convention used by the specifications
that define topics, not a feature of the envelope.

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

## 3. Routing keys — the OVOS / handler-code boundary

`source` and `destination` exist primarily to mark the boundary between
**OVOS core** and **third-party intent-handler code** (or any other
external participant on the bus). Together they tell every observer
which direction a Message is travelling across that boundary at any
given moment.

### 3.1 The boundary, illustrated

A typical end-to-end flow, showing how the routing pair flips as the
Message crosses the boundary:

1. An **emitter** (a microphone service, a chat UI, a HiveMind client,
   a test harness) sends an utterance Message and sets `source` to
   itself. The Message is going *user → OVOS*; OVOS is the
   consumer. **A -> B**
2. OVOS classifies the utterance and matches an intent. It dispatches 
   the handler on the per-intent topic `<skill_id>:<intent_name>` via `.reply`.
   The Message is now going *OVOS → user*. **B -> A**
3. The skill's handler runs and announces its outcome, preserving the
   dispatch's `context` via `.forward`. Observers still see 
   the emitter as `destination`. **B -> A**

At each step the pair `(source, destination)` answers one question
unambiguously: *which side of the OVOS boundary is talking, and to
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
the OVOS boundary*, it is the natural attachment point for **layer-2
systems** that build authentication, authorization, multi-tenant
routing, or remote participation on top of OVOS. HiveMind is the
canonical example: it populates `source`/`destination` with peer
identifiers so a satellite device or a remote client is addressable on
the same bus as a local handler, without OVOS itself learning about
peers.

---

## 4. The session carrier

`session` — JSON object — carrier of the **conversational session**
the Message belongs to. A session ties together a sequence of
user-and-assistant exchanges that should be considered one unit of
conversation (one wake-word interaction, one chat thread, one HiveMind
client connection, one device interaction window — the boundaries are
deployment-defined).

The internal structure of `session` is mostly opaque under this
specification — its full shape (preferences, conversational state,
pipeline configuration, and so on) is to be formalized in a future
session specification. **Two** internal fields are given normative
meaning here, because OVOS subsystems already key behaviour off them:
`session_id` (§4.1) and `lang` (§4.2).

### 4.1 `session_id`

`session_id` — string — the identifier of the session. It uniquely
names the session within a deployment.

The value **`"default"`** is **reserved** and carries one specific
meaning: *the Message originates from the device itself*. It marks a
Message as locally-originated rather than as belonging to any remote
or named user session.

OVOS subsystems **MAY** use the device-local marker to make policy
decisions. The canonical example is **`ovos-audio`**, which uses
`session_id == "default"` to decide that synthesized TTS for this
Message should be played out of the device's own speakers; for any
other `session_id` (a remote chat session, a HiveMind satellite, a
test harness), it does not, leaving audio routing to the originating
session's owner.

### 4.2 `lang`

`lang` — string — the **user's preferred language**, as a BCP-47 tag
(OVOS-INTENT-2 §2). It declares which language the participant on the
external side of the OVOS boundary (§3) wants to communicate in.

The split between session and payload language is sharp:

- **Matching** is keyed on `data.lang` (the language the utterance was
  recognized in) — an `en-US` user can still trigger a `de-DE` intent
  if the utterance happens to be in German.
- **Output localization** — TTS voice selection, rendered dialog,
  prompt selection — is keyed on `session.lang` — the assistant
  speaks back to the user in the user's preferred language even when
  it has just handled a code-switched command.

A subsystem that produces speech, dialog, or text for the user
**SHOULD** read `session.lang` to decide which language to render in.

#### `data.lang` vs. `session.lang`

`lang` also appears in many topic-specific Message payloads under
`data.lang`, with a different meaning:

- `session.lang` is the **user's preferred language** — a property of
  the conversational session.
- `data.lang` describes the **language of the data carried in this
  Message** — the utterance just transcribed, the resource just
  registered, the dialog just rendered. It is a property of the
  payload.

The two **usually agree** — a user whose `session.lang` is `en-US`
typically speaks English utterances and triggers English-registered
intents — but they **MAY differ**. A code-switching user might issue a
Spanish command inside an English-preferred session: `data.lang` =
`es-ES`, `session.lang` = `en-US`. A consumer reading one should not
assume it equals the other.

Topics that carry a `data.lang` field are defined elsewhere; this
specification owns only the session-level `lang`.

### 4.3 Propagation and the absent-session default

Producers **SHOULD** set `session` on every Message that arises within
a session; consumers **SHOULD** propagate it onto Messages derived
from it (`forward`, `reply`, `response`) unchanged. The `session_id`
and `lang` values travel with the propagation, so a downstream decision
keyed on either continues to fire for every derived Message in the
same chain.

When `session` is **absent**, the Message is treated as if it carried
`session_id: "default"` — that is, *the Message originates from the
device itself* (§4.1). Consumers **MUST NOT** treat an absent `session`
as an unknown or untrusted origin; absence and `"default"` are
equivalent for every policy decision defined by this specification.

Because absence and `"default"` are equivalent, an implementation
**MAY** materialize the default during a `forward`, `reply`, or
`response` derivation (§5) — that is, populate a missing `session` on
the derived Message with a freshly-constructed default-session object
carrying `session_id: "default"` and any device-local fields the
implementation chooses. This is a convenience for downstream consumers
that prefer to operate on a present `session` object; it changes no
semantics.

A producer **MUST NOT** materialize a default in a way that overrides
a `session` already present on the source Message. Propagation
preserves the existing session unchanged (§5.1).

### 4.4 The layer-2 picture

`session` combined with `source`/`destination` (§3) is what makes OVOS
a **substrate for higher-level systems**:

- `source` / `destination` mark the OVOS-vs-external boundary on a
  per-Message basis;
- `session.session_id` marks *which* external participant the Message
  belongs to, with `"default"` reserved for "the device itself";
- `session.lang` carries the participant's language preference, so
  every output stage in the same session speaks back appropriately.

Together these mechanisms are sufficient to layer authentication,
authorization, and remote-participant routing above OVOS without
modifying OVOS core. HiveMind is the canonical layer-2 system built
this way: it identifies peers via `source`/`destination` and per-peer
sessions via `session.session_id`, and lets `session_id == "default"`
continue to mean "the device itself" — so a HiveMind satellite's TTS
does not accidentally play on the host's speakers.

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
populate a default session on the result (`session_id: "default"`,
§4.3); it **MUST NOT** modify a `session` already present.

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
       (typically one of the array entries). The exact choice is
       implementation-defined; consumers **MUST NOT** rely on a
       particular member being chosen.
  3. All other `context` keys, including `session` (§4), are
     preserved unchanged. As with `forward`, if the source Message
     has no `session`, the derivation **MAY** populate a default
     session on the result (§4.3).

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
clients) tracks its own state as needed, keyed on
`session.session_id` (§4.1) when it cares about per-channel
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

- emit exactly the top-level keys `type`, `data`, `context` (§2);
- give `type` a non-empty string value matching §2.1;
- give `data` and `context` JSON-object values (possibly empty);
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
  types, missing required keys) as malformed;
- tolerate any `context` shape, including an empty object, and ignore
  `context` keys it does not understand (§2.3);
- treat the values of `source`, `destination`, and the contents of
  `session` (other than the reserved `session_id == "default"`
  marker of §4.1) as opaque (§3.4);
- not require any of `source`, `destination`, or `session` to be
  present — they are all optional, and a Message without them is
  well-formed;
- treat an absent `session` as equivalent to `session_id: "default"`
  for every policy decision (§4.3) — including the `ovos-audio`
  TTS-locality decision and any analogous layer-2 routing.

A consumer **SHOULD**:

- propagate `session` (§4) onto Messages it derives from the received
  one;
- not rely on a particular order of `data` or `context` keys.

### Non-goals

The following are explicitly **outside** this specification and
**MUST NOT** be inferred from it: transport choice, encryption,
authentication, authorization, delivery guarantees, ordering
guarantees, retry behaviour, session lifecycle (start, end, expiry,
resumption), the internal shape of `session` beyond `session_id` and
`lang`, identifier assignment policy, and multi-tenant routing
semantics beyond the opaque layer-2 substrate of §3.4 / §4.4.

---

## See also

- `ovos-bus-client` — the reference Python implementation of this
  envelope and of the derivations of §5.
