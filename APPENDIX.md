# Appendix — Design Notes and Context

**Non-normative.** This document is a companion to the OVOS formal
specifications. It records design rationale, comparisons with other
systems, the catalogue of *deliberate* divergences from current OVOS
code, and implementer-facing reference material that does not belong
in a normative specification body. Nothing here is binding — the
normative documents are OVOS-INTENT-1, OVOS-INTENT-2,
OVOS-INTENT-3, OVOS-INTENT-4, OVOS-MSG-1, OVOS-SESSION-1,
OVOS-PIPELINE-1, OVOS-CONTEXT-1, and OVOS-TRANSFORM-1.

Pointers to specific OVOS code (file paths, class names, function
names) and to specific real projects (HiveMind, Adapt,
padatious, ovos-audio, ovos-workshop, …) are deliberately kept
*out* of the spec bodies and collected here, because implementation
code moves and specifications must not.

---

## 1. About the OVOS specifications

### 1.1 Formalization of an existing system

The OVOS stack — the engines (padatious, Adapt), the skill
ecosystem, the resource file formats, the pipeline, the bus, the
session model — already exists and runs in production. The
specifications were written **after** the system they describe.
They are a *formalization pass*: they document an existing design
implementation-agnostically, tighten under-defined corners, and
remove accidental inconsistencies, so the contracts can be
implemented by new engines, new hosts, and adopted by other
assistants.

This matters for how to read them. They are **prescriptive** —
each spec states a clean target, and where it diverges from
current OVOS behaviour the divergence is a deliberate cleanup
(catalogued in §5) — but they are not speculative. The target is
a lightly-cleaned version of a working system, not a greenfield
design. `padacioso`, `ovos-workshop`, and `ovos-bus-client` are
the closest existing implementations; none yet fully conforms,
and bringing them into conformance is planned work. OVOS-MSG-1
is the closest to current code of all the specs — it is largely
a verbatim formalization of what `ovos-bus-client` already does.

### 1.2 The spec set, in three stacks

The specifications are built bottom-up in three stacks:

- **The intent stack**, in dependency order: OVOS-INTENT-1
  (template grammar) → OVOS-INTENT-2 (resource files) →
  OVOS-INTENT-3 (the intent concept) → OVOS-INTENT-4 (the
  registration wire format on the bus).
- **The bus stack**: OVOS-MSG-1 formalizes the envelope, routing,
  session carrier, and `forward`/`reply`/`response` derivations.
  OVOS-SESSION-1 formalizes the wire shape of the session
  carrier and the field-registry mechanism by which other specs
  claim session fields.
- **The orchestrator stack**: OVOS-PIPELINE-1 defines the
  orchestrator, the pipeline-plugin abstraction, the utterance
  lifecycle, and the handler-lifecycle trio. OVOS-CONTEXT-1
  defines per-session intent-context state (the **declarative**
  continuous-dialog primitive). OVOS-CONVERSE-1 defines the
  active-handler recency stack, the converse plugin role, and
  the interactive response-collection mechanism (the
  **imperative** continuous-dialog primitive, complementary to
  CONTEXT-1 — its §7 fixes the evaluation order between the two
  surfaces). OVOS-TRANSFORM-1 defines the six injection-point
  transformer chains. OVOS-SESSION-2 defines the session
  lifecycle and state-ownership model (stateless orchestrator
  for named sessions, orchestrator-owned default session,
  SHOULD-project pathway for cross-utterance state with
  MAY-internal as the alternative for state too large or
  externally coupled to project). The orchestrator stack sits on top
  of the bus stack (uses MSG-1's envelope and routing,
  SESSION-1's session carrier with SESSION-2's lifecycle) and
  around the intent stack (intent registrations are one kind
  of input pipeline plugins consume).

### 1.3 Compatibility levels

Each specification carries its own integer `Version`, bumped per
PR per the contributing rules in the README. The architecture as
a whole is spoken of at **compatibility levels** — versioned
snapshots a tool may target, checked against by
`ovos-spec-lint`.

The compatibility-level model works cleanly for the **intent
stack**, where a single integer identifies a coherent grammar /
resources / intent-definition snapshot. The bus and orchestrator
stacks do not yet map onto the same single-axis ladder; a
specification-set-wide version tuple covering every spec is a
planned follow-up.

The intent-stack ladder:

- **V0** — *informal.* The undocumented, de-facto behaviour from
  before these specifications existed. V0 is not specified
  anywhere; it is the baseline the formalization started from.
  V0 has no notion of the `.blacklist` resource role or of
  `<name>` references.
- **V1** — the intent stack as first formalized:
  OVOS-INTENT-1, -2 and -3, each at version 1. V1's headline
  addition over V0 is the `.blacklist` role.
- **V2** — V1 plus **inline vocabulary references** (the
  `<name>` token): OVOS-INTENT-1 and OVOS-INTENT-2 at version 2.
  A V2 template cannot be expanded by a V1 tool.

The bus stack (OVOS-MSG-1), the registration spec
(OVOS-INTENT-4), the session spec (OVOS-SESSION-1), the
orchestrator spec (OVOS-PIPELINE-1), the context spec
(OVOS-CONTEXT-1), and the transformer spec (OVOS-TRANSFORM-1)
are versioned **individually** and not placed on a unified
compatibility ladder. A tool targeting them today cites per-spec
versions: "MSG-1 v2, INTENT-4 v1, PIPELINE-1 v2." Whether the
compat-level model evolves into a multi-axis grid, per-stack
ladders, or is quietly deprecated in favour of per-spec
versions only, is deferred.

### 1.4 Reference implementations and ecosystem tooling

The **reference implementation for the intent stack** is
**`ovos-spec-tools`** — expander, resource loader, dialog
renderer, language matching, locale linter — in one
dependency-light Python package. New tools that consume locale
folders or expand templates should depend on it rather than
reimplementing.

The bus and orchestrator stacks do not yet have a comparable
ground-up reference implementation; `ovos-bus-client` is the
closest match for OVOS-MSG-1 and `ovos-core` is the closest
match for OVOS-PIPELINE-1 + OVOS-INTENT-4, but both predate the
specs.

**`ovos-localize`** is the i18n-operation layer atop the intent
stack: a GitHub-native localization platform for OVOS skills,
built specifically around the resource roles of OVOS-INTENT-2.
It scans skill repositories for locale files; analyzes each
skill's Python source (via AST) to recover the **handler
context** of a resource — which function uses a file, what its
slots mean, what dialog it triggers, which is exactly the
intent↔handler binding of OVOS-INTENT-3 §1; validates
translations against a rule set (slot preservation, expansion
validity, variant counts); and lets translators browse, edit,
preview, and submit translations as pull requests. It is the
OVOS counterpart to Home Assistant's managed `intents`
repository.

Two honest notes on `ovos-localize`: it is currently
**descriptive** of real OVOS skills — it also handles legacy
file types these specs deliberately drop — so as the specs and
the ecosystem converge, its file-type coverage and the specs
will need to meet in the middle; and its translation validators
are a natural home for spec conformance checks, distinct from
but related to the planned grammar-level conformance corpus
(§7).

---

## 2. Comparison with other voice-assistant systems

The OVOS specifications occupy territory adjacent to several
existing voice-assistant systems. This section locates the
design choices against each comparator. The summary in §2.7
records where OVOS leads architecturally, where it follows, and
where it makes a deliberately different choice.

### 2.1 Home Assistant and Rhasspy — shared grammar lineage

OVOS, Home Assistant (HA), and Rhasspy share a common lineage.
The bracket-expansion grammar of OVOS-INTENT-1 — `(a|b)`
alternatives, `[optional]` segments, `{slot}` placeholders — is
the same family as HA's `hassil` sentence templates and
Rhasspy's `sentences.ini`. The *syntax* is not novel. What is
distinctive about the OVOS approach is everything around the
grammar.

**What OVOS does differently:**

