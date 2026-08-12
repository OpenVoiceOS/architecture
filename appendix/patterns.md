---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 3. Architectural patterns

Two patterns recur across the spec family and are worth a
dedicated treatment.

### 3.1 The bus as a substrate

Under OVOS-MSG-1's `source` / `destination` / `session` model,
the bus is not just an internal transport — it is the
**substrate higher-level systems plug into without modifying
the assistant core**. Two mechanics make that work:
**single-flip routing** (§3.1.1), which keeps the routing pair
correct end-to-end without per-component effort; **forward vs
reply discipline** (§3.1.2), which ensures non-dispatch
emissions are routed correctly through layer-2 transports; and
**no central state or correlation** (§3.1.3), which makes
layer-2 systems composable. HiveMind is the canonical example
of what all three together enable (§3.1.4).

#### 3.1.1 The single-flip routing model

The most important bus invariant in OVOS, and the one most
often reinvented incorrectly. The routing pair (`source`,
`destination`) flips **exactly once per conversational turn**,
performed by ovos-core, before the intent dispatch is emitted.
From that point on, every handler-side emission is *already*
addressed back at the user.

Three steps:

1. **The user side emits.** An external component —
   microphone service, chat UI, satellite client, test harness
   — emits an utterance with `source` set to itself:

       context: { source: "audio", destination: null, session: {...} }

2. **ovos-core flips, then dispatches.** When the intent
   service matches an intent it derives the dispatch via
   `Message.reply(match_type, data)`
   (`ovos-core/.../service.py:340`). The `.reply` rule of
   MSG-1 §5.2 swaps the routing pair:

       context: { source: "ovos-core", destination: "audio", session: {...} }

   The dispatch goes out on the per-intent topic
   `<skill_id>:<intent_name>`. The flip has already classified
   the message as *going back at the user*, even though a
   skill handler is what actually runs.

3. **The handler `.forward`s.** Every message the skill emits
   in response — `speak`, the handler lifecycle trio, GUI
   events — uses `Message.forward(...)`
   (`ovos-workshop/.../ovos.py:1461, 1472, …`). `.forward`
   preserves `context` unchanged, so every handler emission is
   already addressed back at the original user-side component.

Two consequences fall out:

- **The boundary is user ↔ assistant, not core ↔ handler.**
  Skill handlers are on OVOS's side of the boundary; from
  outside, OVOS is one thing. The user doesn't know or care
  which skill answered them.
- **Handler authors never write addressing code.** Because
  `.forward` preserves the flipped pair, no skill anywhere
  needs to understand `source` / `destination`. Get the
  inversion right once in ovos-core, and every downstream
  skill is automatically correct.

