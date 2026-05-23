# Appendix — Design Notes and Context

**Non-normative.** This document is a companion to the OVOS formal
specifications. It records design rationale, comparisons with other
systems, the catalogue of *deliberate* divergences from current OVOS
code, and topics worth discussing that do not belong in a normative
specification. Nothing here is binding — OVOS-INTENT-1, OVOS-INTENT-2,
OVOS-INTENT-3, and OVOS-MSG-1 are the only normative documents. This
appendix exists so the specs themselves can stay terse and
requirement-focused.

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

## 3. The pipeline — the host-side spec

The intent specs (OVOS-INTENT-1/2/3/4) formalize **intent definition
and delivery**: the grammar, the resource files, what an intent is,
the intent-engine contract, and the bus messages that carry
registration, match, and dispatch. OVOS-MSG-1 formalizes the bus
that carries them. **OVOS-PIPELINE-1** formalizes the
host-side piece that sits *around* all of those — the utterance
lifecycle, the transformer chain, the ordered pipeline of stages
(intent engines, `converse`, `fallback`, `common_query`, `ocp`,
`persona`, …), the confidence-tier convention, and the universal
`ovos.utterance.handled` end-marker.

What PIPELINE-1 leaves out by design: the **internal behaviour of
non-intent stages**. Each of `converse`, `fallback`, `common_query`,
`ocp`, `persona`, `stop` is the natural subject of its own future
specification — PIPELINE-1 only defines the contract every stage
conforms to. See §7 known gaps for the list.

One observation worth flagging: **the engine-agnostic intent
contract is already realized**, not hypothetical. `ovos-persona`
plugs into the pipeline as a first-class LLM stage (`persona-high`,
`persona-low`) — the OVOS-INTENT-3 §6.2 non-normative note about
LLM-backed engines describes something that ships today. The
ordered confidence-tier chain (deterministic Adapt before fuzzy
Padatious before an LLM persona last) is also how the system
*bounds* engine generalization in practice: generalization is not
unconstrained, it is bounded by where an engine sits.

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
migrate and reviewers know what to expect. (OVOS-MSG-1 is by far the
spec closest to current code; the catalogue below is correspondingly
short. Later specs will add more entries.)

### 6.1 Already aligned

The following are formalizations of behaviour that already exists in
current OVOS code paths and need no implementation change:

- The Message envelope (`type` / `data` / `context`) — matches
  `ovos-bus-client.Message`.
- `source`, `destination` semantics, including the
  `Message.reply` swap — matches `ovos-bus-client/message.py`.