- **An implementation-agnostic spec at all.** HA and Rhasspy
  have no format-level specification independent of their
  implementation — the code is the contract. OVOS now has one,
  which is what lets multiple engines (and other assistants)
  implement the same contract.
- **Engine-agnostic matching.** OVOS-INTENT-1 §4 treats
  templates as *training data* and leaves matching, scoring,
  and generalization to the engine. HA's core matching is
  `hassil`, a deterministic template matcher; Rhasspy compiles
  templates into a closed ASR grammar. The OVOS contract
  accommodates a deterministic matcher, a neural classifier,
  or an LLM behind one interface.
- **Templates are training data, not a closed grammar.** A
  capable OVOS engine generalizes beyond the authored samples.
  Rhasspy's closed-grammar model is deterministic and
  offline-guaranteed but brittle — an utterance not derivable
  from `sentences.ini` cannot be recognized at all.
- **A multi-stage pipeline** (§3.2). Intent engines are two
  stage kinds among many. Neither HA nor Rhasspy exposes an
  intent layer this structured.
- **An intent is bound to one handler, owned by one skill**
  (OVOS-INTENT-3 §1). See §2.2 — this follows necessarily from
  the open skill ecosystem.
- **A bus substrate openable to layer-2 systems** (§3.1).
  Neither HA nor Rhasspy exposes their bus this openly.

**What HA and Rhasspy do better:**

- **Reusable template fragments.** `hassil` has
  `expansion_rules` and Rhasspy has `<rule>` references —
  named, reusable sub-templates that let authors share common
  fragments (politeness prefixes, articles, recurring
  phrasings). OVOS-INTENT-1 version 2 closes this with the
  `<name>` inline vocabulary reference, which expands a named
  `.voc` in place — reusing the existing slot-free format
  rather than adding a new construct.
- **i18n corpus maturity.** HA's community `intents`
  repository is a large, managed, professionally-translated
  corpus covering many languages. OVOS has the tooling
  counterpart in `ovos-localize` (§1.4) — so the gap here is
  the *scale and maturity* of the corpus, not the absence of
  tooling.
- **Concrete, testable completeness.** HA and Rhasspy ship
  systems where the hard parts — matching, number and range
  handling, slot typing — are solved concretely. The OVOS
  specs deliberately defer some of these (slot typing to a
  future normalization spec; matching to the engine). That
  deferral is intellectually consistent but means the specs'
  value depends on the engines and tooling that fill the gaps.

### 2.2 Closed domain vs open ecosystem

The sharpest difference between OVOS and HA is not technical
but structural. **Home Assistant is a curated, closed domain**:
home automation, with a vendor-managed intent vocabulary. HA
can treat an intent such as `HassTurnOn` as a *shared contract*
honoured uniformly across hundreds of integrations and many
languages, because HA controls and curates that vocabulary.

**OVOS is an open ecosystem.** Skills are arbitrary third-party
Python packages, installed by pip, developed independently,
running as arbitrary code in process. A skill can do anything;
OVOS voice-enables anything. In that setting a shared global
intent vocabulary is not a missing feature — it is incoherent.
When skills are unbounded, an intent *must* be private to the
skill that defines it and bound directly to that skill's
handler. OVOS-INTENT-3's "an intent is not an event" stance is
therefore the correct model for an open ecosystem, just as HA's
shared-vocabulary model is correct for a curated one. The two
models are right for different platforms; neither is
universally better.

### 2.3 Rasa — closest comparator for intent context

Rasa's "active forms" and slot mappings perform context-aware
matching, but they are baked into the policy engine; you
cannot run a Rasa NLU pipeline without Rasa policies.
OVOS-CONTEXT-1 separates **gating** (`requires_context` /
`excludes_context`, §6 / §6.1 of that spec) from **match-time
capture** (the context-supplied capture rule, §7) from **engine
matching hints** (engine-internal use of values, §6), so every
intent engine that consumes OVOS-INTENT-3 registrations can
gate uniformly without buying into a particular dialog policy.

Rasa wins on conversation-level evaluation infrastructure —
story-based testing, end-to-end success metrics — for which
the OVOS specs have no analogue yet (§7 catalogues this as a
known gap).

Rasa's NLU pipeline is also the closest analogue to
OVOS-TRANSFORM-1's utterance / metadata / intent chains, but
it is a single sequence per language model and the
policy/preference split (TRANSFORM-1 §5.3) does not exist.
TRANSFORM-1's six-injection-point model is genuinely more
expressive.

### 2.4 Amazon ASK / Alexa Skills Kit, Google Dialogflow

Both are closed-domain centrally-trained stacks. Their
built-in entity-type systems (`AMAZON.DATE`,
`@sys.date-time`) are what OVOS-TRANSFORM-1 §3.4 replicates as
an *injectable, deployer-replaceable, engine-agnostic*
contract — at the spec level OVOS is strictly more flexible,
though OVOS defers the **typed value formats themselves**
(date encoding, number representation, duration units) to a
future text-normalization spec (§7), while ASK and Dialogflow
ship them as built-ins.

Neither ASK nor Dialogflow has a `session.pipeline`-equivalent
(the assistant picks one matcher per skill); neither has
anything like the layer-2 substrate of OVOS-MSG-1 §3.4. ASK
has built-in intents (`AMAZON.HelpIntent`) but they are
handled inside the skill; Dialogflow has fallback intents but
they do not have first-class dispatch identity. OVOS-PIPELINE-1's
`<pipeline_id>:<intent_name>` lets a non-skill component
advertise its own intent identity *to the user* on the bus,
indistinguishable from a skill — original to OVOS.

### 2.5 hassil — comparable only at the grammar layer

The Home Assistant template-matcher, comparable only to
OVOS-INTENT-1 / -2 / -3 (grammar + locale resources + intent
concept). hassil has no equivalent of OVOS-MSG-1 (no bus
envelope), OVOS-PIPELINE-1 (no pipeline notion — HA runs a
single matcher), OVOS-TRANSFORM-1 (no per-utterance
transformers), or OVOS-CONTEXT-1 (no decaying session state).
The grammar layer is broadly equivalent to OVOS-INTENT-1;
everything above the grammar is OVOS-only.

### 2.6 Summary — where OVOS leads, follows, and differs

**OVOS leads architecturally** in three places:

- **The pipeline-plugin model with first-class dispatch
  polymorphism.** No comparator lets a non-skill component
  (LLM persona, chatbot, fallback) be a first-class handler
  owner on the same dispatch surface.
- **The six-injection-point transformer chain with per-session
  preference/policy separation.** Nothing in HA, Rhasspy,
  Rasa, ASK, or Dialogflow has a comparable lifecycle-uniform
  extensibility surface.
- **Negative gating (`excludes_context` "match if absent")
  in CONTEXT-1.** ASK/Dialogflow contexts are purely
  positive; Rasa forms are not engine-agnostic; HA has no
  context model. The fire-once and modal-suppression patterns
  fall out of negative gating.

**OVOS follows** where ecosystem investment matters more than
architecture:

- HA's translation corpus scale (the `intents` repository).
- ASK / Dialogflow's typed entity systems.
- Rasa's conversation-level evaluation infrastructure.

**OVOS makes a deliberately different choice** in two places:

- *Engine-agnostic templates as training data* (OVOS-INTENT-1
  §4) rather than Rhasspy-style closed grammars. The trade-off:
  generalization beyond authored samples vs. offline-deterministic
  recognition guarantees.
- *Open skill ecosystem with skill-private intents*
  (OVOS-INTENT-3 §1) rather than HA-style curated vocabulary.
  The trade-off: skill author freedom vs. cross-integration
  vocabulary sharing.

---

## 3. Architectural patterns

Two patterns recur across the spec family and are worth a
dedicated treatment.

### 3.1 The bus as a substrate

