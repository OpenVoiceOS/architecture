---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

> **⚠️ AI-generated draft — not yet fully reviewed.** This content
> was produced by a large language model (Claude Code) and
> has not yet been fully reviewed for accuracy, completeness, or
> consistency with the specifications. The normative specifications
> themselves are human-reviewed; this appendix is supplementary
> context. Readers should verify claims before relying on them.

## 1. About the OVOS specifications

### 1.0 The voice operating system concept

The term *voice operating system* is precise, not marketing. The
distinction matters because OVOS is routinely conflated with two
things it is not:

**It is not a voice assistant product.** A voice assistant is a
closed, vertically-integrated product — a single vendor controls
the NLU, the dialogue policy, the skill ecosystem, and the output
layer. It answers questions. A voice operating system is a
*platform*: it defines contracts that arbitrary third-party
components implement independently, and the platform's job is to
arbitrate between them. The analogy to a general-purpose OS is
direct. The pipeline is a scheduler: it has a priority order, a
first-match-wins dispatch policy, and a circuit-breaker for failing
components. The bus is IPC: broadcast delivery, no central
authority, no guaranteed ordering beyond the single-flip routing
model. The session carrier is shared memory: it propagates opaquely
through every message and every component reads and writes its
owned slice. The handler-lifecycle trio is process supervision: the
orchestrator wraps every handler invocation with start/complete/error
events regardless of what the handler does. Pipeline plugins and
transformer plugins are loadable modules: swapped, replaced, and
composed at deployment time with no changes to the ABI.

**It is not an LLM wrapper.** A language model fits the voice OS
model as a first-class plugin — and in multiple roles. As a
*pipeline plugin*, it implements `match(utterances, lang, session)
→ Match`, returning a match immediately and deferring generation to
its handler (PIPELINE-1 §4.4). As an *utterance transformer*, it
paraphrases, normalizes, or expands the inbound candidate list
before matching (TRANSFORM-1 §3.2). As a *dialog transformer*, it
rewrites the handler's natural-language response before delivery
(TRANSFORM-1 §3.5). As a *metadata transformer*, it enriches the
utterance with detected intent signals before the pipeline sees it
(TRANSFORM-1 §3.3). In each role, the model is one implementation
of a defined plugin contract — swappable, composable, and neutral
to the platform. Whether any LLM is loaded at all, and in which
roles and at what priority, is a deployment decision. An
architecture organized around a single model call is not a voice OS;
it is one possible single-plugin deployment of one.

The consequence of the OS framing: a skill written against the
intent stack runs on any conformant orchestrator, under any pipeline
configuration, with any combination of NLU backends, in any language
the deployment supports. The platform's only invariant is the ABI —
the wire contracts these specifications define.

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
  surfaces). OVOS-STOP-1 defines the stop plugin, the reserved
  intent_name `stop`, and the stoppability-discovery cascade
  (the **interrupt primitive**). OVOS-COMMON-QUERY-1 defines
  the common query plugin, the scatter-gather question-answering
  protocol, and the skill-side question bus contract (the
  **multi-answer primitive**). OVOS-TRANSFORM-1 defines the six injection-point
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
PR per the contributing rules in the README.

For the **intent stack**, a single integer identifies a coherent
grammar / resources / intent-definition snapshot checked by
`ovos-spec-lint`. The ladder:

- **V0** — undocumented pre-spec baseline; no `.blacklist`, no `<name>` references.
- **V1** — INTENT-1, -2, -3 at v1; headline addition is the `.blacklist` role.
- **V2** — V1 plus inline vocabulary references (`<name>`); a V2 template cannot be expanded by a V1 tool.

The bus and orchestrator stacks are versioned **individually**
and not placed on a unified ladder — a tool targeting them cites
per-spec versions ("MSG-1 v2, PIPELINE-1 v2").

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
