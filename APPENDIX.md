# Appendix — Design Notes and Context

**Non-normative.** This document is a companion to the OVOS formal
specifications. It records design rationale, comparisons with other
systems, the catalogue of *deliberate* divergences from current OVOS
code, and topics worth discussing that do not belong in a normative
specification. Nothing here is binding — the normative documents are
OVOS-INTENT-1, OVOS-INTENT-2, OVOS-INTENT-3, OVOS-INTENT-4,
OVOS-MSG-1, and OVOS-PIPELINE-1. This appendix exists so the specs
themselves can stay terse and requirement-focused.

Pointers to specific OVOS code (file paths, class names, function
names) are deliberately kept *out* of the spec bodies and collected
here where appropriate, because implementation code moves and
specifications must not.

---

## 1. These specifications formalize an existing system

The OVOS stack — the engines (padatious, Adapt), the skill ecosystem,
the resource file formats, the pipeline, the bus, the session model —
already exists and runs in production. These specifications were
written **after** the system they describe. They are a *formalization
pass*: they document an existing design implementation-agnostically,
tighten under-defined corners, and remove accidental inconsistencies,
so the contracts can be implemented by new engines, new hosts, and
adopted by other assistants.

This matters for how to read them. They are **prescriptive** — each
spec states a clean target, and where it diverges from current OVOS
behaviour the divergence is a deliberate cleanup (catalogued in §6) —
but they are not speculative. The target is a lightly-cleaned version
of a working system, not a greenfield design. `padacioso`,
`ovos-workshop`, and `ovos-bus-client` are the closest existing
implementations; none yet fully conforms, and bringing them into
conformance is planned work. OVOS-MSG-1 is the closest to current code
of all the specs — it is largely a verbatim formalization of what
`ovos-bus-client` already does.

---

## 2. Comparison with Home Assistant and Rhasspy

OVOS, Home Assistant (HA), and Rhasspy share a common lineage. The
bracket-expansion grammar of OVOS-INTENT-1 — `(a|b)` alternatives, `[optional]`
segments, `{slot}` placeholders — is the same family as HA's `hassil` sentence
templates and Rhasspy's `sentences.ini`. The *syntax* is not novel. What is
distinctive about the OVOS approach is everything around the grammar.

### 2.1 What the OVOS design does differently

- **An implementation-agnostic spec at all.** HA and Rhasspy have no
  format-level specification independent of their implementation — the code is
  the contract. OVOS now has one, which is what lets multiple engines (and
  other assistants) implement the same contract.
- **Engine-agnostic matching.** OVOS-INTENT-1 §4 treats templates as *training
  data* and leaves matching, scoring, and generalization to the engine. HA's
  core matching is `hassil`, a deterministic template matcher; Rhasspy compiles
  templates into a closed ASR grammar. The OVOS contract accommodates a
  deterministic matcher, a neural classifier, or an LLM behind one interface.
- **Templates are training data, not a closed grammar.** A capable OVOS engine
  generalizes beyond the authored samples. Rhasspy's closed-grammar model is
  deterministic and offline-guaranteed but brittle — an utterance not derivable
  from `sentences.ini` cannot be recognized at all.
- **A multi-stage pipeline** (see §3). Intent engines are two stage kinds among
  many. Neither HA nor Rhasspy exposes an intent layer this structured.
- **An intent is bound to one handler, owned by one skill** (OVOS-INTENT-3 §1).
  See §2.3 — this follows necessarily from the open skill ecosystem.
- **A bus substrate that is openable to layer-2 systems** (OVOS-MSG-1
  §3.4, §4.4). The `source`/`destination` boundary pair plus
  `session.session_id` give third parties everything they need to
  layer authentication, routing, and remote participation on top of
  OVOS without modifying it. HiveMind is the canonical example.
  Neither HA nor Rhasspy exposes their bus this openly. See §5.

### 2.2 What Home Assistant and Rhasspy do better

