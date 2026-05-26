# OVOS Formal Specifications

Formal, implementation-agnostic specifications for the OpenVoiceOS
voice assistant ecosystem.

This repository is the **source of truth** for how OVOS components
talk to each other and what their data shapes mean. The specs are
written generically so they can be implemented by any tool, in any
language, and adopted by voice assistants beyond OVOS.

> ⚠️ **Draft.** Specs in this repository are at **Draft** status.
> Implementations are being brought into conformance progressively;
> current OVOS behaviour may not yet match these documents. Where
> it diverges, that is a known implementation bug — not a defect in
> the specification (see *Authority* below). See
> [APPENDIX §5](APPENDIX.md) for the divergence catalogue. The
> notice will be removed when the set reaches a stable status (see
> [#5](https://github.com/OpenVoiceOS/architecture/issues/5)).

---

## Goals

The specs exist to make three things possible:

- **Interoperability.** Multiple implementations — engines, hosts,
  plugins, even non-OVOS assistants — can target the same observable
  contract instead of reverse-engineering each other's code.
- **Stability.** Implementation churn no longer drifts the contract.
  Each spec is a versioned document; behaviour changes go through
  a pull request with a version bump.
- **Adoption beyond OVOS.** The specs are written
  implementation-agnostically so other voice-assistant projects can
  adopt the same formats, grammar, and bus contracts without
  buying into OVOS as a whole.

The specs cover formats and contracts only. They do not mandate
implementation choices — programming language, internal design,
storage, threading, transport — those are the implementer's. What
they fix is the **observable contract**.

---

## Authority

These specifications are **prescriptive, not descriptive**. They
define the intended architecture; they are not a transcript of how
any current code behaves. Where an implementation — in OpenVoiceOS
or any other project — diverges from a spec here, that divergence
is a **bug in the implementation**, not in the specification.

Anyone is free to **adopt** these specifications and free to
**propose changes** to them via pull request (see [contributing]
below). Adoption is voluntary; conformance, once adopted, is not.

[contributing]: #contributing

---

## Specification set

The specs are organised into three stacks, each layered on the one
above. Read a stack top-to-bottom for that subsystem's full
picture; cross-stack dependencies are noted in each spec's header.

### Intent stack — what a skill defines

Defines the language a skill uses to declare *"these commands
trigger me"*, the files that carry the declarations, and the bus
contract for registering them with an orchestrator.

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-INTENT-1 | [Sentence Template Grammar](sentence-template-grammar.md) | 2 | Draft |
| OVOS-INTENT-2 | [Locale Resource Formats](locale-resource-formats.md) | 2 | Draft |
| OVOS-INTENT-3 | [Intent Definition](intent-definition.md) | 1 | Draft |
| OVOS-INTENT-4 | [Intent and Entity Registration](intent-registration.md) | 2 | Draft |

Numbered in dependency order: INTENT-1 (grammar) → INTENT-2
(resource files) → INTENT-3 (the intent concept) → INTENT-4 (the
bus-level registration contract).

### Bus stack — how components talk

Defines the message envelope every interaction rides on, the
session carrier that propagates per-conversation state, and the
lifecycle / ownership rules for that state.

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-MSG-1 | [Bus Message](message-object.md) | 1 | Draft |
| OVOS-SESSION-1 | [Session Carrier Wire Shape](session.md) | 1 | Draft |
| OVOS-SESSION-2 | [Session Lifecycle and State Ownership](session-lifecycle.md) | 1 | Draft |

MSG-1 owns the envelope (`Message` shape, `Message.context`,
`forward`/`reply`/`response` derivations, routing keys). SESSION-1
owns the wire shape of the `session` carrier and the field
registry by which other specs claim session fields. SESSION-2
owns lifecycle, state ownership (stateless orchestrator for named
sessions, orchestrator-owned default session), client-side merge
semantics, and the SHOULD-project pathway for component-held
cross-utterance state.

### Orchestrator stack — what processes utterances

Defines the utterance lifecycle from inbound message to terminal
end-marker, the pipeline-plugin contract, the six injection-point
transformer chains, the declarative continuous-dialog primitive,
and the imperative complement.

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-PIPELINE-1 | [Utterance Lifecycle and Pipeline](pipeline.md) | 2 | Draft |
| OVOS-TRANSFORM-1 | [Transformer Plugins](transformer.md) | 1 | Draft |
| OVOS-CONTEXT-1 | [Intent Context](intent-context.md) | 1 | Draft |
| OVOS-CONVERSE-1 | [Active Handlers and Interactive Response](converse.md) | 1 | Draft |

PIPELINE-1 owns the orchestrator and the pipeline-plugin contract
(`match` returns a `Match | null`; `Match.updated_session` is the
match-phase session-mutation channel). TRANSFORM-1 owns the six
lifecycle hooks at which transformers may mutate the utterance
and session. CONTEXT-1 owns the **declarative** continuous-dialog
primitive (decaying key/value gates on intents). CONVERSE-1 owns
the **imperative** complement (the active-handler recency stack,
the converse-plugin role, and the interactive response-collection
mechanism).

Each spec carries its own scope statement, design rationale, and
conformance section in its header. Open the document for the full
picture — the tables above are an index.

---

## Where to start

Pick the reading order that matches what you're building:

- **Writing a skill?** Start with the intent stack
  (INTENT-1 → -2 → -3). INTENT-4 only if you need to understand
  the registration wire format; otherwise your skill framework
  handles it.
- **Building a pipeline plugin?** Start with PIPELINE-1, then
  SESSION-1 and SESSION-2 for the session model, then the
  domain-specific spec for your plugin role (CONVERSE-1 for
  conversational plugins, CONTEXT-1 if you operate on intent
  context, TRANSFORM-1 if you're a transformer).
- **Building an orchestrator?** Start with MSG-1 and the full
  SESSION stack (SESSION-1 + SESSION-2) for the envelope and
  state model, then PIPELINE-1 for the lifecycle, then the
  consumer specs (INTENT-4, CONTEXT-1, CONVERSE-1, TRANSFORM-1).
- **Surveying the architecture?** Start with [APPENDIX §1](APPENDIX.md)
  for the three-stack narrative, then skim each spec's §1 Scope
  and §N Conformance sections. [APPENDIX §3](APPENDIX.md) walks
  through the architectural patterns that hold across the set.

For background — design rationale, comparisons with other systems,
the catalogue of known divergences from current code, and known
gaps — see [APPENDIX.md](APPENDIX.md). For term definitions, see
[GLOSSARY.md](GLOSSARY.md). For the version history of each spec,
see [CHANGELOG.md](CHANGELOG.md).

---

## How this compares to other voice frameworks

[APPENDIX §2](APPENDIX.md) walks through the comparison in detail.
Brief positioning:

- **Home Assistant Assist / hassil** share grammar lineage with
  OVOS-INTENT-1/-2; the Mycroft-era template syntax is the common
  ancestor. OVOS spec set extends much further into the orchestrator
  and conversational layers, which Assist currently leaves to the
  conversation agent.
- **Rasa** has comparable rigor on dialogue policy (stories,
  forms) but the OVOS specs deliberately stay implementation-
  agnostic about *how* a dialogue is steered, focusing on wire
  contracts.
- **Alexa Skills Kit / Dialogflow** are commercial platforms with
  proprietary infrastructure; the OVOS spec set covers
  comparable scope (intent registration, session, dialog, NLU
  plugins) but as an open contract anyone can implement.
- **Rhasspy / Hermes** is the closest precedent for a fully-spec'd
  open voice assistant; OVOS extends the model with the
  orchestrator stack, transformer plugins, and the
  imperative/declarative continuous-dialog split.
- **Wyoming** is a stateless RPC protocol for audio/intent;
  complementary, not a competitor. An OVOS deployment could front
  a Wyoming-speaking STT or TTS service through the appropriate
  transformer hook.

---

## Reference implementation

[**ovos-spec-tools**](https://github.com/OpenVoiceOS/ovos-spec-tools)
is a reference implementation for the **intent stack** — a
dependency-light Python package providing the sentence-template
expander, the locale resource loader, the dialog renderer, language
matching, and the `ovos-spec-lint` linter. Components that don't
want to reimplement the spec machinery themselves can depend on
it. It is also the intended home of the planned conformance
corpus.

The **bus and orchestrator stacks** do not yet have comparable
ground-up reference implementations. `ovos-bus-client` is the
closest existing match for the bus stack but predates the spec
(see [APPENDIX §5](APPENDIX.md) for the divergence catalogue);
`ovos-core` implements the orchestrator stack with similar
caveats. Both will be brought into conformance progressively.

---

## Implementation status

Where this matters for adoption decisions:

- **Spec set:** moving from Draft to a stable status is tracked in
  [#5](https://github.com/OpenVoiceOS/architecture/issues/5).
  Until then, individual specs may receive breaking revisions
  between published versions.
- **Conformance to current ovos-core:** the intent stack is the
  most aligned; the bus, session, and orchestrator stacks have
  prescriptive renames and shape changes documented in
  [APPENDIX §5](APPENDIX.md).
- **Known gaps:** [APPENDIX §7](APPENDIX.md) tracks specifications
  the set deliberately leaves for follow-up work — most notably
  a stop / interrupt specification (forward-referenced from
  CONVERSE-1) and per-plugin behavioural specs for
  `fallback` / `common_query` / `ocp` / `persona`.

---

## Contributing

Specifications are **versioned documents, not living wikis**. Any
change to a spec — however small — **MUST** be submitted as a pull
request, never committed directly. Each PR changes exactly one
normative file; supporting metadata edits (this README,
CHANGELOG, GLOSSARY entries) go in separate PRs.

Each PR that alters normative content **MUST**:

- bump the spec's `Version` field in its header (at merge time);
- add a corresponding entry to [CHANGELOG.md](CHANGELOG.md).

A version identifies an exact, citable state of a document, so
implementations and conformance results can name the version they
target.

PRs that touch only the non-normative material —
[APPENDIX.md](APPENDIX.md), [GLOSSARY.md](GLOSSARY.md), this
README, examples — do not require a version bump.

All PRs target `dev`; `master` represents released state and is
updated only by maintainers in batched releases.

---

## Credits

These specifications were produced as part of a documentation and
interoperability effort for OpenVoiceOS, funded by NLnet's
[NGI0 Commons Fund](https://nlnet.nl/project/OpenVoiceOS) under
grant agreement No
[101135429](https://cordis.europa.eu/project/id/101135429).

![NGI0 / NLnet](./ngi.png)