Under OVOS-MSG-1's `source` / `destination` / `session` model,
the bus is not just an internal transport — it is the
**substrate higher-level systems plug into without modifying
the assistant core**. Two mechanics make that work:
**single-flip routing** (§3.1.1), which keeps the routing pair
correct end-to-end without per-component effort; and **no
central state or correlation** (§3.1.2), which makes layer-2
systems composable. HiveMind is the canonical example of what
both together enable (§3.1.3).

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

#### 3.1.2 No central correlation, no central state

The bus is **fully asynchronous**. OVOS does not centrally
correlate request/response chains, and does not centrally
track per-conversation state. There is no per-message
identifier, no in-reply-to field, no host-side index mapping
a `.response` back to its request, no shared "current
conversation" record.

`session.session_id` identifies an **interaction channel** —
nothing more. Two messages sharing a `session_id` are on the
same channel, but the spec guarantees nothing about ordering,
state continuity, or pending requests.

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
other session knobs current OVOS carries beyond `session_id`
and `lang` (`persona_id`, `time_format`, `date_format`,
`system_unit`, `tts_preferences`, …), and the eventual shape
of conversational state. The async-by-default model means
those future specs only need to define *what* the state is,
not *how* it travels.

#### 3.1.3 Why HiveMind works

HiveMind is the canonical layer-2 system this design enables.
A HiveMind satellite is just another user-side emitter — it
sets `source` to its peer ID, populates `session` with a
per-peer session, and emits a Message. Inside OVOS:

- ovos-core runs the same `.reply` flip (§3.1.1 step 2) —
  `destination` becomes the satellite's peer ID instead of
  the local microphone.
- Skills `.forward` as usual — `destination` stays the
  satellite ID through every handler emission.
- HiveMind, watching the bus, sees each message addressed to
  its peer and routes it back over the HiveMind transport.

The pre-existing `session_id == "default"` rule keeps
device-local TTS on the device's speakers (per
`ovos-audio/utils.py`'s `require_default_session`), because
remote HiveMind sessions carry their own `session_id` and
never `"default"`.

None of this required HiveMind to modify OVOS core. The
mechanism that makes it work — single-flip routing, opaque
per-session identifiers, no central state — was an OVOS
design, built into `ovos-bus-client/message.py:194-198`
before this spec family was written; OVOS-MSG-1 formalizes
the design rather than introducing it.

A layer-2 substrate also has a uniform **authorization
surface** in the spec family without inventing a separate
channel: client sessions populate the preference fields of
OVOS-SESSION-1 (`pipeline`, the six `<type>_transformers`)
to request behaviour, while the layer-2 substrate populates
the policy fields (`blacklisted_pipelines`,
`blacklisted_skills`, `blacklisted_intents`, the six
`blacklisted_<type>_transformers`) from the peer's grant.
OVOS-PIPELINE-1 §5.5 and OVOS-TRANSFORM-1 §5.3 compose them
deterministically (preference → availability → policy) at the
orchestrator without per-hop re-authorization.

### 3.2 The pipeline-plugin model

The piece that sits *around* the intent and bus stacks — the
multi-stage orchestrator that decides which engine even gets
a turn, runs `converse` / `fallback` / `common_query` / `ocp` /
`persona` stages, and produces the universal
`ovos.utterance.handled` end-marker — is what makes OVOS
structurally distinctive (HA and Rhasspy have no equivalent
layer).

The plugin abstraction is **already in current code**:
`OVOSPipelineFactory` loads pipeline plugins by id at startup,
the orchestrator holds them in a `pipeline_plugins` dict
keyed on `pipeline_id`, and the default `Session.pipeline` is
an ordered list of plugin identifiers (with a migration map
translating legacy `padatious_high`-style names into modern
`ovos-padatious-pipeline-plugin-high`-style ones). The
official `ovos-padatious-pipeline-plugin`,
`ovos-adapt-pipeline-plugin`, `ovos-converse-pipeline-plugin`,
`ovos-fallback-pipeline-plugin`,
`ovos-common-query-pipeline-plugin`,
`ovos-ocp-pipeline-plugin`, and the persona plugins all
already conform to this model.

OVOS-PIPELINE-1's contribution is therefore a **prescriptive
refinement**, not a wholesale new abstraction. It:

- formalizes the plugin contract (the `match` shape, the
  `Match` result, the side-effect-free discipline);
- defines `<owner_id>:<intent_name>` **dispatch
  polymorphism** so a plugin can bundle its own handler (a
  language-model persona, a chatbot) as a first-class
  participant alongside skill-owned handlers;
- prescribes the **universal `ovos.utterance.handled`
  end-marker** on every terminal path;
- renames the `mycroft.skill.handler.*` trio →
  `ovos.intent.handler.*`.

The current high/medium/low confidence-tier convention is
**compatible** with PIPELINE-1 and out of scope for the spec.
From the bus's perspective each tier is already a distinct
`pipeline_id` in the session's pipeline list (e.g.
`padatious_high`, `padatious_medium`, `padatious_low`), which
is exactly what the spec prescribes. How a Python plugin
class internally serves multiple `pipeline_id`s — one class
with `match_high` / `match_medium` / `match_low` methods,
three separate plugin instances, an orchestrator-side
suffix-decoding helper — is implementation choice the spec
does not constrain.

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
- **The engine-agnostic intent contract is already
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
plugin's own `pipeline_id` as `owner_id`. The external
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
single-flip routing of §3.1.1 and the no-central-state stance
of §3.1.2 are what make the bus-boundary adapter feasible
without modifying the assistant core.

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
  `owner_id = <pipeline_id>` and bundles its own handler
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
  format. This is concrete planned tooling, not just an
  architectural possibility (§7).
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

---

## 4. Design rationale, per specification

Short notes on *why* the specifications make the choices they
do — the reasoning, not the requirement. Cross-reference into
the normative sections.

### 4.1 Intent grammar and resources (INTENT-1, -2, -3)

- **ASR-normalized input, no escaping** (INTENT-1 §2). The
  grammar targets voice input. By contract, text reaching an
  engine is already lowercased, punctuation-stripped,
  single-spaced. Bracket metacharacters therefore cannot
  occur as literal input, so no escape mechanism is needed.
  A simplification *bought* by scoping the grammar to voice.
- **Templates are training data** (INTENT-1 §4). Enumerating
  every phrasing is futile for natural speech. A template
  describes the *shape* of the training data; the engine
  generalizes. This is why expansion is defined precisely
  but matching is not.
- **An intent is not an event** (INTENT-3 §1). Necessary for
  an open skill ecosystem — see §2.2.
- **Two non-interoperable methods** (INTENT-3 §2). Keyword
  and template intents describe a command in fundamentally
  different shapes. Rather than forcing one model, the spec
  keeps both and makes engines declare which they accept.
  The cost is that a developer must choose per intent and
  know which engines an installation runs.
- **Slot typing is deferred** (INTENT-1 §5.3). Interpreting
  a slot value as a number or date is inseparable from how
  ASR output is normalized — and normalization is not yet
  specified. Specifying typing first would be incoherent, so
  a value is, for now, an opaque sequence of words.
- **`.blacklist` vs `excluded`** (INTENT-3 §4.2, §5.4). The
  template grammar is purely generative — it cannot express
  "not this". Template intents therefore need a separate
  `.blacklist` artifact for suppression. Keyword intents
  express the same idea natively with the `excluded`
  constraint role. The asymmetry follows from the grammar,
  not from inconsistency.
- **No regular expressions** (INTENT-3 §4.4). Free-form
  structured text is a slot — use a template intent and the
  slot extractor. Regexes are also notoriously hard to
  localize, which conflicts with the per-language model.
- **Inline vocabulary references reuse `.voc`** (INTENT-1
  §3.7). A reusable template fragment and a keyword
  vocabulary are the same thing — a named, slot-free phrase
  set — so `<name>` resolves to a `.voc` rather than
  introducing a new file role. The change is one grammar
  token plus an expander step.

