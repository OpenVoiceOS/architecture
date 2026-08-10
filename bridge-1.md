# Bus Bridge and Opaque Relay Specification

**Spec ID:** OVOS-BRIDGE-1 · **Version:** 2 · **Status:** Draft

This specification defines the **bus bridge** — a participant on the
internal message bus that terminates an external communication
channel and relays messages between the bus and remote participants.

The bridge itself carries very little normative weight. Most of what
this document describes are **emergent patterns** — behaviours that
arise naturally when composing the session-field, routing,
state-ownership, and lifecycle specifications across a bus boundary.
The bridge is simply the point in the architecture where these
compositions become visible.

It builds on **OVOS-MSG-1** (envelope and derivations),
**OVOS-SESSION-1** (session field registry and wire shape),
**OVOS-SESSION-2** (state ownership and the client-authority rule),
**OVOS-PIPELINE-1** (pipeline composition and per-session overrides),
**OVOS-TRANSFORM-1** (transformer chain per-session overrides), and
**OVOS-CONTEXT-1** (declarative intention context).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **bridge role** (§2) — the architectural position of a relay
  between internal and external message layers;
- the **normative core** (§3) — identity stamping, outbound routing,
  `site_id` assignment, and session preservation;
- **emergent patterns** (§4) — capabilities that arise from
  composing other specifications at the bridge boundary: access
  control, pipeline overrides, context pre-population,
  multi-deployment topologies, out-of-utterance session sync, and
  satellite skill registration;
- **ordering guidance** (§5) — the expectation of FIFO delivery;
- **conformance** (§6).

This specification does **not** define:

- **authentication or authorization** — how an external participant
  proves its identity or what it is permitted to do is a **layer-2**
  concern (see OVOS-MSG-1 §3.4); the bridge is assumed to have
  resolved these before injecting a message;
- **capability discovery** — how a participant signals its hardware
  features (microphone, screen, etc.) is a layer-2 concern;
- **wire protocol or encryption** — whether the external network
  uses WebSockets, MQTT, binary framing, or TLS is out of scope;
- **site policy** — the logic that decides to blacklist certain
  skills for a given participant is a layer-2 concern;
- **audio capture and local rendering** — microphone capture, STT,
  and local speaker output are deployment concerns; deployments
  that cannot render audio locally MAY receive synthesised audio
  over the bridge via `ovos.audio.speech` (OVOS-AUDIO-1 §4.3).

---

## 2. The bridge role

A **bridge** is a participant on the internal message bus that
terminates an external communication channel. It treats external
participants as opaque sources of messages.

To the internal bus, the bridge is the producer of the messages it
relays. To the external participant, the bridge is its connection to
the rest of the bus. The bridge is the **enforcement point** where
external protocol-specific data is translated into conformant
`OVOS-MSG-1` envelopes.

The bridge's normative obligations are enumerated in §6. Everything
else described in this spec — policy injection, pipeline overrides,
topology patterns — emerges from the way other specifications
interact at the bus boundary.

---

## 3. Normative core

### 3.1 Inbound identity stamping (`source`)

On receiving a message from an external participant, the bridge
**MUST** ensure the resulting bus Message carries a unique
identifier for that participant in `context.source`. Without a
unique `source`, the orchestrator and skills cannot route responses
back to the correct participant.

- If the external participant provides its own identifier, the
  bridge MAY use it verbatim or prefix it to guarantee uniqueness on
  the local bus.
- If the participant provides no identifier, the bridge MUST
  generate a unique value (e.g. a UUID).
- If the bridge manages multiple participants, it MUST ensure every
  active `source` string is distinct regardless of origin.

By stamping a unique `source`, the bridge makes the remote
participant appear as a first-class, addressable entity on the bus.

### 3.2 Outbound routing

The bridge relays messages from the bus to its external participants.
It MAY use any combination of `context.destination`, `context.session.session_id`,
and `context.session.site_id` to determine whether and where to relay
a message.

A participant is **known** to the bridge from the moment the bridge
assigns it a `source` identifier (§3.1) until the grace period
expires after the participant disconnects (§5). Messages matching a
known participant MUST be relayed; messages matching a participant
whose grace period has expired MUST be discarded.

**Primary signal: `destination`.** The bridge MUST relay every
Message whose `context.destination` matches one of its known
participants. This is the mechanism prescribed by **OVOS-MSG-1 §3**
— `destination` names the intended consumer, and the bridge is the
component that fulfills that naming by delivering the message across
the external channel.

