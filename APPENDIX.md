# Appendix — Design Notes and Context

**Non-normative.** This document is a companion to the OVOS formal
specifications. It records design rationale, comparisons with other systems,
and topics worth discussing that do not belong in a normative specification.
Nothing here is binding — OVOS-INTENT-1, OVOS-INTENT-2, and OVOS-INTENT-3 are
the only normative documents. This appendix exists so the specs themselves can
stay terse and requirement-focused.

---

## 1. These specifications formalize an existing system

The OVOS intent stack — the engines (padatious, Adapt), the skill ecosystem,
the resource file formats, the pipeline — already exists and runs in
production. These specifications were written **after** the system they
describe. They are a *formalization pass*: they document an existing design
implementation-agnostically, tighten under-defined corners, and remove
accidental inconsistencies, so the contracts can be implemented by new engines
and adopted by other assistants.

This matters for how to read them. They are **prescriptive** — each spec states
a clean target, and where it diverges from current OVOS behaviour the
divergence is a deliberate cleanup — but they are not speculative. The target
is a lightly-cleaned version of a working system, not a greenfield design.
padacioso and ovos-workshop are the closest existing implementations; neither
yet fully conforms, and bringing them into conformance is planned work.

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
  the tooling counterpart in **ovos-localize** (§6) — a GitHub-native
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
(§6). The grammar itself is a commodity shared by all three; the OVOS bet is the
engine-agnostic contract and the pipeline.

---

## 3. What these specifications do not cover — the pipeline

OVOS-INTENT-1, -2, and -3 formalize **intent definition**: the grammar, the
resource files, what an intent is, and the intent-engine contract. That is one
slice of the OVOS intent stack.

The larger structure is the **pipeline**. In OVOS, intent resolution is an
ordered, user-configurable chain of stages tried in priority order until one
matches. The stages include far more than intent engines:

- `stop` — halt active skills;
- `converse` — route a follow-up utterance to a skill that is mid-dialogue;
- `padatious` and `adapt` — template and keyword intent engines, each tried at
  high, medium, and low confidence tiers, interleaved;
- `common_query` — route factual questions to question-answering skills;
- `fallback` — generic handlers when nothing else matched;
- `ocp` — media playback requests;
- `persona` — an LLM-backed conversational agent (see below).

Keyword and template intent engines are two stage kinds among roughly eight.
The pipeline's ordering, its high/medium/low tier model, and the contracts for
the non-intent stages are **not yet specified**. OVOS-INTENT-3 references the
"host" and a "pipeline plugin" but stops at the intent-engine boundary.

This is the natural next formalization. The pipeline is what makes OVOS
distinctive relative to HA and Rhasspy, and it is currently undocumented.

**ovos-persona** is worth singling out: it is an LLM-backed persona system that
plugs into the pipeline as a first-class stage (`persona-high`, `persona-low`),
not as a bolt-on. OVOS-INTENT-3 §6.2's non-normative note — "an installation
may load an LLM- or chatbot-based engine" — is not hypothetical; it describes
something that already ships. The engine-agnostic contract is already realized.

The confidence tiers plus the ordered fallback chain (deterministic Adapt
before fuzzy Padatious before an LLM persona last) are also how the system
*manages* the open-endedness of engine generalization: generalization is not
unconstrained in practice, it is bounded by where an engine sits in the chain.

---

## 4. Design rationale

Short notes on *why* the specifications make the choices they do — the
reasoning, not the requirement.

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

---

## 5. Known gaps and planned work

- **A pipeline specification.** Stage ordering, the confidence-tier model, and
  the contracts for `converse`, `fallback`, `common_query`, `ocp`, and
  `persona` stages are unspecified (§3). This is the highest-value next spec.
- **Text normalization of ASR output.** The basis for slot value typing
  (OVOS-INTENT-1 §5.3). Deferred to its own specification.
- **A machine-checkable conformance corpus** of `template → sample set` pairs
  for OVOS-INTENT-1 expansion, so expander conformance can be verified
  automatically.
- **An end-to-end worked example.** The specs have local examples; none shows a
  single skill defining one keyword intent and one template intent through the
  whole path — files, registration, match, handler.
- **i18n corpus.** OVOS-INTENT-2 defines the locale file format, and
  ovos-localize (§6) provides the operations layer; what remains is the *scale*
  of the translated corpus — an ongoing community effort, not a missing piece
  of design or tooling.