### 4.2 Bus message envelope (MSG-1)

- **One spec, not two.** Envelope + routing + derivations
  are tightly coupled — every routing key lives in
  `context`, every derivation manipulates routing, and all
  of them formalize *existing* OVOS code. Splitting them
  was tried; the split did not survive the derivations
  (which can only meaningfully be defined where the routing
  keys are), so they were merged into a single bus-message
  spec. The session carrier, by contrast, did split out
  cleanly into OVOS-SESSION-1.
- **`context` is extensible by design.** Only the keys
  other systems already key behaviour off (`source`,
  `destination`, `session`) are given normative meaning.
  Everything else — GUI routing, tracing, security — is
  layered by other specs without touching the envelope.
- **`source`/`destination` are informational, not
  authorization** (MSG-1 §3.3). The bus is not a security
  boundary. Layer-2 systems (HiveMind) build authentication
  and routing enforcement on top of the pair without OVOS
  itself learning about peers.
- **The boundary is user ↔ assistant, not core ↔ handler.**
  The `(source, destination)` pair marks who is currently
  talking to whom across one boundary only: the external
  participant on one side, the assistant — core and every
  skill handler — on the other. The flip happens **once**
  per conversational turn (§3.1.1), not on every internal
  hop.
- **No central correlation, no central state** (MSG-1 §5.4,
  §3.1.2 above). The bus is fully asynchronous. Components
  that need correlation or state own it themselves, keyed
  on `session.session_id`. Multi-turn conversation, intent
  context, cross-skill state, and similar concerns are
  deferred to other specifications.
- **Topic naming conventions** (MSG-1 v2 §2.1.2). The
  conventions other specs in the family already follow are
  now codified as SHOULD-rules: dot-separated hierarchy
  with `:` reserved for component-pair shapes; stable
  ecosystem-identifying root; verb-tense pattern for the
  trailing segment; request/terminal pairs sharing a root
  verb (`handle` ↔ `handled`); `.response` suffix for
  response derivations; per-instance
  `<root>.<domain>.<id>.<verb>` form.

### 4.3 Session carrier (SESSION-1)

- **Why a separate session spec.** `Message.context.session`
  is a load-bearing carrier claimed by multiple specs
  (PIPELINE-1, CONTEXT-1, TRANSFORM-1) — without a single
  owner, its wire contract drifts. SESSION-1 consolidates
  the wire shape and fixes a **registry mechanism** so
  future specs claim fields without amending SESSION-1
  itself.
- **Prescriptive, not descriptive.** Only the fields
  normatively claimed by other specs are recognized.
  Implementations carrying extra per-session state
  (current OVOS Session has `persona_id`, `system_unit`,
  `time_format`, `date_format`, `location`, `is_speaking`,
  `is_recording`, …) are non-normative under v1 — they
  ride through as opaque pass-through and can be claimed
  by future per-domain specs.
- **Omission means "let the orchestrator decide".** Single
  deferral mechanism: omitted single field, empty
  `session: {}`, absent `session`, explicit
  `session_id: "default"` — all equivalent on the wire,
  all resolve at consumption to deployment defaults filled
  by each consumer. No `null`, no sentinels.
- **Language signals.** Six BCP-47 fields with normative
  meanings but stage-dependent consolidation: `lang` (user
  preference, base), `secondary_langs` (additional
  understood languages, constrains lang-detect predictions
  and fallback selection), `output_lang` (renderer's
  preferred output language; simplifies the
  bidirectional-translation transformer to a fallback role),
  `stt_lang` / `request_lang` / `detected_lang`
  (per-utterance signals from STT, emitter, and lang-detect
  respectively). `request_lang` is an emitter-reported hint
  (per-wakeword language assignment in multi-wakeword
  setups), not an override.

### 4.4 Intent registration broadcast (INTENT-4)

- **Registrations are broadcast — already how OVOS works.**
  Skills emit registration messages on the bus; plugins
  that care about a particular registration kind subscribe
  to the corresponding topic. There has never been a
  central routing party in OVOS; INTENT-4 just gives this
  existing model normative topic names. The legacy bus
  topics (`padatious:register_intent`, `register_vocab`,
  etc.) are renamed into the `ovos.intent.*` namespace —
  see §5.7 for the mapping. Migration is mostly a string
  replacement.
- **No "no plugin claimed" error.** Following from the
  broadcast model: a registration that no plugin consumes
  is silently dropped. The producer gets no signal — the
  introspection topics (`ovos.intent.list` /
  `ovos.intent.describe`) are the supported way to verify
  what the orchestrator's passive index recorded.
- **The orchestrator passively indexes; it does not
  gate.** The introspection topics serve from a passive
  registration index built by listening to broadcasts
  (this *is* new — current OVOS has no central index). The
  index reflects what skills *declared*, not what plugins
  actually match against — observability-only.
- **Skill self-identification on every emission**
  (INTENT-4 §3.1). Every Message a skill emits or
  modifies in place carries `Message.context["skill_id"]`.
  Enforcement is structural on the dispatch path: the
  orchestrator stamps `context.skill_id` from the
  `<skill_id>:<intent_name>` dispatch topic prefix
  (PIPELINE-1 §7.1), and skill emissions via
  `forward`/`reply` inherit automatically.

### 4.5 Pipeline and lifecycle (PIPELINE-1)

- **The plugin model is already in place; PIPELINE-1
  refines it** (§3.2). The current orchestrator already
  loads plugins by id through `OVOSPipelineFactory` and
  iterates `Session.pipeline`. PIPELINE-1 tightens the
  contract rather than introducing the abstraction.
- **Orchestrator and plugin contracts live in one spec**,
  since the orchestrator's job *is* iterating plugins and
  translating their matches into bus events. Splitting
  them would leave neither coherent.
- **Plugin contract is minimal.** `match(utterance, lang,
  session) → Match | None`. Side-effect-free during
  `match`; everything else (state, registrations,
  language-model calls, response generation) is
  plugin-internal black box. The smaller the contract, the
  wider the set of plugins it accommodates.
- **`lang` parameter is propagation-only.** The
  orchestrator passes `lang` through from
  `Message.data.lang`; it **MUST NOT** synthesize a value
  from `session.lang` or any per-utterance signal field
  when `data.lang` is absent. Absence is a faithful
  "unknown" signal; consumer-side fallback policy is the
  consumer's.
- **Tier conventions are out of scope.** The current
  high / medium / low suffix is implementation strategy:
  from the bus, each tier is already a distinct
  `pipeline_id` in `Session.pipeline`. The current
  convention is compatible with PIPELINE-1 unchanged.
- **Skills and plugins are equivalent handler owners.**
  Dispatch topic `<owner_id>:<intent_name>` polymorphism
  (owner is `skill_id` or `pipeline_id`) lets a plugin
  bundle its own handler — for example, a language-model
  persona plugin that has no skills behind it — and still
  be addressed uniformly.
- **Universal `ovos.utterance.handled` end-marker on every
  terminal path.** One reserved invariant lets observers
  count turns, route fallbacks, and know "the assistant
  is idle now" without per-stage knowledge.
- **Three-stage composition** (PIPELINE-1 §5.5) —
  preference (from `session.pipeline` or default-session
  pipeline) → availability (drop unloaded plugins) →
  policy (drop denylisted). Mirrors TRANSFORM-1 §5.3
  exactly. The same shape supports the
  client-requests/layer-2-enforces split (§3.1).

### 4.6 Intent context (CONTEXT-1)

- **Lifts intent context out of Adapt.** The Adapt-specific
  `add_context` / `remove_context` mechanism, and the
  legacy `mycroft.skill.set_cross_context` /
  `remove_cross_context` fan-out for cross-skill use, are
  Adapt-only at the matcher level — Padatious and other
  engines ignore them. CONTEXT-1 generalizes the mechanism
  into a session-bound, decaying flat key/value store
  consumed by every intent engine uniformly via
  `requires_context` and `excludes_context` declarations.