- `context.session` as a serialized Session object — matches
  `ovos-bus-client/client/client.py`'s `message.context["session"] =
  sess.serialize()`.
- `session.session_id == "default"` for device-local origin — matches
  `ovos-audio/utils.py`'s `require_default_session` decorator.
- `session.lang` as the user's preferred language — matches the
  Session class's `lang` attribute and existing OVOS read paths.
- `forward` / `reply` / `response` derivation semantics — matches
  `ovos-bus-client.Message.{forward,reply,response}`.
- The `.response` suffix convention — pervasive across OVOS topics
  today.

### 6.2 New, no legacy

The only thing OVOS-MSG-1 introduces that has no direct precedent in
current code:

- The **materialize-default-session** rule on `forward` / `reply` /
  `response` (MSG-1 §4.3) — formalizes a "MAY" convenience for
  in-process subsystems; not currently implemented, but compatible
  with current behaviour (today `session` is propagated only when
  present, never materialized).

### 6.3 Things the spec does *not* change

- The session object's internal shape beyond `session_id` and `lang`
  — every other field current OVOS puts inside `context.session`
  remains opaque under this spec until the future session
  specification.
- The Mycroft-era `mycroft.*` topic prefix outside the intent layer
  (e.g. `mycroft.audio.*`) — these are not part of any spec here and
  are out of scope.

---

## 7. Known gaps and planned work

- **A bus-level intent registration and dispatch spec.** OVOS-MSG-1
  defines the envelope and the routing/session keys, but the
  *concrete topics* for intent registration, match notification,
  handler dispatch, and the handler-lifecycle messages
  (`mycroft.skill.handler.{start,complete,error}` etc.) are still
  informal. The natural next bus spec is OVOS-INTENT-4, which builds
  on OVOS-MSG-1 + OVOS-INTENT-3.
- **Per-stage behavioural specs.** OVOS-PIPELINE-1 defines the
  stage *contract* (the `match` signature, the `StageMatch` shape,
  the confidence-tier convention) but explicitly defers what each
  non-intent stage actually *does*. `converse`, `fallback`,
  `common_query`, `ocp`, `persona`, `stop` are each natural
  subjects of their own specifications.
- **A session specification.** MSG-1 §4 carries `session` opaquely
  and names only `session_id` and `lang`. Everything else about the
  session is deferred — see §5.2 for the explicit list: session
  lifecycle (start, end, expiry, resumption), the full set of
  session preferences current OVOS already carries (`pipeline`,
  `site_id`, `persona_id`, `time_format`, `date_format`,
  `system_unit`, `tts_preferences`, …), and the shape of any
  conversational state. The future session specification will pick
  these up; MSG-1's job is to make sure the carrier is in place.
- **A multi-turn conversation specification.** When a skill asks a
  question and waits for the next utterance, the "next utterance
  belongs to that pending question" link is not formalized today
  (handled informally by `converse` + skill-side state). MSG-1's
  async-by-default stance (§5.2) leaves room for this to be
  formalized either in the session spec or as a separate one.
- **Intent context.** Adapt's `add_context` / `remove_context`
  feature — where one intent's match influences a later intent's
  eligibility — is not formalized at the spec level. See §5.2.
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

### 9.1 The set, in two stacks

Built bottom-up in three stacks:

- The **intent stack**, in dependency order: OVOS-INTENT-1 (template
  grammar) → OVOS-INTENT-2 (resource files built on it) →
  OVOS-INTENT-3 (the intent concept, built on both).
- The **bus stack**, anchored on existing `ovos-bus-client` wire
  format: OVOS-MSG-1 formalizes the envelope, routing, session
  carrier, and `forward`/`reply`/`response` derivations.
  Originally drafted as two specs (envelope + session/routing) and
  merged once it became clear the derivations could only
  meaningfully be defined where the routing keys lived.
- The **host stack**, sitting around the intent and bus stacks:
  OVOS-PIPELINE-1 formalizes the utterance lifecycle, the
  transformer chain, the ordered pipeline of stages (intent and
  non-intent), the confidence-tier convention, and the universal
  `ovos.utterance.handled` end-marker. It builds on OVOS-MSG-1
  and provides the host context that all the other specs operate
  inside.

Each was a formalization pass over machinery already running in
production (§1), not a greenfield design.

### 9.2 The reference implementation

The specs are implementation-agnostic, but a spec benefits from
one conformant implementation. **ovos-spec-tools** is that for the
intent stack — expander, resource loader, dialog renderer, language
matching, locale linter, in one dependency-light package. It
exists because the same machinery had drifted across six separate
copies in the ecosystem; ovos-spec-tools is what those components
are meant to converge on, and the intended home of the planned
conformance corpus.

The bus stack does not yet have a comparable reference;
`ovos-bus-client` is the closest match for MSG-1 but predates the
spec.

### 9.3 Audit-driven refinement

Before initial release, each spec was revised across several review
rounds — malformed-form rules, the expansion algorithm, slot
handling, the envelope/routing split (later un-split, see §9.1),
cross-spec terminology. Those rounds happened pre-release, so they
left no intermediate version numbers behind: the audited result
*is* version 1. The CHANGELOG records versioned changes from there
on.

---

## 10. Compatibility levels

Each specification carries its own integer `Version`, bumped per
PR per the contributing rules in the README. The architecture as a
whole is also spoken of at **compatibility levels** — versioned
snapshots a tool may target, and that `ovos-spec-lint` checks
against.

The levels defined to date apply to the **intent stack**
(OVOS-INTENT-1/2/3):

- **V0** — *informal.* The undocumented, de-facto behaviour of
  Mycroft- and OVOS-derived code from before these specifications
  existed. V0 is not specified anywhere; it is the baseline the
  formalization started from, named here only so tools can refer to
  "pre-spec" behaviour. V0 has no notion of the `.blacklist`
  resource role or of `<name>` references.
- **V1** — the specifications as first formalized: OVOS-INTENT-1,
  -2 and -3, each at version 1. V1's headline addition over V0 is
  the `.blacklist` role — formalized intent suppression.
- **V2** — V1 plus **inline vocabulary references** (the `<name>`
  token): OVOS-INTENT-1 and OVOS-INTENT-2 at version 2. A V2
  template cannot be expanded by a V1 tool, so V2 is not backward
  compatible with V1.

A specification that does not change between levels keeps its
lower version number — OVOS-INTENT-3 is at version 1 in both V1
and V2.

### How the bus and host stacks will be layered in

OVOS-MSG-1 introduces the bus envelope; OVOS-INTENT-4 introduces
intent registration and dispatch on top of it; OVOS-PIPELINE-1
introduces the host-side utterance lifecycle. All three are
structurally orthogonal to the intent grammar/resource stack — a
tool can implement either without the other. As a result the
single-axis V0/V1/V2 ladder above is no longer sufficient to
describe the architecture as a whole.

The compatibility-level model is expected to evolve into either a
multi-axis grid or per-stack ladders. Until that's settled, the
bus and host specs (OVOS-MSG-1, OVOS-INTENT-4, OVOS-PIPELINE-1)
are versioned individually but not yet placed on a compatibility
ladder. Implementers targeting them today should cite each spec's
own `Version` field rather than a compat level.