---

## 6. Ecosystem tooling: ovos-localize

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

ovos-localize is the OVOS counterpart to Home Assistant's managed `intents`
repository. Two honest notes: it is currently **descriptive** of real OVOS
skills — it also handles legacy file types these specs deliberately drop — so
as the specs and the ecosystem converge, its file-type coverage and the specs
will need to meet in the middle; and its translation validators are a natural
home for spec conformance checks, distinct from but related to the planned
grammar-level conformance corpus (§5).

---

## 7. Design history

How the specification set was arrived at — context that explains the *why*,
but that has no place in a normative document.

### 7.1 Three specs, in dependency order

The set was built bottom-up, each spec depending on the one before it:

- **OVOS-INTENT-1** formalizes the sentence template grammar — the
  bracket-expansion syntax that padatious-like engines and skill resource
  files already used informally.
- **OVOS-INTENT-2** builds on it to formalize the `locale/` folder and the
  resource file roles.
- **OVOS-INTENT-3** builds on both to define what an intent *is* — a
  developer's binding from a natural-language command to a handler — and the
  two ways to define one (keyword and template).

Each was a formalization pass over machinery already running in production
(§1), not a greenfield design.

### 7.2 Prescriptive, not descriptive

The specs describe a **clean target**, not current OVOS behaviour in full.
Where the existing system carried accidental inconsistencies or legacy cruft,
the specs diverge deliberately — they are something for OVOS to grow into, not
a transcript of what it does. A handful of those divergences were genuine
decisions, resolved explicitly:

- **Nested locale directories** are allowed — a `locale/<lang>/` tree may have
  subdirectories, resolved by recursive search. This matches current
  behaviour, and was kept rather than forcing a flat layout.
- **Legacy file types are dropped.** `.rx`, `.value`, `.list`, `.word`,
  `.template`, and `.qml` are not resource roles in OVOS-INTENT-2. Regex
  entities in particular are recommended against — they localize poorly.
- **`.blacklist` is new.** Intent suppression was previously ad hoc (a list of
  `.voc` files passed as `voc_blacklist`); the `.blacklist` role and the
  keyword `excluded` constraint formalize it. This is a prescriptive addition
  for OVOS to adopt, not a description of today.
- **Slot names may contain digits**, aligning the rule with what skills
  already write.
- **The `{{ }}` double-brace dialog form is dropped** — a
  backward-compatibility artifact; only the single-brace `{name}` is
  recognized.

### 7.3 Audit-driven refinement

Before the first release the specs were revised across several review rounds —
the malformed-form rules, the expansion algorithm, slot handling, and
cross-spec terminology were all tightened. Those rounds happened pre-release,
so they left no intermediate version numbers behind: the audited result *is*
version 1. The CHANGELOG records versioned changes from there on.

### 7.4 OVOS-INTENT-1 version 2 — inline vocabulary references

The one feature that is *not* a formalization of existing behaviour is the
`<name>` inline vocabulary reference — the equivalent of Home Assistant's
`expansion_rules` (§2.2). It reuses the existing `.voc` role rather than
adding a separate file type, so the change is one grammar token plus an
expander step. It arrived with OVOS-INTENT-1 version 2 (issue #1, PR #2).
Because a `<name>` template cannot be expanded by a version-1 tool, it is a
breaking change, and so version 2 is a major version bump.

### 7.5 The reference implementation

The specifications are implementation-agnostic, but a spec benefits from one
conformant implementation to point at. **ovos-spec-tools** is that — the
expander, the resource loader, the dialog renderer, language matching, and a
locale linter, in one dependency-light package. It exists because the same
machinery had been reimplemented and had drifted across the ecosystem: bracket
expansion alone existed in six separate copies, and language matching in
several more. ovos-spec-tools is the single conformant implementation those
components are meant to converge on, and the intended home of the planned
conformance corpus (§5).

### 7.6 What was deliberately left out

Two things were consciously deferred rather than rushed:

- **Slot value typing** — interpreting a slot as a number or a date — is left
  unspecified, because it is inseparable from a normalization of ASR output
  that does not yet exist (§4; OVOS-INTENT-1 §5.3).
- **The pipeline** — the ordered, multi-stage intent-resolution chain — is the
  largest unformalized piece, and the natural next specification (§3).