- **Two explicit scopes encoded in the key shape.**
  `private` (orchestrator auto-prefixes with
  `<owner_id>:`) and `shared` (flat, cross-skill). The
  current OVOS code models the same distinction informally
  (`MycroftSkill.set_context` auto-prefixes with
  `alphanumeric_skill_id`; `set_cross_skill_context` fans
  out via a bus event); CONTEXT-1 names the scopes
  explicitly and routes both through one bus surface.
- **Why private is the default.** A skill that calls
  `ovos.context.set` without specifying `scope` gets a
  private entry. This optimises for the safer case: a
  cross-skill leak from an accidentally-shared entry is
  harder to debug than a cross-skill miss from an
  accidentally-private entry. The current Adapt
  `set_context` pattern is effectively skill-private; the
  default preserves migration fidelity. Cross-skill
  coordination is a conscious decision that deserves an
  explicit `scope: "shared"`.
- **Prior art for the negative gate.** Three in-tree
  intent engines under `/plugins-pipeline/` —
  [jurebes](https://github.com/OpenJarbas/jurebes),
  [nebulento](https://github.com/OpenJarbas/nebulento),
  and [palavreado](https://github.com/OpenJarbas/palavreado)
  — independently implement `exclude_context` as a
  first-class negative gate. CONTEXT-1's `excludes_context`
  adopts the same primitive at the spec level, addressing
  patterns ("fire once", "modal suppression") that
  positive gating alone cannot express.
- **Engine-side mutation as a sanctioned non-bus
  pathway.** The Adapt pipeline plugin auto-injects matched
  entities into context *inside* `match()`, which conflicts
  with PIPELINE-1 §4.2's side-effect-free `match` rule.
  CONTEXT-1 §5.3 carves an explicit window between
  match-accept and dispatch-emit for engine-side session
  mutation, with the orchestrator (not the bus) carrying
  the write. This both legitimizes the established
  practice and resolves the PIPELINE-1 contradiction.
- **Eight-level lifecycle-position owner precedence**
  (CONTEXT-1 §5.2). When a Message carries multiple
  component-identity keys (skill_id, pipeline_id, the six
  `<type>_transformer_ids`) from a derivation chain that
  crossed component boundaries, the orchestrator picks the
  owner by lifecycle position: the latest stage to run is
  the most specific.

### 4.7 Transformer plugins (TRANSFORM-1)

- **Spec'd as an architectural pattern, not a feature
  list.** An orchestrator MAY implement chains at any
  subset of six injection points (audio, utterance,
  metadata, intent, dialog, TTS); a null-implementation is
  conformant. For each chain it does implement, the
  per-type contract binds. Each injection point's
  existence is justified by what the lifecycle holds at
  that exact moment — what's possible there that isn't
  possible elsewhere.
- **Intent transformers as the system-typing home.**
  INTENT-1 §5.3 defers slot value typing pending a text
  normalization specification. TRANSFORM-1 §3.4 is the
  spec'd injection home for typing: a deployer ships
  date / number / duration parsing once, and every skill
  receives typed values in `Match.captures` regardless of
  which engine matched. The OVOS analogue of ASK's
  `AMAZON.DATE` and Dialogflow's `@sys.date-time`, but as
  an injected enrichment rather than a built-in engine
  feature.
- **Concrete in-tree plugins as prior art.** Nine plugins
  live under `/plugins-transformer/` today, covering five
  of the six injection points: utterance transformers
  (`ovos-utterance-normalizer`,
  `ovos-utterance-corrections-plugin`,
  `ovos-transcription-validator-plugin`,
  `ovos-utterance-plugin-cancel`,
  `ovos-bidirectional-translation-plugin`); dialog
  transformers (`ovos-dialog-normalizer-plugin`,
  `ovos-bidirectional-translation-plugin`,
  `ovos-dialog-transformer-openai-plugin`); audio
  transformers
  (`ovos-audio-transformer-plugin-speechbrain-langdetect`,
  `ovos-audio-transformer-plugin-ggwave`,
  `ovos-audio-transformer-redis-publish`); intent
  transformers (`ovos-keyword-template-matcher`,
  `ovos-ahocorasick-ner-plugin`). The
  `bidirectional-translation` plugin exercises the
  cross-chain coordination via `Message.context` that
  TRANSFORM-1 §7 formalizes.
- **Ascending priority.** TRANSFORM-1 §4 specifies
  ascending priority (lower = earlier, default 50).
  Current OVOS sorts transformer chains **descending**
  (`ovos_core/transformers.py:53,117,205`, `reverse=True`);
  the spec aligns with the **ascending** convention
  already used by fallback skills (`fallback_service.py:49`,
  default 101 = run last) and the natural "stages count
  up" reading. Bringing current plugins into conformance
  only requires flipping relative priorities, not
  rewriting.
- **Cancellation aligned with prior plugin convention.**
  Two existing utterance transformers
  (`ovos-utterance-plugin-cancel`,
  `ovos-transcription-validator-plugin`) already signal
  the lifecycle should abort by returning empty utterance
  lists with `{canceled: true, cancel_word: <reason>}`
  context keys. TRANSFORM-1 §8 keeps the convention,
  renaming `cancel_word` to `cancel_reason` (the structured
  concept the field encodes) and adding orchestrator-stamped
  `cancel_by: <transformer_id>`. The spec's
  `ovos.utterance.cancelled` terminal event sits alongside
  `complete_intent_failure`, keeping cancellation and
  failure observably distinct on the bus.
- **`lang` parameter is bidirectional** (TRANSFORM-1 §3.0).
  Four of the six per-type contracts (audio, utterance,
  dialog, TTS) take `lang` as input and return it as
  output. A bidirectional-translation transformer that
  takes Spanish in and produces English out returns the
  destination language; the orchestrator writes the
  chain's final `lang` back into `Message.data.lang` for
  downstream stages. Language-detector and clearing cases
  fall out of the same channel.
- **Per-type self-identification keys, list-valued.**
  TRANSFORM-1 §1.3 claims six `Message.context` keys — one
  per transformer type (`audio_transformer_ids`, …,
  `tts_transformer_ids`) — rather than a single generic
  key. Two reasons. First, role matters: a Message at the
  dialog stage may have been touched by five transformer
  types in sequence, and lumping them into one slot loses
  the role partitioning that exists in every other
  surface of the spec (per-type registries, per-type
  `*_transformers` overrides, per-type introspection
  topics). Second, multi-type plugins disambiguate: a
  plugin shipping both an utterance and a dialog
  transformer under the same `transformer_id` would, with
  a single generic key, leave consumers unable to tell
  which role emitted. The keys are *lists*, not single
  strings, because transformers chain by design — the
  list preserves the full per-type chain on the wire in
  order of touch.
- **Per-type denylists complete the policy surface.**
  TRANSFORM-1 §5.2 claims six
  `blacklisted_<type>_transformers` session fields,
  paralleling the six `<type>_transformers` chain-ordering
  fields of §5.1 and the
  `pipeline` / `blacklisted_pipelines` pair of PIPELINE-1
  §5. Three-stage composition (preference → availability
  → policy) in §5.3 mirrors PIPELINE-1 §5.5 exactly.
- **The per-type "explosion" is deliberate.** Counting
  transformer-related session-field claims: six chain
  orderings + six denylists = twelve fields, plus six
  `Message.context` attribution keys. The alternative — a
  `transformer_<type>:<name>` prefix-encoded single
  namespace — would require prefix parsing at every
  lookup. The per-type partition matches the partitioning
  that already exists in the §1.1 registries, the §4
  chain ordering rules, and the §6 introspection topics.
  Under the canonical SHOULD-omit rule of SESSION-1 §3.4,
  the common case carries zero of these fields on the
  wire. If the field count ever proves painful in
  practice, the cleanest fallback is an object-valued form
  (`session.transformers: {audio: [...], ...}`),
  collapsing twelve flat fields into two structured ones
  with the per-type partition preserved as object keys.
- **Language signals live in SESSION-1.** Language signals
  (`stt_lang`, `request_lang`, `detected_lang`, alongside
  `lang`, `secondary_langs`, `output_lang`) are
  session-scoped fields with normative meanings but a
  non-binding consolidation order — the right priority is
  stage-dependent. TRANSFORM-1 §7.1 names which
  transformer types are natural producers of which
  signals; consolidation is the consumer's decision per
  SESSION-1 §3.2.7.

---

## 5. Where the specs differ from current OVOS code

These specifications are *prescriptive*. Some of what they
prescribe matches what runs in OVOS today verbatim; some is a
deliberate cleanup the implementations are expected to grow
into. This section catalogues every known divergence so
implementers know what to migrate and reviewers know what to
expect.

### 5.1 Already aligned

Formalizations of behaviour that exists in current OVOS code
and needs no implementation change:

- The Message envelope (`type` / `data` / `context`) — matches
  `ovos-bus-client.Message`.
- `source`, `destination` semantics including the
  `Message.reply` swap — matches `ovos-bus-client/message.py`.
- `context.session` as a serialized Session object — matches
  `ovos-bus-client/client/client.py`'s
  `message.context["session"] = sess.serialize()`.
- `session.session_id == "default"` for device-local origin —
  matches `ovos-audio/utils.py`'s `require_default_session`
  decorator.
- `session.lang` as the user's preferred language — matches
  the Session class's `lang` attribute.
- `forward` / `reply` / `response` derivation semantics —
  matches `ovos-bus-client.Message.{forward,reply,response}`.
- The `.response` suffix convention — pervasive across OVOS
  topics today.
- The `complete_intent_failure` no-match topic (PIPELINE-1) —
  matches current topic name verbatim.
- `ovos.utterance.cancelled` and `ovos.utterance.handled`
  (PIPELINE-1) — match current topic names verbatim.
- Per-utterance first-match-wins iteration (PIPELINE-1) —
  matches `ovos-core/intent_services/service.py`'s
  `handle_utterance` / `get_pipeline`.
- Per-session pipeline configuration (PIPELINE-1) — matches
  `Session.pipeline`.
- The `<skill_id>:<intent_name>` dispatch topic shape
  (PIPELINE-1) — matches current OVOS practice; skills
  already subscribe to these topics.

### 5.2 Prescriptive renames

| Spec | Current | Prescribed | Notes |
|------|---------|------------|-------|
| INTENT-3 v1.1 | "host" | "orchestrator" | Editorial; conformance unchanged. |
| PIPELINE-1 | `mycroft.skill.handler.start` / `.complete` / `.error` | `ovos.intent.handler.start` / `.complete` / `.error` | Renamed into the `ovos.intent.*` namespace for uniformity. Breaks every existing handler-lifecycle observer; the migration cost is real. |
| PIPELINE-1 | `recognizer_loop:utterance` | `ovos.utterance.handle` | See §5.4 entry. Migration touches `ovos-dinkum-listener`, `ovos-simple-listener`, `ovos-audio`, and `ovos-core/intent_services/service.py`. |

### 5.3 Prescriptive shape changes

- **Keyword intent registration is atomic** (INTENT-4 §5).
  Today a keyword intent is built up via multiple
  `register_vocab` messages followed by a `register_intent`
  with an Adapt `IntentBuilder.__dict__` payload. INTENT-4
  collapses this into a single message with structured
  `{required, optional, one_of, excluded}` arrays of
  vocabulary descriptors. Every skill's keyword-intent path
  needs to be rewritten in the workshop layer.
- **Template intent registration uses structured identity**
  (INTENT-4 §6). Today `padatious:register_intent` carries
  `{name, samples, file_name, lang, blacklisted_words}`; the
  prescribed shape uses the structured `(skill_id,
  intent_name, lang)` triple plus `samples|file` and
  `blacklist|blacklist_file`.
- **Dispatch payload uses unified `owner_id`** (PIPELINE-1
  §7.0, §7.1). Today dispatch carries `skill_id` only.
  PIPELINE-1 §7.0 collapses handler-owner shapes to two:
  plain skill (handler reached via its `skill_id`) and
  pipeline plugin with bundled handlers (the plugin's
  `pipeline_id == skill_id` — one identifier filling both
  roles). Conceptually `skill_id` is the voice-app identity
  (every handler-owner has one); `pipeline_id` is the
  matching-engine identity (only loaded plugins have one).
  Plugins-with-handlers MUST NOT register their intents
  under INTENT-4 — they own the handler directly — and
  SHOULD publish their intent_names via the per-pipeline
  passive index (§7.0, §10) for observability. A
  pure-matcher plugin (Padatious, Adapt, the converse
  plugin) has only a `pipeline_id` and produces matches
  whose `owner_id` is some other component's identity. The
  dispatch payload uniformly carries `owner_id` regardless
  of shape.
- **Handler-lifecycle payload includes `owner_id`**
  (PIPELINE-1 §8.2). Today the trio payload is
  `{name: <handler_func_name>}`. Prescribed: `{owner_id,
  intent_name, optional exception}`.

### 5.4 Architectural divergences

- **The orchestrator maintains a passive registration index**
  (INTENT-4 §10). Today there is no central index — each
  plugin knows what it consumed; nothing aggregates that
  view. INTENT-4 prescribes the orchestrator subscribe to
  all registration topics in parallel with plugins and serve
  `ovos.intent.list` / `ovos.intent.describe` from the
  passive view. This is a new orchestrator responsibility,
  not a change to existing behaviour.
- **The match contract is the single obligation** (PIPELINE-1
  §4.2). The plugin's `match` operation has one MUST: return
  a `Match` (§4.1) or `null`. Bus emissions during `match`
  are allowed — a plugin that polls other components, calls
  out to a model server, or runs any matching strategy that
  requires bus communication is conformant. This matches the
  actual OVOS converse-plugin pattern (it polls active skills
  during its match decision) and accommodates LLM-backed and
  agent-backed plugin shapes that are inherently bus-active.
  Session mutation during `match` is via the explicit
  `Match.updated_session` channel — see the next entry — so
  declined plugins' exploratory mutations never reach the
  next iteration step.
- **`Match.updated_session` as the match-phase session channel**
  (PIPELINE-1 §4.1, §4.2). Promotes the existing ovos-core
  code pattern
  `sess = match.updated_session or SessionManager.get(message)`
  to a normative Match field. The plugin that produces a
  claiming match composes any session mutations it needs
  (decrementing a response-mode counter, pre-promoting an
  active-handler to the head, setting intent_context
  alongside the match) into a fresh snapshot returned in
  `Match.updated_session`. The orchestrator uses that
  snapshot for the dispatch and every downstream stage; a
  declined-match (plugin returns `null`) drops the snapshot
  at the plugin boundary. This is what makes match-phase
  mutation safe under §6.2 first-match-wins iteration.
- **`ovos.utterance.handled` on every terminal path**
  (PIPELINE-1 §9.5). Current `ovos-workshop`'s
  `_on_event_error` does not emit it on the handler-error
  path (`ovos.py:1478-1497`). PIPELINE-1 §8 places trio
  emission on the orchestrator-wrapper around the handler,
  not on the handler itself — workshop is the wrapper in
  current OVOS, and the spec contract requires the wrapper
  to emit `ovos.utterance.handled` unconditionally.
- **Handler-trio is orchestrator-owned** (PIPELINE-1 §8).
  The orchestrator that invokes the handler wraps the call
  and emits `ovos.intent.handler.start` / `.complete` /
  `.error` around it. Third-party handler code carries **no
  normative obligation** to participate in trio emission.
  Skill authors are not protocol authors; the wrapper
  observes start / return / exception around an opaque
  callable.
- **Per-pipeline_id intent introspection** (PIPELINE-1 §10).
  Pull-query / scatter-response surface keyed on
  `pipeline_id`, giving consumers visibility into *which
  intents a particular pipeline plugin's matcher has
  compiled*, distinct from the orchestrator's manifest of
  declared intents (INTENT-4 §10). No current OVOS analogue.
- **CONTEXT-1 scope and ownership encoded in the key shape**
  (CONTEXT-1 §2, §3). A bare key `Person` is shared; a
  prefixed key `music.skill:Person` is private to
  `music.skill`. The `:` is load-bearing — mirroring the
  `<skill_id>:<intent_name>` dispatch topic. Drops separate
  `scope` and `origin` fields on stored entries (both were
  redundant with the key shape). `requires_context` and
  `excludes_context` declarations take an OPTIONAL
  `scope: private|shared` discriminator (default `private`)
  to express which lookup the gate uses; bare-string
  declarations default to private to prevent shared-leak.
- **Skill self-identification on every emission** (INTENT-4
  §3.1). Current OVOS skills set `context.skill_id` on some
  emissions but not uniformly. Enforcement is structural on
  the dispatch path: the orchestrator stamps
  `context.skill_id` from the `<skill_id>:<intent_name>`
  dispatch topic prefix, and skill emissions via
  `forward`/`reply` inherit automatically. Loader-side
  interception covers off-dispatch emissions.
- **Entry-point topic renamed `ovos.utterance.handle`**
  (PIPELINE-1 §9.1). Current deployments use the
  legacy `recognizer_loop:utterance` topic name. That name fails
  the naming conventions of OVOS-MSG-1 §2.1.2 on three
  counts: it uses `:` as a segment separator (where `:` is
  reserved for `<owner_id>:<intent_name>` dispatch topics);
  its leading segment names an implementation role (the
  audio-input "recognizer loop") rather than a stable
  assistant root; and it does not pair with the past-tense
  terminal event `ovos.utterance.handled`. The rename fixes
  all three: dot-separated hierarchy, stable `ovos.` root,
  request/terminal pair (`handle` ↔ `handled`) sharing a
  root verb. Migration cost is real — every audio-input
  service emits this, every intent-service handler
  subscribes — touching `ovos-dinkum-listener`,
  `ovos-simple-listener`, `ovos-audio`, and
  `ovos-core/intent_services/service.py`. A transitional
  deployment MAY subscribe to both names during migration.

### 5.5 New topics with no direct precedent

- **`ovos.intent.matched`** (PIPELINE-1 §9.2). The
  positive-match broadcast notification. Current OVOS has
  `complete_intent_failure` for the negative case but no
  positive equivalent.
- **`ovos.intent.list` / `ovos.intent.describe`** (INTENT-4
  §10). Introspection topics served from the orchestrator's
  passive registration index.
- **`ovos.context.set` / `.unset` / `.clear` / `.list`**
  (CONTEXT-1 §5). Skill-facing API replacing Adapt-specific
  `add_context` / `remove_context` plus
  `mycroft.skill.set_cross_context`.
- **`ovos.transformer.{type}.list`** (TRANSFORM-1 §6).
  Per-type introspection of loaded transformers.
- **Materialize-default-session rule** on `forward` /
  `reply` / `response` (MSG-1 §4.3). Formalizes a "MAY"
  convenience for in-process subsystems; not currently
  implemented but compatible with current behaviour.

### 5.6 Things the specs do *not* change

- The session object's internal shape is owned by
  OVOS-SESSION-1; the field set is the closed set defined
  there plus whatever future specs claim via SESSION-1 §2.1.
  The "extra" fields current OVOS Session carries
  (`persona_id`, `system_unit`, `time_format`, `date_format`,
  …) ride through as non-normative pass-through and may be
  claimed by future per-domain specs.
- The `mycroft.*` topic prefix outside the intent layer (e.g.
  `mycroft.audio.*`) — these are not part of any spec here.
- The `<skill_id>:<intent_name>` dispatch topic — kept
  verbatim from current OVOS so no skill needs to migrate
  its handler subscription.
- **Engine-specific introspection topics.** The standard
  plugins expose their own debug / inspection topics — for
  example `intent.service.adapt.reply`,
  `intent.service.adapt.manifest`,
  `intent.service.adapt.vocab.manifest`, and
  `intent.service.padatious.get`. These are plugin-specific
  surface, parallel to the spec's generic
  `ovos.intent.list` / `ovos.intent.describe` (INTENT-4
  §10). The specs do not claim authority over them — they
  remain plugin-defined and may continue to coexist with
  the orchestrator's generic index.

### 5.7 Predecessor-topic mapping

The bus topics formalized by INTENT-4 and PIPELINE-1 replace
a number of legacy names. Implementer migration aid:

#### Registration topics (INTENT-4)

| Legacy topic | v1 replacement | Notes |
|--------------|---------------|-------|
| `register_vocab` | folded into `ovos.intent.register.keyword` | Vocabularies in v1 are inline `samples` or `file`-by-path inside the registration. |
| `register_intent` (Adapt parser) | `ovos.intent.register.keyword` | Adapt's `IntentBuilder.__dict__` payload replaced by the structured shape. |
| `padatious:register_intent` | `ovos.intent.register.template` | Same content, structured payload. |
| `padatious:register_entity` | `ovos.entity.register` | Entities are not Padatious-specific. |
| `detach_intent` | `ovos.intent.deregister` | Identity now expressed as the structured triple, not the munged `skill_id:intent_name` string. |
| `detach_skill` | `ovos.skill.deregister` | |
| `mycroft.skill.enable_intent` / `mycroft.skill.disable_intent` | `ovos.intent.enable` / `ovos.intent.disable` | First-class topics under v1, with the prefix dropped. |

#### Utterance-lifecycle topics (PIPELINE-1)

| Legacy topic | Status |
|--------------|--------|
| `recognizer_loop:utterance` | renamed to `ovos.utterance.handle` (see §5.4) |
| `complete_intent_failure` | **unchanged** — kept as the no-match signal. |
| `ovos.utterance.cancelled` | **unchanged** — kept as the cancellation signal. |
| `ovos.utterance.handled` | **unchanged** — kept as the universal end-marker. |
| `<skill_id>:<intent_name>` | **unchanged** — kept as the dispatch topic; PIPELINE-1 extends the shape to `<owner_id>:<intent_name>` so plugins can also own handlers. |
| `mycroft.skill.handler.start` / `.complete` / `.error` | renamed to `ovos.intent.handler.start` / `.complete` / `.error` |

#### Out of scope

| Legacy topic | Status |
|--------------|--------|
| `add_context` / `remove_context` | Replaced by `ovos.context.set` / `.unset` under CONTEXT-1. |
| `mycroft.skill.set_cross_context` / `remove_cross_context` | Replaced by `ovos.context.set` / `.unset` with `scope: "shared"` under CONTEXT-1. |
| `<skill_id>.activate` | Activity-tracking emit currently in `ovos-core`; not part of any spec here. |

---

## 6. Implementer reference

Material an implementer reaches for repeatedly: cross-spec
tables that don't fit cleanly in any single normative spec.

### 6.1 Topic-name conventions across the family

The naming conventions of OVOS-MSG-1 v2 §2.1.2 — dot-separated
hierarchy, stable root, verb-tense pattern for the trailing
segment, request/terminal pairs sharing a root verb,
`.response` suffix, per-instance
`<root>.<domain>.<id>.<verb>` form — apply across the family.
The four-way collision of the word "intent" in introspection
topics deserves an explicit callout:

- `ovos.intent.list` (INTENT-4 §10) — list of registered
  *intents* (skills declare them; `data` entries name
  `intent_name`).
- `ovos.pipeline.<pipeline_id>.intents.list` (PIPELINE-1
  §10) — list of *intents currently compiled by one plugin's
  matcher* (`data` entries name `intent_name`).
- `ovos.transformer.intent.list` (TRANSFORM-1 §6) — list of
  *intent-transformer plugins* loaded at the intent-transformer
  injection point (`data` entries name `transformer_id`).
  Despite the topic shape, this is **not** an intent-listing
  surface; it follows the per-chain pattern
  `ovos.transformer.<type>.list` where `<type>` happens to
  be `intent` for this chain (alongside `audio`, `utterance`,
  `metadata`, `dialog`, `tts`).

The collision is at the human-reading level only; payload
shapes are distinct and a consumer subscribing to one cannot
accidentally parse responses from another.

### 6.2 Session-field cheat-sheet

Every spec in the family that claims a `session` field does
so via the OVOS-SESSION-1 §2.1 registry mechanism. The full
set spans four specs; this table consolidates them. All
fields follow the canonical SHOULD-omit /
`[]`-equivalent-to-omission wire-weight rule of
OVOS-SESSION-1 §3.4.

| Field | Owner | Role | Empty-array semantics |
|-------|-------|------|------------------------|
| `session_id` | SESSION-1 §3.1 | identity / channel | n/a (string; `"default"` reserved) |
| `lang` | SESSION-1 §3.2.1 | preference (user) | n/a (string) |
| `secondary_langs` | SESSION-1 §3.2.2 | preference (user) | ≡ absent |
| `output_lang` | SESSION-1 §3.2.3 | preference (renderer) | n/a (string) |
| `stt_lang` | SESSION-1 §3.2.4 | signal (per-utterance) | n/a (string) |
| `request_lang` | SESSION-1 §3.2.5 | signal (emitter hint) | n/a (string) |
| `detected_lang` | SESSION-1 §3.2.6 | signal (lang-detect) | n/a (string) |
| `site_id` | SESSION-1 §3.3 | opaque group identifier | n/a (string) |
| `pipeline` | PIPELINE-1 §5.1 | preference (ordering) | ≡ absent |
| `blacklisted_pipelines` | PIPELINE-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_skills` | PIPELINE-1 §5.3 | policy (denylist) | ≡ absent |
| `blacklisted_intents` | PIPELINE-1 §5.4 | policy (denylist) | ≡ absent |
| `audio_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `utterance_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `metadata_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `intent_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `dialog_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `tts_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `blacklisted_audio_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_utterance_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_metadata_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_intent_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_dialog_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_tts_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `intent_context` | CONTEXT-1 §2 | per-session state | object; absent ≡ empty |

**Role glossary:**

- *Preference* — populated by the session origin to request
  specific behaviour. Orchestrator narrows the request by
  availability and policy.
- *Policy* — populated by deployment / layer-2 substrate to
  enforce constraints. Overrides preference at the
  composition stage (PIPELINE-1 §5.5, TRANSFORM-1 §5.3).
- *Signal* — recorded by a producer or earlier lifecycle
  stage to communicate information about this specific
  utterance.
- *Identity / channel* — names the session itself; not a
  preference or policy knob.

### 6.3 Component-identity stamp-rule cheat-sheet

Each component type self-identifies via a reserved context
key. The keys coexist freely on a single Message when the
derivation chain crosses component boundaries; attribution
consumers apply the eight-level lifecycle-position precedence
of CONTEXT-1 §5.2 to pick a single owner when needed.

| Context key | Owner | Stamps on (origination + modify-in-place) | `.reply` / `.response` | `.forward` |
|-------------|-------|------|----------|--------|
| `skill_id` | INTENT-4 §3.1 | yes | yes (authorial — overwrite) | no (preserve inherited) |
| `pipeline_id` | PIPELINE-1 §3.1 | yes | yes (authorial — overwrite) | no (preserve inherited) |
| six `<type>_transformer_ids` (list-valued) | TRANSFORM-1 §1.3 | yes (append) | yes (append) | no (list rides through) |

The `<type>_transformer_ids` list-valued form preserves the
full per-type chain provenance on the wire (every transformer
of that type that touched the Message, in order of touch).
Single-string `skill_id` / `pipeline_id` reflect that those
component types *originate* Messages rather than chain over
them.

### 6.4 Introspection patterns

Four specs in this set define pull-query / scatter-response
introspection surfaces. The shapes are intentionally similar
but serve different scopes:

| Spec | Topic | Scope | Authoritative responder |
|------|-------|-------|-------------------------|
| INTENT-4 §10 | `ovos.intent.list` / `.describe` | Declared intents observed on the bus | Orchestrator (the manifest) |
| PIPELINE-1 §10 | `ovos.pipeline.<pipeline_id>.intents.list` | Intents currently compiled inside a specific plugin's matcher | The pipeline plugin |
| CONTEXT-1 §5.4 | `ovos.context.list` | Post-decay session-context snapshot | The orchestrator process owning the match round |
| TRANSFORM-1 §6 | `ovos.transformer.<type>.list` | Loaded transformers per injection point | The orchestrator process implementing that chain |

Three properties hold across all four:

1. **Pull-query is the source of truth.** Producers MAY
   broadcast load-time announcements; consumers MUST NOT
   rely on having received them. The bus is asynchronous
   and gives no delivery guarantee; a consumer that started
   late missed the broadcast.
2. **No completeness signal.** A consumer that wants
   completeness keeps its own roster of expected responders
   and times out non-responders.
3. **Per-process slices under split orchestrators.** When
   the orchestrator is split (PIPELINE-1 §2), each process
   responds from its own slice; consumers aggregate.

All four surfaces share the `ovos.<domain>.` prefix; verb
segments vary by domain (some nest, some don't). The
uniformity is in the namespace, not in a fixed depth.

---

## 7. Known gaps and planned work

- **Per-plugin behavioural specs.** OVOS-PIPELINE-1 defines
  the plugin contract (the `match` shape, the orchestrator's
  iteration semantics) but explicitly defers what each
  non-trivial plugin type actually *does*. Real candidates
  for their own specifications: `converse`, `fallback`,
  `common_query`, `ocp`, `persona`, `stop`. Each defines its
  own internal behaviour and its own bus emissions beyond
  the universal lifecycle PIPELINE-1 prescribes.
- **Session preference fields not yet claimed.** SESSION-1
  defines the wire shape and OVOS-SESSION-2 (in flight at
  PR #27) defines the lifecycle and state-ownership model;
  what remains deferred is the full set of session
  preferences current OVOS already carries (`persona_id`,
  `time_format`, `date_format`, `system_unit`,
  `tts_preferences`, `location`, …) — these need to be
  claimed under SESSION-1 §2.1's field registry by their
  respective owning specs (a future preferences spec,
  OCP / persona / locale specs as appropriate).
- **Text normalization of ASR output.** The basis for slot
  value typing (INTENT-1 §5.3). Deferred to its own
  specification.
- **A machine-checkable conformance corpus** of `template →
  sample set` pairs for INTENT-1 expansion, so expander
  conformance can be verified automatically. A parallel
  corpus of bus-message fixtures for MSG-1 would be the
  equivalent at the bus layer.
- **An end-to-end worked example.** The specs have local
  examples; none shows a single skill defining one keyword
  intent and one template intent through the whole path —
  files, registration, match, handler.
- **Conversation-level evaluation infrastructure.** Rasa
  has story-based testing and end-to-end success metrics;
  the OVOS specs do not currently have a counterpart.
- **OVOS-INTENT-2 ↔ hassil `intents` translation tool.**
  The grammar lineage (§2.1) makes a mechanical translator
  between OVOS-INTENT-2 locale resources and HA's `intents`
  YAML feasible. Such a tool would let the two corpora
  cross-pollinate without either format changing. Sits at
  injection point 3 of §3.3 conceptually but is
  build-time rather than runtime tooling.
- **i18n corpus.** OVOS-INTENT-2 defines the locale file
  format, and `ovos-localize` (§1.4) provides the
  operations layer; what remains is the *scale* of the
  translated corpus.