- **Reusable template fragments.** `hassil` has `expansion_rules` and Rhasspy
  has `<rule>` references — named, reusable sub-templates that let authors share
  common fragments (politeness prefixes, articles, recurring phrasings). The
  version-1 OVOS grammar had no equivalent. **OVOS-INTENT-1 version 2 closes
  this** with the `<name>` inline vocabulary reference (issue #1), which expands
  a named `.voc` in place — reusing the existing slot-free format rather than
  adding a new construct (see §4).
- **i18n corpus maturity.** HA's community `intents` repository is a large,
  managed, professionally-translated corpus covering many languages. OVOS has
  the tooling counterpart in **ovos-localize** (§8) — a GitHub-native
  localization platform built around the OVOS-INTENT-2 resource roles — so the
  gap here is the *scale and maturity* of the corpus, not the absence of
  tooling.
- **Concrete, testable completeness.** HA and Rhasspy ship systems where the
  hard parts — matching, number and range handling, slot typing — are solved
  concretely. The OVOS specs deliberately defer some of these (slot typing to a
  future normalization spec; matching to the engine). That deferral is
  intellectually consistent but means the specs' value depends on the engines
  and tooling that fill the gaps.

### 2.3 Closed domain vs open ecosystem

The sharpest difference is not technical but structural. **Home Assistant is a
curated, closed domain**: home automation, with a vendor-managed intent
vocabulary. HA can treat an intent such as `HassTurnOn` as a *shared contract*
honoured uniformly across hundreds of integrations and many languages, because
HA controls and curates that vocabulary.

**OVOS is an open ecosystem.** Skills are arbitrary third-party Python packages,
installed by pip, developed independently, running as arbitrary code in
process. A skill can do anything; OVOS voice-enables anything. In that setting a
shared global intent vocabulary is not a missing feature — it is incoherent.
When skills are unbounded, an intent *must* be private to the skill that
defines it and bound directly to that skill's handler. OVOS-INTENT-3's "an
intent is not an event" stance is therefore the correct model for an open
ecosystem, just as HA's shared-vocabulary model is correct for a curated one.
The two models are right for different platforms; neither is universally
better.

### 2.4 Summary

OVOS is not out-designed by HA or Rhasspy at the architecture level — at the
pipeline layer (§3) it is ahead of both, and its intent-as-handler-binding
model is the correct consequence of being an open platform. HA's real advantage
is the maturity and scale of its translation corpus — an ecosystem investment,
not an architectural one, and one OVOS now has tooling for in ovos-localize
(§8). The grammar itself is a commodity shared by all three; the OVOS bet is the
engine-agnostic contract and the pipeline.

---

## 3. The pipeline-plugin model

The piece that sits *around* the intent and bus stacks — the
multi-stage orchestrator that decides which engine even gets a
turn, runs `converse` / `fallback` / `common_query` / `ocp` /
`persona` stages, and produces the universal
`ovos.utterance.handled` end-marker — is what makes OVOS
structurally distinctive (Home Assistant and Rhasspy have no
equivalent layer).

The plugin abstraction is **already in current code**:
`OVOSPipelineFactory` loads pipeline plugins by id at startup,
the orchestrator holds them in a `pipeline_plugins` dict keyed
on `pipeline_id`, and the default `Session.pipeline` is an
ordered list of plugin identifiers (with a migration map
translating legacy `padatious_high`-style names into
modern `ovos-padatious-pipeline-plugin-high`-style ones). The
official `ovos-padatious-pipeline-plugin`,
`ovos-adapt-pipeline-plugin`, `ovos-converse-pipeline-plugin`,
`ovos-fallback-pipeline-plugin`, `ovos-common-query-pipeline-plugin`,
`ovos-ocp-pipeline-plugin`, and the persona plugins all
already conform to this model.

OVOS-PIPELINE-1's contribution is therefore a **prescriptive
refinement**, not a wholesale new abstraction. It:

- formalizes the plugin contract (the `match` shape, the `Match`
  result, the side-effect-free discipline);
- defines `<owner_id>:<intent_name>` **dispatch polymorphism** so
  a plugin can bundle its own handler (a language-model persona,
  a chatbot) as a first-class participant alongside skill-owned
  handlers;
- prescribes the **universal `ovos.utterance.handled` end-marker**
  on every terminal path;
- renames the `mycroft.skill.handler.*` trio → `ovos.intent.handler.*`.

The current high/medium/low confidence-tier convention is
**compatible** with PIPELINE-1 and out of scope for the spec.
From the bus's perspective each tier is already a distinct
`pipeline_id` in the session's pipeline list (e.g.
`padatious_high`, `padatious_medium`, `padatious_low`), which is
exactly what the spec prescribes. How a Python plugin class
internally serves multiple `pipeline_id`s — for example one class
with `match_high` / `match_medium` / `match_low` methods, an
orchestrator-side suffix-decoding helper, three separate plugin
instances, etc. — is implementation choice this spec does not
constrain.

Three properties make the resulting model unusually expressive:

- **All plugins are equivalent.** No spec-level distinction
  between intent engines, converse handlers, fallbacks,
  language-model personas, classic chatbots, anything else.
  They all expose the same `match` contract. A deployment loads
  whichever plugins its skills need.
- **Skills and plugin-bundled handlers are indistinguishable as
  handler owners.** From outside, the assistant responded — the
  user does not know or care whether a skill matched against a
  registered intent or a language-model plugin generated the
  response on the fly.
- **The engine-agnostic intent contract is already realized**,
  not hypothetical. OVOS persona plugins (`ovos-persona`,
  `ovos-persona-server`, `ovos-claude-plugin`,
  `ovos-openai-plugin`, etc.) plug into the pipeline as
  first-class language-model stages. The ordered chain
  (deterministic keyword engines before fuzzy template engines
  before language-model fallbacks last) is also how the system
  *bounds* generalization in practice.

What OVOS-PIPELINE-1 deliberately leaves out: **per-plugin
behavioural contracts**. A `converse` plugin, a `fallback`
plugin, a persona plugin: each defines itself. PIPELINE-1 only
defines the contract every plugin conforms to and the universal
utterance lifecycle around the iteration.

---

## 4. Design rationale

Short notes on *why* the specifications make the choices they do — the
reasoning, not the requirement.

### Intent grammar and resources (INTENT-1, -2, -3)

- **ASR-normalized input, no escaping** (OVOS-INTENT-1 §2). The grammar targets
  voice input. By contract, text reaching an engine is already lowercased,
  punctuation-stripped, single-spaced. Bracket metacharacters therefore cannot
  occur as literal input, so no escape mechanism is needed. This is a
  simplification *bought* by scoping the grammar to voice.
- **Templates are training data** (OVOS-INTENT-1 §4). Enumerating every
  phrasing is futile for natural speech. A template describes the *shape* of
  the training data; the engine generalizes. This is why expansion is defined
  precisely but matching is not.
- **An intent is not an event** (OVOS-INTENT-3 §1). See §2.3 — necessary for an
  open skill ecosystem.
- **Two non-interoperable methods** (OVOS-INTENT-3 §2). Keyword and template
  intents describe a command in fundamentally different shapes. Rather than
  forcing one model, the spec keeps both and makes engines declare which they
  accept. The cost is that a developer must choose per intent and know which
  engines an installation runs.