The primary value of `destination`-based routing is **client
isolation**. The orchestrator uses `.reply()` to route responses,
setting `destination` to the original `source` stamped by the bridge
(§3.1). Two participants sharing the same `session_id` — including
`"default"` — are still distinguished: their `source` values differ,
so each receives only messages addressed to it.

**Secondary signal: `session_id`.** The bridge MAY also relay
messages by matching `context.session.session_id` to a known
participant, including messages whose `destination` is absent or
does not match. This is useful for catching session-scoped broadcasts
(e.g. `ovos.utterance.speak`, `ovos.utterance.handled`) that carry
the participant's session but no explicit destination. A bridge that
routes by `session_id` SHOULD also route by `destination`; the two
signals are complementary, not alternatives.

**Group signal: `site_id`.** If the bridge groups participants into
logical or physical clusters (a household, an office, a tenant), it
**MUST** use `context.session.site_id` (§3.3) to identify the group.
Group-level routing (e.g. "all satellites in this household receive
broadcast `ovos.stop`") is distinguished by `site_id`, not by
enumerating individual `session_id` or `destination` values.

**Identity mapping (NAT).** Just as a bridge MAY rewrite
`context.source` on outbound messages ("topology hiding", §3.1), it
MAY also rewrite `context.session.session_id` as messages cross the
boundary — translating a participant-local session identifier into a
hub-side identifier and vice versa. This enables a participant to
use its own `session_id` namespace without coordinating with the
hub. When performing `session_id` mapping, the bridge MUST maintain
a stable bijection between the participant's value and the hub-side
value for the lifetime of the participant's connection. When the
participant disconnects, the bridge SHOULD emit any cleanup events
(e.g. `ovos.skill.deregister` per §4.4) using the **hub-side**
`session_id` before dropping the bijection, so that orchestrator
state keyed on the hub-side value is cleaned up correctly.

In all routing modes:

- On matching a Message by any routing signal, the bridge MUST
  relay it to the corresponding external participant.
- The bridge SHOULD strip or translate internal routing metadata
  that is irrelevant to the external protocol before relaying.
  Internal metadata includes `context.pipeline_id`,
  `context.skill_id`, and internal `context.source` values that
  name components on the local bus; participant-facing fields
  such as `context.session` and `context.destination` SHOULD be
  preserved.
- A bridge MAY overwrite the `source` of outbound messages with a
  generic assistant ID ("topology hiding") when the identity of the
  emitting component is not meaningful to the external participant.
- A bridge MAY restrict topic subscription to a hardened minimum
  set for reduced attack surface. The minimum set for a
  multi-deployment topology (§4.2) is the utterance lifecycle
  defined in **OVOS-PIPELINE-1 §9** plus
  `ovos.session.sync` (**OVOS-SESSION-2 §2.7**), and any additional
  topics the participant's pipeline plugins depend on. The same matching
  signals (`destination`, `session_id`, `site_id`) apply within
  this restricted set.

### 3.3 `site_id` assignment

`site_id` is the **opaque group identifier** for a session, defined
in **OVOS-SESSION-1 §3.3** and owned by this specification. It names
the physical or logical cluster the participant belongs to; the
grouping criterion is chosen by the deployer.

A client or external participant **MAY** report a `site_id` in its
session. The bridge evaluates `site_id` assignment in this order:

1. If the bridge has its own determination (e.g. from resolving a
   Wi-Fi or Bluetooth scan, or reading a canonical area name from a
   connected home-automation system), it sets `site_id` to that
   value, overriding whatever the client supplied.
2. If the bridge has no determination of its own, it preserves the
   client's value unchanged.
3. If neither provides a `site_id`, the field is absent; consumers
   MUST treat an absent `site_id` as an unknown group and MUST NOT
   infer a default.

Once `site_id` is present on an inbound message after bridge
processing, downstream components **MUST NOT** overwrite it. The
value travels unchanged through every `forward` / `reply` /
`response` derivation per **OVOS-MSG-1 §5**.

Consumers **MUST NOT** parse or ascribe structure to `site_id`
beyond string equality. No specific value is reserved by this
specification.

### 3.4 Session preservation

The bridge **MUST** ensure every inbound bus Message carries a valid
`context.session` object (**OVOS-SESSION-1**) and **MUST** include
the session from every outbound bus Message in the external payload.

#### 3.4.1 Relaying mode (spec-aware participants)

When the external participant runs spec-compliant code and manages
its own session — as in the satellite-and-hub topology (§4.2.1) —
the bridge is a transparent carrier. It extracts the session from
the external payload and places it in the bus Message context
unchanged, and copies the session from outbound bus Messages back
into the external payload. The client is the authority for its own
session state per **OVOS-SESSION-2 §2**; the bridge MUST NOT inspect
or modify the session content.

A layer-2 system (§4.1) MAY apply policy mutations to the session
after extraction and before bus injection — this is an additional
layer operating on top of the transparent bridge, not a violation
of it. The bridge's transparency obligation is that it does not
alter the session on its own initiative; layer-2 policy is a
deliberate deployment-level decision separate from the bridge itself.

In this mode, the client is responsible for merging updated sessions
from `ovos.utterance.handled` (**PIPELINE-1 §9.5**) and for sending
the current session on every subsequent message.

#### 3.4.2 Managing mode (opaque participants)

When the external participant has no concept of OVOS sessions — as
in a chat-room gateway, an SMS bridge, or any non-OVOS client — the
bridge owns the session lifecycle for each participant. It assigns
a `session_id` per participant (or per logical channel such as a
room), synthesizes the session object on inbound messages, and
propagates session updates from outbound `ovos.utterance.handled`
events back into its own store for use on the next inbound message.

The bridge MUST assign a distinct `session_id` to each managed
participant. This is required to correlate `ovos.utterance.handled`
events — which carry the `session_id` of the utterance round — back
to the correct participant's session store.

If the external payload carries no session the bridge MUST
materialize one per **OVOS-SESSION-1 §4.1**. The bridge SHOULD apply
session updates carried by `ovos.utterance.handled` to the stored
session for that participant before injecting the next utterance. If
the next utterance arrives before `ovos.utterance.handled` is
received, the bridge MAY inject it using the last known session
state; the orchestrator will supply the updated session in the
following `ovos.utterance.handled`.

---

In both modes the session object on the bus MUST conform to
**OVOS-SESSION-1** and the orchestrator reads it as the
authoritative state for the round (**OVOS-SESSION-2 §2**). Any
field the bridge places in the session is automatically visible to
every downstream processing stage.

---

## 4. Emergent patterns

The behaviours described in this section are not new protocol. They
are what happens when you compose existing specifications at a bus
boundary. The bridge ensures every inbound Message carries a valid session
(§3.4); the session fields do the work.

### 4.1 Policy injection via session fields

A layer-2 system (see **OVOS-MSG-1 §3.4**) MAY mutate the session
object at the bridge boundary before the message is placed on the
internal bus. Because the orchestrator and plugins read session
fields to determine what to process, any field set or restricted at
the boundary is automatically enforced downstream — no
bridge-to-orchestrator protocol is needed.

The session fields available for injection are catalogued in
**OVOS-SESSION-1 §3**. Each is owned by the specification that
defines its semantics. The bridge need not understand any of them;
it carries them transparently. A layer-2 system SHOULD limit
mutations to fields registered in **OVOS-SESSION-1 §3**; overwriting
client-owned fields violates the client-authority rule of
**OVOS-SESSION-2 §2**.

**The gate invariant.** A boundary component that injects policy
fields — any `blacklisted_*` array, or a restricted `pipeline` —
**MUST** re-apply them to **every** inbound Message from the
governed participant. Applying policy once at connect time is not
conformant: the client is authoritative for its own session object
(**OVOS-SESSION-2 §2.5**) and may legitimately send a session that
omits the injected fields on any subsequent Message — at which
point a connect-time-only gate has silently granted the participant
everything the policy was meant to deny. The bridge is a gate, not
a handshake.

#### 4.1.1 Access control (denylist model)

A layer-2 system MAY populate any of the `blacklisted_*` arrays
catalogued in **OVOS-SESSION-1 §3** at the bridge boundary to
restrict what the participant can access. In this version the
registered denylist fields are three **OVOS-PIPELINE-1 §5** fields
(`blacklisted_skills`, `blacklisted_intents`,
`blacklisted_pipelines`) and six **OVOS-TRANSFORM-1 §5.2** fields
(`blacklisted_audio_transformers`,
`blacklisted_utterance_transformers`,
`blacklisted_metadata_transformers`,
`blacklisted_intent_transformers`,
`blacklisted_dialog_transformers`,
`blacklisted_tts_transformers`). All are arrays of string and absent
by default. The denylist is the only access-control mechanism —
there are no allowlist equivalents. Policy can only narrow, never
expand, a participant's capabilities.

#### 4.1.2 Pipeline and transformer preference

A participant MAY express a preference for its utterance lifecycle
by setting `session.pipeline` (**OVOS-PIPELINE-1 §5.1**) or any of
the six `session.*_transformers` lists (**OVOS-TRANSFORM-1 §5.1**).
A layer-2 system MAY override these by setting
`blacklisted_pipelines` or `blacklisted_*_transformers` at the
bridge boundary.

The split follows the composition model of **OVOS-PIPELINE-1 §5.5**:
preference is a wish, availability drops unloaded components, policy
removes disallowed entries. No later stage adds what an earlier
stage rejected.

#### 4.1.3 Context pre-population

`session.intent_context` (**OVOS-CONTEXT-1 §2**) MAY be seeded at the
bridge boundary so that a remote participant's first utterance
benefits from declarative gating or slot values. For example, a
layer-2 system that knows the participant's locale can pre-populate
a `lang` context entry, which intent matchers may gate on.

### 4.2 Multi-deployment topology

The bridge MAY connect two or more spec-compliant voice-OS
deployments as peers. In this topology every participant is a
spec-compliant deployment that speaks the native bus protocol
internally. The bridge is not a gateway to a foreign API; it relays
native bus messages between autonomous deployments.

The bridge carries no audio. Audio capture and rendering are local
concerns for each deployment; what crosses the bridge are bus
messages: utterances (post-STT, pre-intent), speak requests,
handler lifecycle events, and session mutations. A deployment MAY
run a separate audio transport channel alongside the bridge, but
that channel is outside the scope of this specification.

#### 4.2.1 Satellite and hub

One or more satellite deployments connect to a hub that owns intent
resolution and skill dispatch. The satellite sends utterances over
the bridge and receives speak responses back. See §4.2.5 for the
variant where the satellite has no TTS and the hub synthesises audio
on its behalf.

The lifecycle partition is conveyed by **message type**, not by
session fields. The bridge entry point is **OVOS-PIPELINE-1 §9.1**
`ovos.utterance.handle` — the post-STT, pre-intent message that
enters the hub's utterance lifecycle. The hub processes intent
matching, dispatch, and response generation. Two messages cross the
bridge back to the satellite: `ovos.utterance.speak` (**PIPELINE-1
§9.6**) carrying the text response, and `ovos.utterance.handled`
(**PIPELINE-1 §9.5**) carrying the terminal session state. The
satellite SHOULD wait for `ovos.utterance.handled` before accepting
the next utterance, and SHOULD merge the session it carries per
**OVOS-SESSION-2 §4**.

Where STT and audio output run depends on the deployment:

- **Local audio stack** — the satellite runs STT locally, injects
  `ovos.utterance.handle` with the transcribed text, receives
  `ovos.utterance.speak` as text, and runs TTS (and its
  dialog-transformer chain) locally before rendering audio. The
  bridge carries only bus messages.
- **Hub-side audio stack** — raw audio is transmitted to the hub
  (via a mechanism outside the scope of this specification). The
  hub runs STT and injects
  `ovos.utterance.handle` internally; it also runs the full
  audio-output layer including dialog-transformer and TTS,
  returning final audio to the satellite rather than text. In
  this model the satellite needs no local audio stack at all.

Session fields (`session.pipeline`, `session.*_transformers`,
`blacklisted_*`) are available to communicate per-session
preferences and policy between the two deployments (§4.1), but the
fundamental "we split here" is defined by which bus event the
satellite emits and which it receives back.

#### 4.2.2 Cascading intelligence

A local deployment runs its own full pipeline and handles utterances
it has skills for. When the local pipeline produces no match, the
local deployment forwards the utterance to a more capable remote
deployment over the bridge. Unlike the satellite-and-hub pattern
(§4.2.1) — where the satellite has no local intent pipeline at all —
the cascading deployment actively participates in the utterance
lifecycle before deciding to escalate.

The local deployment MAY communicate what it has already attempted
by seeding `session.intent_context` (**OVOS-CONTEXT-1 §2**) or
adjusting `session.active_handlers` (**OVOS-PIPELINE-1 §7.1**)
before forwarding the utterance. The bridge relays the message
unchanged; the remote deployment's pipeline receives the session as
context for its own matching. The remote deployment's speak and
handled events follow the routing path back through the bridge to
the originating participant.

#### 4.2.3 Session identity under multi-participant routing

Multiple participants connecting through the same bridge may use the
same `session_id` — most commonly `"default"`. The bridge
distinguishes them by `source` (§3.1), but session state is keyed on
`session_id` across the deployment (OVOS-SESSION-2 §3.1). Two
participants sharing `session_id: "default"` therefore share the
orchestrator's default-session store. Deployments that need session
isolation between participants SHOULD use distinct `session_id`
values or `session_id` NAT (§3.2) to prevent state collisions.
`site_id` identifies a routing group, not a session boundary, and
does not provide isolation on its own.

In a multi-satellite deployment, each satellite SHOULD use a
distinct `session_id`; a **managed-mode bridge** (§3.4.2) fronting
concurrent independent clients **MUST** assign each a distinct
`session_id` — in managed mode the bridge owns session attachment
for its opaque participants (§3.4.2), so distinctness is its
obligation, not theirs. This is necessary not only for session
isolation but for correct message routing: the orchestrator routes
responses (including `ovos.utterance.speak` and
`ovos.utterance.handled`) by deriving from the inbound message via
`.reply()`, which sets `context.destination` to the satellite's
`source`. A bridge routing by `session_id` alone cannot distinguish
two satellites sharing the same `session_id`.

#### 4.2.4 Per-satellite response personalisation

Because each satellite carries its own session, a single hub
utterance round may produce N distinct spoken responses — one per
satellite — each grounded on the same base orchestrator output but
shaped by that satellite's local transformer chain.

The hub emits the same base `ovos.utterance.speak` text to every
satellite. Personalisation happens **on the satellite**, not the hub:
the dialog-transformer chain (**OVOS-TRANSFORM-1 §3.5**) runs in the
satellite's local audio-output layer just before TTS. A satellite
with an LLM-backed dialog transformer (persona rewriting, tone
adjustment, translation) rewrites the hub's plain text before
synthesis.

The satellite's `session.dialog_transformers` field controls which
transformers its audio-output layer applies. Because the session is
carried in `ovos.utterance.speak` back to the satellite, the
satellite's audio-output service reads its own transformer
preferences from that session and applies them locally.

This pattern requires no bridge-level protocol and no coordination
between satellites. The bridge's only role is transparent session
relay (§3.4.1): each satellite's session — including its
`dialog_transformers` preference — travels to the hub on the
inbound utterance and returns on `ovos.utterance.speak`, where the
satellite's local audio-output layer reads it and applies the
configured chain.

> **Note — hub-side audio stack.** The pattern above assumes the
> bridge carries only bus messages and each satellite runs its own
> audio-output layer. An alternative deployment MAY transmit audio
> over the bus itself (for example, as base64-encoded payloads in
> bus Messages), in which case the full audio stack — including the
> dialog-transformer and TTS-transformer chains — runs on the hub
> rather than the satellite. The satellite in that model receives
> final audio bytes and has no local audio-output layer. This
> topology is outside the scope of BRIDGE-1 (§1 excludes audio
> input and output); its bus surface is defined by **OVOS-AUDIO-1**
> (see §4.2.5 for the TTS-as-a-service variant).

#### 4.2.5 TTS as a service

A satellite without a local TTS engine MAY request that the hub
synthesise speech on its behalf. The bridge translates the
`ovos.utterance.speak` Message it would normally relay back to the
satellite into `ovos.utterance.speak.b64` before placing it on the
hub bus. The hub's audio output service runs the full TTS pipeline
and emits `ovos.audio.speech` (OVOS-AUDIO-1 §4.3) with the
synthesised audio encoded as base64. The bridge relays
`ovos.audio.speech` to the satellite; the satellite decodes and
plays the audio directly.

In this topology audio crosses the bridge as base64 data rather than
as a local rendering obligation. The hub renders nothing locally for
sessions owned by the satellite.

### 4.3 Out-of-utterance session sync

The hub MAY emit `ovos.session.sync` (**OVOS-SESSION-2 §2.7**) to
push session-state changes to participants between utterance rounds
— for example, when configuration changes or a background process
mutates shared state. A bridge in a multi-deployment topology SHOULD
relay `ovos.session.sync` to the appropriate satellite(s) by
matching on `context.destination` or `context.session.session_id`,
using the same routing signals as for any other outbound message
(§3.2). `ovos.session.sync` is included in the hardened minimum
topic set (§3.2) for this reason.

### 4.4 Satellite skill registration

A satellite running its own **intent-based skills** may register
those skills on the hub's orchestrator by relaying the registration
messages through the bridge. The bridge forwards these messages
unchanged; the orchestrator keys them by the `session_id` in the
message context (**OVOS-INTENT-4 §11.1**), which is the satellite's
own `session_id`. No special registration protocol is needed.
Registration of satellite-side **pipeline plugins** on the hub is
not supported by any current specification and is out of scope here.

The effective intent pool, inheritance rule, and blacklist
interaction are defined in **OVOS-INTENT-4 §11.2**. When a
session-scoped intent is matched, the hub dispatches it as a
`.reply()` to the inbound utterance, setting `context.destination`
to the satellite's `source`; the bridge routes it back per §3.2.

**Disconnect.** When the satellite disconnects, the bridge SHOULD
emit `ovos.skill.deregister` (**OVOS-INTENT-4 §8.4**) for each skill
the satellite registered, carrying the satellite's session in the
Message **context**. The deregistration is scoped to the `session_id`
the orchestrator reads from `context.session.session_id`, never from
`Message.data` (**OVOS-INTENT-4 §8.4, §11.1**); the payload carries
`skill_id` only. When the bridge uses `session_id` NAT (§3.2), the
`session_id` it places in that context MUST be the hub-side value.

**Reconnect.** When the satellite reconnects, its session-scoped
registrations are gone. The satellite is responsible for
re-emitting its registration messages; the bridge relays them as on
initial connect. A bridge operating in managing mode (§3.4.2) MAY
cache the satellite's registration set and re-emit it on reconnect
on the satellite's behalf.

**Bridge emissions.** When the bridge itself emits bus messages
(cleanup events, health signals) rather than relaying participant
messages, it SHOULD use the `"default"` session unless the emission
is explicitly scoped to a specific participant's session.

---

## 5. Message ordering

A bridge SHOULD preserve the temporal ordering of messages for any
given `session_id` or `source`, where the underlying transport
provides ordered delivery. On transports that do not guarantee
ordering (e.g. UDP, MQTT QoS 0), out-of-order delivery is a
degradation of service quality but does not violate this
specification.

- Sequential utterances from the same participant SHOULD be placed
  on the bus in the order they were received.
- Responses from the bus targeting the same participant SHOULD be
  delivered in the order they were emitted.

A bridge MUST discard undeliverable Messages for a disconnected
participant after a deployment-defined grace period and MUST NOT
buffer them indefinitely.

---

## 6. Conformance

### A bridge **MUST**:

- stamp a unique identifier in `context.source` for every inbound
  message (§3.1);
- preserve the `session` object's content and structure during relay
  (§3.4);
- relay every matched message to the corresponding external
  participant, regardless of which routing signal produced the match
  (§3.2);
- conform to **OVOS-MSG-1** for all bus emissions;
- discard undeliverable messages after a deployment-defined grace
  period and MUST NOT buffer them indefinitely (§5).

### A bridge **SHOULD**:

- maintain FIFO ordering per `source` and per `session_id`, where
  the transport supports it (§5).

### A bridge **MAY**:

- use any routing signal combination described in §3.2
  (`destination`, `session_id`, `site_id`), including restricting
  to a hardened minimum topic set or performing `session_id` NAT;
- perform "topology hiding" by overwriting the `source` of outbound
  messages with a generic assistant ID (§3.2);
- mutate the `session` object to inject layer-2 policy or metadata
  before bus injection (§4.1);
- connect spec-compliant voice-OS deployments as peers (§4.2);
- relay `ovos.session.sync` to satellites between utterance rounds
  (§4.3);
- relay intent registration messages from satellite-side skills to
  the hub, and emit `ovos.skill.deregister` on satellite disconnect
  (§4.4);
- translate `ovos.utterance.speak` to `ovos.utterance.speak.b64`
  for satellites without local TTS, and relay `ovos.audio.speech`
  back to them (§4.2.5).

---

## See also

- **OVOS-MSG-1** — envelope and routing key definitions.
- **OVOS-SESSION-1** — session field registry and wire shape.
- **OVOS-SESSION-2** — client-side state authority and resumption.
- **OVOS-PIPELINE-1** — pipeline composition and per-session
  overrides.
- **OVOS-TRANSFORM-1** — transformer plugin chains and their
  per-session override fields.
- **OVOS-CONTEXT-1** — `session.intent_context` and declarative
  intent gating.
- **OVOS-AUDIO-1** — `ovos.utterance.speak.b64` and `ovos.audio.speech`
  for TTS-as-a-service (§4.2.5).
- **OVOS-INTENT-4** — session-keyed registration and
  `ovos.skill.deregister` (§4.4).