What this rules out: no per-hop addressing (handlers don't
pick their own `destination`); no second flip (handlers
`.forward`, they don't `.reply` to the dispatch); the dispatch
topic `<skill_id>:<intent_name>` selects the handler, not
`destination` (the destination belongs to the user).
Implementers using `.reply` where `.forward` is appropriate
produce mis-routed messages that work in local tests but
silently break layer-2 routing.

#### 3.1.2 Choosing `forward` vs `reply` for non-dispatch emissions

The single-flip model above applies to the dispatch path, but the
same choice arises whenever any component emits a Message derived
from one it received. The rule is:

- **Use `forward`** when the new Message travels in the **same
  direction** as the source — toward the same destination. A handler
  emitting `speak`, a session sync, or a lifecycle event during a
  dispatch uses `forward`; the user-side client receives it because
  the `destination` was already set by the earlier `reply` flip.
- **Use `reply`** when the new Message travels **back toward the
  sender** of the source — a responder answering a requester. A
  skill responding to `ovos.stop.ping`, a candidate answering an
  `ovos.converse.ping` poll, or a plugin answering an
  introspection request all use `reply`; the flip routes the answer
  back to whoever asked.

The practical consequence for layer-2 systems (satellites, gateways,
HiveMind nodes): routing metadata is the only mechanism a transport
has to decide where a Message goes. A component that uses `reply`
where `forward` is correct sends the Message back at itself instead
of toward the user; a component that uses `forward` where `reply` is
correct broadcasts with no destination instead of targeting the
requester. Both mistakes work on a single-node local bus (where
everything is broadcast anyway) and silently fail in layer-2
deployments.

The specs enforce this consistently:

| Emission | Derivation | Reason |
|---|---|---|
| Handler lifecycle trio (PIPELINE-1 §8) | `forward` | Travels toward the client alongside the dispatch |
| `ovos.session.sync` from a handler (SESSION-2 §2.7) | `forward` | Session update travels toward the client |
| `ovos.stop.pong` (STOP-1 §4.2) | `reply` | Response back to the stop plugin that pinged |
| `ovos.converse.pong` (CONVERSE-1 §4.2) | `reply` | Response back to the converse plugin that polled |
| `ovos.fallback.pong` (FALLBACK-1 §6.1) | `reply` | Claim or explicit decline, back to the fallback plugin |
| `ovos.common_query.pong` / `.response` (COMMON-QUERY-1 §6.2, §7.1) | `reply` | Claim and full answer, back to the common-query plugin |
| Pipeline introspection response (PIPELINE-1 §10.2) | `reply` | Response back to the observer that requested |

Every pong above is a `reply` of the round's ping, which is what
carries `context.utterance_id` back to the poller unchanged — the
correlation the poller uses to tell this round's answers from the
last one's (MSG-1 §4.2).

#### 3.1.3 No central correlation, no central state

The bus is **fully asynchronous**. OVOS does not centrally
correlate request/response chains, and does not centrally
track per-conversation state. There is no per-message
identifier, no in-reply-to field, no host-side index mapping
a `.response` back to its request, no shared "current
conversation" record.

The one identifier that does travel is
`context.utterance_id` (MSG-1 §4.2), and it does not
contradict any of that. It names an **interaction lifecycle** —
one utterance, one out-of-band query, one UI command — and it
is stamped once at lifecycle entry and copied by every
derivation. It is a value in the envelope, not an index:
nothing looks it up, nothing stores it, and a component that
correlates on it still keeps its own outstanding-request
table. What it removes is the need for each poll family to
invent a correlation key of its own; what it does *not* add is
a central registry of who is waiting for what.

`session.session_id` identifies an **interaction channel** —
nothing more. Two messages sharing a `session_id` are on the
same channel, but the spec guarantees nothing about ordering,
state continuity, or pending requests. Two messages sharing an
`utterance_id` are in the same lifecycle, which is a narrower
and stronger statement, and the only one either identifier
makes.

Every component — skills, pipeline plugins, external clients,
layer-2 systems — owns whatever state it needs. An asker that
wants `.response` correlation keeps its own outstanding-request
table; a skill that wants conversational memory keeps its own
per-session store; a layer-2 system that wants per-peer state
keys on `session_id`. Whatever a later consumer needs is **in
the Message** (`data` / `context` / `session`) or **out of
band** — never recovered from a hidden host-side index.

This is what lets layer-2 systems plug in cleanly: if OVOS
kept a central correlation index or a central conversation
state, every layer-2 system would have to replicate it, hook
into it, or work around it. Because OVOS keeps neither, they
compose without contention.

Several real concerns are deferred by this stance and are
listed under §7 (Known gaps): multi-turn conversation, the
other session knobs OVOS carries beyond `session_id`
and `lang` (`persona_id`, `time_format`, `date_format`,
`system_unit`, `tts_preferences`, …), and the shape
of conversational state. The async-by-default model means
those specs only need to define *what* the state is,
not *how* it travels.

#### 3.1.4 Layer-2 substrates

The single-flip routing model and the no-central-state
design make layer-2 federation composable without modifying
the assistant core. A remote peer is just another user-side
emitter: it sets `source` to its peer ID, populates `session`
with its own named session, and emits a Message. The
orchestrator runs the same `.reply` flip; response messages
carry `destination == peer ID`; the bridge (watching the bus)
routes them back over the transport. The
`session_id == "default"` rule keeps device-local TTS on the
device's speakers because remote sessions carry their own
`session_id` and never `"default"`.

Layer-2 bridges also inherit the session-field
**preference/policy split** without extra mechanism: client
sessions populate the preference fields
(`pipeline`, `<type>_transformers`) to request behaviour;
the bridge populates the policy fields
(`blacklisted_pipelines`, `blacklisted_<type>_transformers`)
from the peer's grant. PIPELINE-1 §5.5 and TRANSFORM-1 §5.3
compose them deterministically at the orchestrator.

### 3.2 The pipeline-plugin model

The piece that sits *around* the intent and bus stacks — the
multi-stage orchestrator that decides which engine even gets
a turn, runs `converse` / `fallback` / `common_query` / `ocp` /
`persona` stages, and produces the universal
`ovos.utterance.handled` end-marker — is what makes OVOS
structurally distinctive (HA and Rhasspy have no equivalent
layer).

The plugin abstraction is **realized in the reference
implementation**: `OVOSPipelineFactory` loads pipeline plugins
by id at startup, the orchestrator holds them in a
`pipeline_plugins` dict keyed on `pipeline_id`, and the default
`Session.pipeline` is an ordered list of plugin identifiers
(with a name-mapping that accepts both the short
`padatious_high`-style ids and their fully-qualified
`ovos-padatious-pipeline-plugin-high`-style forms). The
official `ovos-padatious-pipeline-plugin`,
`ovos-adapt-pipeline-plugin`, `ovos-converse-pipeline-plugin`,
`ovos-fallback-pipeline-plugin`,
`ovos-common-query-pipeline-plugin`,
`ovos-ocp-pipeline-plugin`, and the persona plugins all
conform to this model.

OVOS-PIPELINE-1 **formalizes** this model rather than
introducing a new abstraction. It:

- formalizes the plugin contract (the `match` shape, the
  `Match` result, the side-effect-free discipline);
- defines `<skill_id>:<intent_name>` **dispatch
  polymorphism** so a plugin can bundle its own handler (a
  language-model persona, a chatbot) as a first-class
  participant alongside skill-owned handlers;
- prescribes the **universal `ovos.utterance.handled`
  end-marker** on every terminal path;
- names the handler-lifecycle trio
  `ovos.intent.handler.*`.

The high/medium/low confidence-tier convention is
**compatible** with PIPELINE-1 and out of scope for the spec.
Each tier is a separate entry in the session's pipeline list
(e.g. `padatious_high`, `padatious_medium`,
`padatious_low`), but an entry is a reference to a match
configuration, not an actor: a plugin instance has one
`pipeline_id` whichever entry invoked it (PIPELINE-1 §3).
Whether a deployment runs one class with
`match_high` / `match_medium` / `match_low` methods or three
separate instances each with their own `pipeline_id` is
implementation choice the spec does not constrain.

Three properties make the resulting model unusually
expressive:

- **All plugins are equivalent.** No spec-level distinction
  between intent engines, converse handlers, fallbacks,
  language-model personas, classic chatbots, anything else.
  They all expose the same `match` contract.
- **Skills and plugin-bundled handlers are indistinguishable
  as handler owners.** From outside, the assistant
  responded — the user does not know or care whether a skill
  matched against a registered intent or a language-model
  plugin generated the response on the fly.
- **The engine-agnostic intent contract is
  realized**, not hypothetical. OVOS persona plugins
  (`ovos-persona`, `ovos-persona-server`,
  `ovos-claude-plugin`, `ovos-openai-plugin`, etc.) plug into
  the pipeline as first-class language-model stages. The
  ordered chain (deterministic keyword engines before fuzzy
  template engines before language-model fallbacks last) is
  also how the system *bounds* generalization in practice.

What OVOS-PIPELINE-1 deliberately leaves out: **per-plugin
behavioural contracts**. A `converse` plugin, a `fallback`
plugin, a persona plugin: each defines itself. PIPELINE-1
only defines the contract every plugin conforms to and the
universal utterance lifecycle around the iteration.

### 3.3 Interoperability with external protocols

The spec family does not define new transport protocols and
does not aim to replace existing ones. Where an external
voice-assistant protocol — Wyoming, OpenAI Chat Completions,
MCP tool calls, hassil templates, MQTT-based stacks — already
exists and serves a population, the spec family is designed to
**interoperate** with it through three well-defined injection
points. An adapter that plugs an external protocol into the
right injection point is a third-party implementation concern;
the spec family makes the integration shape predictable.

**1. Pipeline plugins (OVOS-PIPELINE-1 §3) — the dispatch-layer
adapter.** A pipeline plugin wraps an external matcher,
consumes the utterance, and returns a `Match` with the
plugin's own `pipeline_id` as `skill_id`. The external
protocol becomes a first-class participant in the dispatch
surface, indistinguishable from a skill from the bus's
perspective. This is how language-model APIs, deterministic
template matchers, and external intent classifiers attach.

**2. Transformer chains (OVOS-TRANSFORM-1 §3) — the
artifact-pipeline adapter.** A transformer wraps an external
protocol that operates on an audio, text, or rendered-output
artifact but does not claim intents. Examples: a
bidirectional-translation service at the utterance and dialog
chains; an external STT-confidence validator at the utterance
chain; a content-policy filter at the dialog or TTS chain; an
acoustic-event detector at the audio chain.

**3. Bus boundary (OVOS-MSG-1 §3.4) — the wire-level
adapter.** A bridge component subscribes to the bus, translates
to and from an external transport, and either operates entirely
external (Wyoming-style audio / STT / TTS services talking
over TCP to a bridge that proxies the OVOS bus) or remotes the
whole bus (HiveMind-style layer-2 substrates). The
single-flip routing of §3.1.1, the forward/reply discipline
of §3.1.2, and the no-central-state stance of §3.1.3 are what
make the bus-boundary adapter feasible without modifying the
assistant core.

#### Per-protocol notes

- **Wyoming** (the component protocol used by Home Assistant
  Voice and its ecosystem) operates at the audio-input / STT /
  intent / TTS service boundary. A Wyoming bridge sits at the
  bus boundary (§3.1, injection point 3 above): translate
  Wyoming's `transcript` event into an `ovos.utterance.handle`
  emission and translate the assistant's `speak` Messages
  into Wyoming's `synthesize` event. Pipeline plugins are
  unaffected; Wyoming components plug in *under* the
  utterance lifecycle, not into it.
- **OpenAI Chat Completions and compatible APIs** (the
  de-facto LLM interface). A persona-style pipeline plugin
  wraps an OpenAI-compatible client (§3 of PIPELINE-1,
  injection point 1 above). The plugin emits `Match` with
  `skill_id = <pipeline_id>` and bundles its own handler
  using the dispatch polymorphism of OVOS-PIPELINE-1 §7. The
  user sees a normal response; the LLM is a first-class
  intent owner.
- **MCP (Model Context Protocol) and similar agent-tool
  protocols.** A pipeline plugin can expose OVOS intents to
  an MCP client (the OVOS-INTENT-4 §10 introspection topics
  enumerate available intents) or call out to MCP tools from
  within a plugin-bundled handler. Either direction sits at
  injection point 1.
- **hassil templates and the Home Assistant `intents`
  corpus.** A pipeline plugin can wrap hassil as a
  deterministic template matcher (injection point 1).
  Separately, the OVOS-INTENT-1 / hassil grammar lineage is
  close enough that a **translation tool** between
  OVOS-INTENT-2 locale resources and HA's `intents` YAML is
  mostly mechanical — both formats are template-and-vocabulary
  YAML at the same level of abstraction. Such a tool would
  let the HA `intents` corpus and the OVOS locale corpus
  cross-pollinate without either project changing its
  format. This is a concrete tooling opportunity the shared
  grammar lineage invites, not just an architectural
  abstraction (§7).
- **MQTT-based stacks** (Rhasspy 2.x, miscellaneous IoT
  voice systems). Bridge at the bus boundary (injection
  point 3), same shape as Wyoming.
- **A2A and other agent-bus protocols.** Same shape as MCP;
  pipeline-plugin wrapper or bus-boundary bridge depending
  on whether the protocol participates in intent dispatch
  or in cross-process bus routing.

The three injection points are not exhaustive of where
adapters *could* go — a determined integrator can hook
almost anywhere — but they are the points the spec family
deliberately designs to keep clean. Any new protocol that
needs deeper integration than the three points permit is a
signal that the protocol genuinely overlaps the assistant's
own architecture rather than complementing it, at which
point the integration is a co-architecture decision rather
than an adapter.