- **Slot typing is deferred** (OVOS-INTENT-1 §5.3). Interpreting a slot value
  as a number or date is inseparable from how ASR output is normalized — and
  normalization is not yet specified. Specifying typing first would be
  incoherent, so a value is, for now, an opaque sequence of words.
- **`.blacklist` vs `excluded`** (OVOS-INTENT-3 §4.2, §5.4). The template
  grammar is purely generative — it cannot express "not this". Template intents
  therefore need a separate `.blacklist` artifact for suppression. Keyword
  intents express the same idea natively with the `excluded` constraint role.
  The asymmetry follows from the grammar, not from inconsistency.
- **No regular expressions** (OVOS-INTENT-3 §4.4). Free-form structured text is
  a slot — use a template intent and the slot extractor. Regexes are also
  notoriously hard to localize, which conflicts with the per-language model.
- **Inline vocabulary references reuse `.voc`** (OVOS-INTENT-1 §3.7). A
  reusable template fragment and a keyword vocabulary are the same thing — a
  named, slot-free phrase set — so `<name>` resolves to a `.voc` rather than
  introducing a new file role. The change is one grammar token plus an
  expander step.

### Bus, session, and routing (MSG-1)

- **One spec, not two.** Envelope + routing + session + derivations
  are tightly coupled — every routing key lives in `context`, every
  derivation manipulates routing or session, and all of them
  formalize *existing* OVOS code. Splitting them was tried; the split
  did not survive the derivations (which can only meaningfully be
  defined where the routing keys are), so they were merged into a
  single bus-message spec.
- **`context` is extensible by design.** Only the keys other systems
  already key behaviour off (`source`, `destination`, `session`) are
  given normative meaning. Everything else — GUI routing, tracing,
  security — is layered by other specs without touching the
  envelope.
- **`source`/`destination` are informational, not authorization**
  (MSG-1 §3.3). The bus is not a security boundary. Layer-2 systems
  (HiveMind) build authentication and routing enforcement on top of
  the pair without OVOS itself learning about peers.
- **The boundary is user ↔ assistant, not core ↔ handler.** The
  `(source, destination)` pair marks who is currently talking to whom
  across one boundary only: the external participant (user, chat UI,
  satellite client, test harness) on one side, the assistant — OVOS
  core *and* every skill handler — on the other. Skills are not on the
  other side of this boundary from OVOS core; from the user's
  perspective the assistant is one thing. The flip happens **once**
  per conversational turn (§5.1), not on every internal hop.
- **`session_id == "default"` is the only normative-magic value**
  (MSG-1 §4.1). It marks "originated by the device itself" and is the
  hook `ovos-audio` already uses to decide whether to play TTS
  locally. One reserved string, one well-defined consequence — enough
  for layer-2 routing without specifying a full session model.
- **Absent `session` equals `session_id: "default"`** (MSG-1 §4.3).
  Code paths that never set a session shouldn't accidentally get
  treated as untrusted; the rule makes the substrate forgiving for
  in-process subsystems while keeping the policy hook intact.
- **No central correlation, no central state** (MSG-1 §5.4). The bus
  is fully asynchronous. There is no per-message ID, no
  in-reply-to chain, no host-managed request/response index, and no
  spec-level state tracking of any kind. Components that need to
  correlate or remember things do it themselves, keyed on
  `session.session_id` (the interaction-channel identifier — §5.2
  below). Multi-turn conversation, intent context, cross-skill
  state, and similar concerns are deferred to future specifications;
  see §5.2 for the model and §7 for the list of planned work.

### Intent registration broadcast (INTENT-4)

- **Registrations are broadcast — already how OVOS works.** Skills
  emit registration messages on the bus; plugins that care about a
  particular registration kind subscribe to the corresponding
  topic. There has never been a central routing party in OVOS;
  INTENT-4 just gives this existing model normative topic names.
  The legacy bus topics (`padatious:register_intent`,
  `register_vocab`, etc.) are renamed into the `ovos.intent.*`
  namespace — see §6.7 for the mapping. A migration to the
  prescribed topic names is mostly a string replacement.
- **No "no plugin claimed" error.** Following from the
  broadcast model: a registration that no plugin consumes is
  silently dropped. The producer gets no signal — the
  introspection topics (`ovos.intent.list` /
  `ovos.intent.describe`) are the supported way to verify what
  the orchestrator's passive index recorded.
- **The orchestrator passively indexes; it does not gate.** The
  introspection topics serve from a passive registration index
  built by listening to broadcasts (this *is* new — current OVOS
  has no central index). The index reflects what skills
  *declared*, not what plugins actually match against —
  observability-only.

### Pipeline plugins (PIPELINE-1)

- **The plugin model is already in place; PIPELINE-1 refines it**
  (see §3). The current orchestrator already loads plugins by id
  through `OVOSPipelineFactory` and iterates `Session.pipeline`.
  PIPELINE-1 tightens the contract rather than introducing the
  abstraction.
- **Orchestrator and plugin contracts live in one spec**, since
  the orchestrator's job *is* iterating plugins and translating
  their matches into bus events. Splitting them would leave
  neither coherent.
- **Plugin contract is minimal.** `match(utterance, session) →
  Match | None`. Side-effect-free during `match`; everything
  else (state, registrations, language-model calls, response
  generation) is plugin-internal black box. The smaller the
  contract, the wider the set of plugins it accommodates.
- **Tier conventions are out of scope.** The current high /
  medium / low suffix is implementation strategy: from the bus,
  each tier is already a distinct `pipeline_id` in
  `Session.pipeline`. PIPELINE-1 prescribes only that the
  orchestrator iterates opaque `pipeline_id`s; whether a Python
  plugin class internally serves multiple tiers via
  `match_high` / `match_medium` / `match_low` methods, separate
  plugin instances, or anything else is implementation choice the
  spec does not constrain. The current convention is compatible
  with PIPELINE-1 unchanged.
- **Skills and plugins are equivalent handler owners.** Dispatch
  topic `<owner_id>:<intent_name>` polymorphism (owner is
  `skill_id` or `pipeline_id`) lets a plugin bundle its own
  handler — for example, a language-model persona plugin that
  has no skills behind it — and still be addressed uniformly.
  From outside, the assistant responded; the internal owner
  type is invisible.
- **Universal `ovos.utterance.handled` end-marker on every
  terminal path.** One reserved invariant lets observers count
  turns, route fallbacks, and know "the assistant is idle now"
  without per-stage knowledge.
- **`session.pipeline` is per-session.** Different
  sessions can carry different pipeline configurations — for
  example, a remote-peer session may run a restricted pipeline
  that excludes destructive plugins. This composes with the
  layer-2 substrate (§5) without orchestrator-side changes.

### Intent context (CONTEXT-1)

- **Lifts intent context out of Adapt.** The Adapt-era
  `add_context` / `remove_context` mechanism, and the
  Mycroft-era `mycroft.skill.set_cross_context` /
  `mycroft.skill.remove_cross_context` fan-out for cross-skill
  use, are Adapt-only at the matcher level — Padatious and
  other engines ignore them. CONTEXT-1 generalizes the
  mechanism into a session-bound, decaying flat key/value store
  consumed by every intent engine uniformly via
  `requires_context` and `excludes_context` declarations.
- **Two explicit scopes.** `private` (orchestrator
  auto-prefixes with `<skill_id>:`) and `shared` (flat,
  cross-skill). The current OVOS code models the same distinction
  informally (`MycroftSkill.set_context` auto-prefixes with
  `alphanumeric_skill_id`; `set_cross_skill_context` fans out via
  a bus event); CONTEXT-1 names the scopes explicitly and routes
  both through one bus surface (`intent.context.set` / `.unset` /
  `.clear` / `.list`).
- **Prior art for the negative gate.** Three in-tree intent
  engines under `/plugins-pipeline/` —
  [jurebes](https://github.com/OpenJarbas/jurebes),
  [nebulento](https://github.com/OpenJarbas/nebulento), and
  [palavreado](https://github.com/OpenJarbas/palavreado) —
  independently implement `exclude_context` as a first-class
  negative gate. CONTEXT-1's `excludes_context` adopts the same
  primitive at the spec level, addressing patterns ("fire once",
  "modal suppression") that positive gating alone cannot express.
- **Engine-side mutation as a sanctioned non-bus pathway.** The
  Adapt pipeline plugin auto-injects matched entities into context
  *inside* `match()`, which conflicts with PIPELINE-1 §4.2's
  side-effect-free `match` rule. CONTEXT-1 §5.3 carves an explicit
  window between match-accept and dispatch-emit for engine-side
  session mutation, with the orchestrator (not the bus) carrying
  the write. This both legitimizes the established practice and
  resolves the PIPELINE-1 contradiction.

### Transformer plugins (TRANSFORM-1)

- **Spec'd as an architectural pattern, not a feature list.** An
  orchestrator MAY implement chains at any subset of six
  injection points (audio, utterance, metadata, intent, dialog,
  TTS); a null-implementation is conformant. For each chain it
  does implement, the per-type contract binds. Each injection
  point's existence is justified by what the lifecycle holds at
  that exact moment — what's possible there that isn't possible
  elsewhere.
- **Intent transformers as the system-typing home.**
  OVOS-INTENT-1 §5.3 defers slot value typing pending a text
  normalization specification. TRANSFORM-1 §3.4 is the spec'd
  injection home for typing: a deployer ships date / number /
  duration parsing once, and every skill receives typed values
  in `Match.captures` regardless of which engine matched. The
  OVOS analogue of ASK's `AMAZON.DATE` and Dialogflow's
  `@sys.date-time`, but as an injected enrichment rather than a
  built-in engine feature.
- **Concrete in-tree plugins as prior art.** Nine plugins live
  under `/plugins-transformer/` today, covering five of the six
  injection points: utterance transformers
  (`ovos-utterance-normalizer`, `ovos-utterance-corrections-plugin`,
  `ovos-transcription-validator-plugin`,
  `ovos-utterance-plugin-cancel`,
  `ovos-bidirectional-translation-plugin`); dialog transformers
  (`ovos-dialog-normalizer-plugin`,
  `ovos-bidirectional-translation-plugin`,
  `ovos-dialog-transformer-openai-plugin`); audio transformers
  (`ovos-audio-transformer-plugin-speechbrain-langdetect`,
  `ovos-audio-transformer-plugin-ggwave`,
  `ovos-audio-transformer-redis-publish`); intent transformers
  (`ovos-keyword-template-matcher`,
  `ovos-ahocorasick-ner-plugin`). The
  `bidirectional-translation` plugin exercises the cross-chain
  coordination via `Message.context` that TRANSFORM-1 §7
  formalizes.
- **Ascending priority.** TRANSFORM-1 §4 specifies ascending
  priority (lower = earlier, default 50). Current OVOS sorts
  transformer chains **descending**
  (`ovos_core/transformers.py:53,117,205`, `reverse=True`); the
  spec aligns with the **ascending** convention already used by
  fallback skills (`fallback_service.py:49`, default 101 = run
  last) and the natural "stages count up" reading. Bringing
  current plugins into conformance only requires flipping
  relative priorities, not rewriting.
- **Cancellation aligned with prior plugin convention.** Two
  existing utterance transformers
  (`ovos-utterance-plugin-cancel`,
  `ovos-transcription-validator-plugin`) already signal the
  lifecycle should abort by returning empty utterance lists with
  `{canceled: true, cancel_word: <reason>}` context keys.
  TRANSFORM-1 §8 keeps the convention, renaming `cancel_word` to
  `cancel_reason` (the structured concept the field encodes) and
  adding orchestrator-stamped `cancel_by: <transformer_id>`. The
  spec's `ovos.utterance.cancelled` terminal event sits alongside
  the existing `complete_intent_failure` from PIPELINE-1, keeping
  cancellation and failure observably distinct on the bus.
- **Language disambiguation.** TRANSFORM-1 §7.1 spec'd a
  precedence hierarchy for resolving the in-flight utterance's
  operative language: `stt_lang` (STT-attested) >
  `request_lang` (source-channel-volunteered) > `detected_lang`
  (transformer-derived) > `data.lang` (Message producer) >
  existing `session.lang` > config default, gated by
  `valid_langs`. Mirrors what current OVOS does informally in
  `ovos_core/intent_services/service.py:197-222`
  (`disambiguate_lang`); the spec elevates it to a normative
  hierarchy with reserved context keys, and explicitly deprecates
  the legacy top-level `Message.context["lang"]` shortcut.

---

## 5. The OVOS bus as a substrate

Under MSG-1's `source` / `destination` / `session` model, the bus is
not just an internal transport — it is the **substrate higher-level
systems plug into without modifying OVOS**. Two mechanics make that
work: **single-flip routing** (§5.1), which keeps the routing pair
correct end-to-end without per-component effort; and **no central
state or correlation** (§5.2), which makes layer-2 systems
composable. HiveMind is the canonical example of what both
together enable (§5.3).

### 5.1 The single-flip routing model

The most important bus invariant in OVOS, and the one most often
reinvented incorrectly. The routing pair (`source`, `destination`)
flips **exactly once per conversational turn**, performed by
ovos-core, before the intent dispatch is emitted. From that point
on, every handler-side emission is *already* addressed back at the
user.

Three steps:

1. **The user side emits.** An external component — microphone
   service, chat UI, satellite client, test harness — emits an
   utterance with `source` set to itself:

       context: { source: "audio", destination: null, session: {...} }

2. **ovos-core flips, then dispatches.** When the intent service
   matches an intent it derives the dispatch via
   `Message.reply(match_type, data)` (`ovos-core/.../service.py:340`).
   The `.reply` rule of MSG-1 §5.2 swaps the routing pair:

       context: { source: "ovos-core", destination: "audio", session: {...} }

   The dispatch goes out on the per-intent topic
   `<skill_id>:<intent_name>`. The flip has already classified the
   message as *going back at the user*, even though a skill handler
   is what actually runs.

3. **The handler `.forward`s.** Every message the skill emits in
   response — `speak`, the handler lifecycle trio, GUI events —
   uses `Message.forward(...)` (`ovos-workshop/.../ovos.py:1461,
   1472, …`). `.forward` preserves `context` unchanged, so every
   handler emission is already addressed back at the original
   user-side component.

Two consequences fall out:

- **The boundary is user ↔ assistant, not core ↔ handler.** Skill
  handlers are on OVOS's side of the boundary; from outside, OVOS
  is one thing. The user doesn't know or care which skill answered
  them.
- **Handler authors never write addressing code.** Because
  `.forward` preserves the flipped pair, no skill anywhere needs
  to understand `source` / `destination`. Get the inversion right
  once in ovos-core, and every downstream skill is automatically
  correct.

What this rules out: no per-hop addressing (handlers don't pick
their own `destination`); no second flip (handlers `.forward`,
they don't `.reply` to the dispatch); the dispatch topic
`<skill_id>:<intent_name>` selects the handler, not `destination`
(the destination belongs to the user). Implementers using `.reply`
where `.forward` is appropriate produce mis-routed messages that
work in local tests but silently break layer-2 routing.

### 5.2 No central correlation, no central state

The bus is **fully asynchronous**. OVOS does not centrally
correlate request/response chains, and does not centrally track
per-conversation state. There is no per-message identifier, no
in-reply-to field, no host-side index mapping a `.response` back to
its request, no shared "current conversation" record.

`session.session_id` identifies an **interaction channel** —
nothing more. Two messages sharing a `session_id` are on the same
channel, but the spec guarantees nothing about ordering, state
continuity, or pending requests.

Every component — skills, pipeline plugins, external clients,
layer-2 systems — owns whatever state it needs. An asker that
wants `.response` correlation keeps its own outstanding-request
table; a skill that wants conversational memory keeps its own
per-session store; a layer-2 system that wants per-peer state
keys on `session_id`. Whatever a later consumer needs is **in the
Message** (`data` / `context` / `session`) or **out of band** —
never recovered from a hidden host-side index.

This is what lets layer-2 systems plug in cleanly: if OVOS kept a
central correlation index or a central conversation state, every
layer-2 system would have to replicate it, hook into it, or work
around it. Because OVOS keeps neither, they compose without
contention.

Several real concerns are deferred by this stance and are listed
under §7 known gaps: multi-turn conversation, intent context
(adapt's `add_context`/`remove_context`), the other session knobs
current OVOS carries beyond `session_id` and `lang` (`pipeline`,
`site_id`, `persona_id`, `time_format`, `date_format`,
`system_unit`, `tts_preferences`, …), and the eventual shape of
conversational state. The async-by-default model means those
future specs only need to define *what* the state is, not *how*
it travels.

### 5.3 Why HiveMind works

HiveMind is the canonical layer-2 system this design enables. A
HiveMind satellite is just another user-side emitter — it sets
`source` to its peer ID, populates `session` with a per-peer
session, and emits a Message. Inside OVOS:

- ovos-core runs the same `.reply` flip (§5.1 step 2) —
  `destination` becomes the satellite's peer ID instead of the
  local microphone.
- Skills `.forward` as usual — `destination` stays the satellite
  ID through every handler emission.
- HiveMind, watching the bus, sees each message addressed to its
  peer and routes it back over the HiveMind transport.

The pre-existing `session_id == "default"` rule keeps device-local
TTS on the device's speakers (per `ovos-audio/utils.py`'s
`require_default_session`), because remote HiveMind sessions
carry their own `session_id` and never `"default"`.

None of this required HiveMind to modify OVOS core. The mechanism
that makes it work — single-flip routing + opaque per-session
identifiers + no central state — was already in
`ovos-bus-client/message.py:194-198`; MSG-1 just names and
formalizes it.

---

## 6. Where the specs differ from current OVOS code

These specifications are *prescriptive*. Some of what they prescribe
matches what runs in OVOS today verbatim; some is a deliberate
cleanup the implementations are expected to grow into. This section
catalogues every known divergence so implementers know what to
migrate and reviewers know what to expect.

### 6.1 Already aligned

Formalizations of behaviour that exists in current OVOS code and
needs no implementation change:

- The Message envelope (`type` / `data` / `context`) — matches
  `ovos-bus-client.Message`.
- `source`, `destination` semantics including the `Message.reply`
  swap — matches `ovos-bus-client/message.py`.
- `context.session` as a serialized Session object — matches
  `ovos-bus-client/client/client.py`'s
  `message.context["session"] = sess.serialize()`.
- `session.session_id == "default"` for device-local origin —
  matches `ovos-audio/utils.py`'s `require_default_session`
  decorator.
- `session.lang` as the user's preferred language — matches the
  Session class's `lang` attribute.
- `forward` / `reply` / `response` derivation semantics — matches
  `ovos-bus-client.Message.{forward,reply,response}`.
- The `.response` suffix convention — pervasive across OVOS
  topics today.
- The `recognizer_loop:utterance` entry point and
  `complete_intent_failure` no-match topic (PIPELINE-1) — match
  current topic names verbatim.
- `ovos.utterance.cancelled` and `ovos.utterance.handled`
  (PIPELINE-1) — match current topic names verbatim.
- Per-utterance first-match-wins iteration (PIPELINE-1) — matches
  `ovos-core/intent_services/service.py`'s
  `handle_utterance` / `get_pipeline`.
- Per-session pipeline configuration (PIPELINE-1) — matches
  `Session.pipeline` (modulo the field rename in §6.3 below).
- The `<skill_id>:<intent_name>` dispatch topic shape (PIPELINE-1)
  — matches current OVOS practice; skills already subscribe to
  these topics.

### 6.2 Prescriptive renames

| Spec | Current | Prescribed | Notes |
|------|---------|------------|-------|
| INTENT-3 v1.1 | "host" | "orchestrator" | Editorial; conformance unchanged. |
| PIPELINE-1 | `mycroft.skill.handler.start` / `.complete` / `.error` | `ovos.intent.handler.start` / `.complete` / `.error` | Renamed into the `ovos.intent.*` namespace for uniformity. Breaks every existing handler-lifecycle observer; the migration cost is real (see §B in PR #11 discussion). |

### 6.3 Prescriptive shape changes

- **Keyword intent registration is atomic** (INTENT-4 §5). Today
  a keyword intent is built up via multiple `register_vocab`
  messages followed by a `register_intent` with an Adapt
  `IntentBuilder.__dict__` payload. INTENT-4 collapses this into
  a single message with structured `{required, optional, one_of,
  excluded}` arrays of vocabulary descriptors. Every skill's
  keyword-intent path needs to be rewritten in the worship layer.
- **Template intent registration uses structured identity**
  (INTENT-4 §6). Today `padatious:register_intent` carries
  `{name, samples, file_name, lang, blacklisted_words}`; the
  prescribed shape uses the structured `(skill_id, intent_name,
  lang)` triple plus `samples|file` and `blacklist|blacklist_file`.
- **Dispatch payload uses polymorphic `owner_id`** (PIPELINE-1
  §7.1). Today dispatch carries `skill_id` only. PIPELINE-1's
  `owner_id` is either a `skill_id` or a `pipeline_id` — same
  field, polymorphic value.
- **Handler-lifecycle payload includes `owner_id`** (PIPELINE-1
  §8.2). Today the trio payload is `{name: <handler_func_name>}`.
  Prescribed: `{owner_id, intent_name, optional exception}`.

### 6.4 Architectural divergences

- **The orchestrator maintains a passive registration index**
  (INTENT-4 §10). Today there is no central index — each plugin
  knows what it consumed; nothing aggregates that view. INTENT-4
  prescribes the orchestrator subscribe to all registration
  topics in parallel with plugins and serve
  `ovos.intent.list` / `ovos.intent.describe` from the passive
  view. This is a new orchestrator responsibility, not a change
  to existing behaviour.
- **Plugins are side-effect-free during `match`** (PIPELINE-1
  §4.2). This is a forward-looking rule rather than a fix for
  current code. The standard `match_high`/`match_medium`/
  `match_low` methods in the official plugins are already
  side-effect-free (they compute and return). Where side effects
  do happen today, they are orchestrator-side after the match
  wins (e.g. the `<skill_id>.activate` emit in
  `ovos-core/intent_services/service.py:365`), or in *other* bus
  handlers a plugin subscribes to. The spec rule keeps the
  current discipline normative as alternative plugin types
  (LLM-backed, agent-backed) are written.
- **`ovos.utterance.handled` on every terminal path** (PIPELINE-1
  §9.6). Current `ovos-workshop`'s `_on_event_error` does not
  emit it on the handler-error path (`ovos.py:1478-1497`). The
  spec requires it. Fix tracked separately as a workshop
  implementation bug.

### 6.5 New topics with no direct precedent

- **`ovos.intent.matched`** (PIPELINE-1 §9.2). The
  positive-match broadcast notification. Current OVOS has
  `complete_intent_failure` for the negative case but no
  positive equivalent.
- **`ovos.intent.list` / `ovos.intent.describe`** (INTENT-4 §10).
  Introspection topics served from the orchestrator's passive
  registration index.
- **Materialize-default-session rule** on `forward` / `reply` /
  `response` (MSG-1 §4.3). Formalizes a "MAY" convenience for
  in-process subsystems; not currently implemented but compatible
  with current behaviour.

### 6.6 Things the specs do *not* change

- The session object's internal shape beyond `session_id`,
  `lang`, and `pipeline` (deferred to a future session
  spec).
- The `mycroft.*` topic prefix outside the intent layer (e.g.
  `mycroft.audio.*`) — these are not part of any spec here.
- The `<skill_id>:<intent_name>` dispatch topic — kept verbatim
  from current OVOS so no skill needs to migrate its handler
  subscription.
- **Engine-specific introspection topics.** The standard plugins
  expose their own debug / inspection topics — for example
  `intent.service.adapt.reply`,
  `intent.service.adapt.manifest`,
  `intent.service.adapt.vocab.manifest`, and
  `intent.service.padatious.get`. These are
  plugin-specific surface, parallel to the spec's generic
  `ovos.intent.list` / `ovos.intent.describe` (INTENT-4 §10).
  The specs do not claim authority over them — they remain
  plugin-defined and may continue to coexist with the
  orchestrator's generic index.

### 6.7 Predecessor-topic mapping

The bus topics formalized by INTENT-4 and PIPELINE-1 replace a
number of legacy names. Implementer migration aid:

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

| Topic | Status |
|-------|--------|
| `recognizer_loop:utterance` | **unchanged** — kept as the entry point. |
| `complete_intent_failure` | **unchanged** — kept as the no-match signal. |
| `ovos.utterance.cancelled` | **unchanged** — kept as the cancellation signal. |
| `ovos.utterance.handled` | **unchanged** — kept as the universal end-marker. |
| `<skill_id>:<intent_name>` | **unchanged** — kept as the dispatch topic; PIPELINE-1 extends the shape to `<owner_id>:<intent_name>` so plugins can also own handlers. |
| `mycroft.skill.handler.start` / `.complete` / `.error` | renamed to `ovos.intent.handler.start` / `.complete` / `.error` |

#### Out of scope

| Topic | Status |
|-------|--------|
| `add_context` / `remove_context` | Adapt conversational context — not part of intent registration. A future spec may define it. |
| `<skill_id>.activate` | Activity-tracking emit currently in `ovos-core`; not part of any spec here. |

---

## 7. Known gaps and planned work

- **Per-plugin behavioural specs.** PIPELINE-1 defines the plugin
  contract (the `match` shape, the orchestrator's iteration
  semantics) but explicitly defers what each non-trivial plugin
  type actually *does*. Real candidates for their own
  specifications: `converse`, `fallback`, `common_query`, `ocp`,
  `persona`, `stop`. Each defines its own internal behaviour and
  its own bus emissions beyond the universal lifecycle PIPELINE-1
  prescribes.
- **A session specification.** MSG-1 §4 carries `session` opaquely
  and names only `session_id` and `lang`; PIPELINE-1 §5 adds
  `pipeline`. Everything else about the session is
  deferred — session lifecycle (start, end, expiry, resumption),
  the full set of session preferences current OVOS already carries
  (`site_id`, `persona_id`, `time_format`, `date_format`,
  `system_unit`, `tts_preferences`, …), and the shape of any
  conversational state. A future session specification picks
  these up.
- **A multi-turn conversation specification.** When a skill asks
  the user a question and waits for the next utterance, the "next
  utterance belongs to that pending question" link is not
  formalized today (handled informally by the `converse` plugin
  type plus skill-side state). MSG-1's async-by-default stance
  (§5.2) leaves room for this to be formalized either in the
  session spec or as a separate one.
- **Intent context.** Formalized in **OVOS-CONTEXT-1** — see §4
  *Intent context* above. The Adapt-era `add_context` /
  `remove_context` feature is lifted to a session-bound,
  decaying, engine-agnostic primitive.
- **The utterance-transformer chain.** Formalized in
  **OVOS-TRANSFORM-1** — see §4 *Transformer plugins* above —
  covering six injection points (audio, utterance, metadata,
  intent, dialog, TTS) and their cancellation contract.
- **Text normalization of ASR output.** The basis for slot value
  typing (OVOS-INTENT-1 §5.3). Deferred to its own specification.
- **A machine-checkable conformance corpus** of `template → sample
  set` pairs for OVOS-INTENT-1 expansion, so expander conformance
  can be verified automatically. A parallel corpus of bus-message
  fixtures for MSG-1 would be the equivalent at the bus layer.
- **An end-to-end worked example.** The specs have local examples;
  none shows a single skill defining one keyword intent and one
  template intent through the whole path — files, registration,
  match, handler.
- **i18n corpus.** OVOS-INTENT-2 defines the locale file format, and
  ovos-localize (§8) provides the operations layer; what remains is
  the *scale* of the translated corpus.

---

## 8. Ecosystem tooling: ovos-localize

The specifications define formats and contracts; turning those into a working
i18n operation takes tooling. **ovos-localize** is that layer — a GitHub-native
localization platform for OVOS skills, built specifically around the resource
roles of OVOS-INTENT-2.

It scans skill repositories for locale files; analyzes each skill's Python
source (via AST) to recover the **handler context** of a resource — which
function uses a file, what its slots mean, what dialog it triggers, which is
exactly the intent↔handler binding of OVOS-INTENT-3 §1; validates translations
against a rule set (slot preservation, expansion validity, variant counts); and
lets translators browse, edit, preview, and submit translations as pull
requests. It also exports a unified intent/dialog/vocabulary dataset.

ovos-localize is the OVOS counterpart to Home Assistant's managed
`intents` repository. Two honest notes: it is currently
**descriptive** of real OVOS skills — it also handles legacy file
types these specs deliberately drop — so as the specs and the
ecosystem converge, its file-type coverage and the specs will need to
meet in the middle; and its translation validators are a natural home
for spec conformance checks, distinct from but related to the planned
grammar-level conformance corpus (§7).

---

## 9. Design history

How the specification set was arrived at — context that explains
the *why*, but that has no place in a normative document.

### 9.1 The set, in three stacks

Built bottom-up in three stacks:

- The **intent stack**, in dependency order: OVOS-INTENT-1
  (template grammar) → OVOS-INTENT-2 (resource files) →
  OVOS-INTENT-3 (the intent concept) → OVOS-INTENT-4 (the
  registration wire format on the bus).
- The **bus stack**, anchored on existing `ovos-bus-client` wire
  format: OVOS-MSG-1 formalizes the envelope, routing, session
  carrier, and `forward`/`reply`/`response` derivations.
  Originally drafted as two specs (envelope + session/routing) and
  merged once it became clear the derivations could only
  meaningfully be defined where the routing keys lived.
- The **orchestrator stack**: OVOS-PIPELINE-1 defines the
  orchestrator, the pipeline-plugin abstraction, the utterance
  lifecycle, and the handler-lifecycle trio. Sits on top of the
  bus stack (uses MSG-1's envelope and routing) and around the
  intent stack (intent registrations are one kind of input
  pipeline plugins consume).

Each was a formalization pass over machinery already running in
production (§1), not a greenfield design.

### 9.2 The reference implementation

The specs are implementation-agnostic, but a spec benefits from
one conformant implementation. **ovos-spec-tools** is that for
the intent stack — expander, resource loader, dialog renderer,
language matching, locale linter, in one dependency-light
package. It exists because the same machinery had drifted across
six separate copies in the ecosystem; ovos-spec-tools is what
those components are meant to converge on, and the intended home
of the planned conformance corpus.

The bus and orchestrator stacks do not yet have a comparable
reference; `ovos-bus-client` is the closest match for MSG-1 and
`ovos-core` is the closest match for PIPELINE-1 + INTENT-4, but
both predate the specs.

### 9.3 Audit-driven refinement

Before initial release, each spec was revised across several
review rounds — malformed-form rules, the expansion algorithm,
slot handling, the envelope/routing split (later un-split, see
§9.1), the host → orchestrator rename, the
intent-stage-vs-non-intent-stage distinction (later dissolved
into the uniform pipeline-plugin abstraction), cross-spec
terminology. Those rounds happened pre-release, so they left no
intermediate version numbers behind: the audited result *is*
version 1 (or 1.1 where editorial-only). The CHANGELOG records
versioned changes from there on.

---

## 10. Compatibility levels

Each specification carries its own integer (or minor) `Version`,
bumped per PR per the contributing rules in the README. The
architecture as a whole was previously spoken of at
**compatibility levels** — versioned snapshots a tool may target,
checked against by `ovos-spec-lint`.

The compatibility-level model was designed when the architecture
was one stack (the intent grammar / resources / intent definition
chain) and a single integer cleanly identified "all the specs at
once." With the addition of the bus and orchestrator stacks, that
single-axis model no longer describes the architecture.

The historical intent-stack ladder:

- **V0** — *informal.* The undocumented, de-facto behaviour from
  before these specifications existed. V0 is not specified
  anywhere; it is the baseline the formalization started from.
  V0 has no notion of the `.blacklist` resource role or of
  `<name>` references.
- **V1** — the intent stack as first formalized: OVOS-INTENT-1,
  -2 and -3, each at version 1. V1's headline addition over V0
  is the `.blacklist` role.
- **V2** — V1 plus **inline vocabulary references** (the
  `<name>` token): OVOS-INTENT-1 and OVOS-INTENT-2 at version 2.
  A V2 template cannot be expanded by a V1 tool.

These intent-stack levels continue to make sense in isolation.
The bus stack (OVOS-MSG-1), the registration spec (OVOS-INTENT-4),
and the orchestrator spec (OVOS-PIPELINE-1) are versioned
**individually** and not placed on a unified compatibility
ladder. A tool targeting them today cites per-spec versions:
"MSG-1 v1, INTENT-4 v1, PIPELINE-1 v1." Whether the compat-level
model evolves into a multi-axis grid, per-stack ladders, or is
quietly deprecated in favour of per-spec versions only, is
deferred.

