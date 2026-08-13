# Voice Operating System Specifications

Formal, implementation-agnostic specifications for a **voice
operating system** — a platform that provides a stable application
binary interface for voice-interactive applications.

This repository is the **source of truth** for how components
talk to each other and what their data shapes mean. The specs are
written generically so they can be implemented by any tool, in any
language, and adopted by any voice-assistant project.

### What a voice operating system is

A voice OS is not a voice assistant. A voice assistant is a product
that answers questions. A voice OS is a **platform**: it defines the
boundary between user input and computation, arbitrates which
application handles each utterance, manages conversation state across
interactions, and provides a stable ABI that arbitrary third-party
applications run against without knowing anything about each other.

The analogy to a general-purpose OS is direct:

| OS concept | Voice OS equivalent |
|---|---|
| Process scheduler | Pipeline plugin ordering (PIPELINE-1 §5–6) |
| IPC / message passing | The bus and MSG-1 envelope |
| Shared memory | Session carrier (SESSION-1, SESSION-2) |
| Process supervision | Handler-lifecycle trio (PIPELINE-1 §8) |
| Loadable kernel modules | Pipeline plugins, transformer plugins |
| System call ABI | The `match(utterances, lang, session) → Match` contract |

The consequence is that this corpus does not describe a chatbot, an
LLM wrapper, or a monolithic product. It describes a **runtime**: swap
the scheduler (pipeline ordering), the NLU engines (pipeline plugins),
the dialogue policy (converse / context), the output layer (TTS,
display), or any combination — the ABI stays stable and the rest
keeps working. A skill written against the intent stack runs on any
conformant orchestrator, under any pipeline configuration, in any
language a deployment supports.

> **Draft status.** Every spec in this repository is at **Draft**
> status (the Status column below). A Draft spec is prescriptive:
> where an implementation diverges from it, the divergence is an
> implementation bug, not a defect in the specification (see
> *Authority* below).

---

## Goals

The specs exist to make three things possible:

- **Interoperability.** Multiple implementations — engines, hosts,
  plugins, entire assistants — can target the same observable
  contract instead of reverse-engineering each other's code.
- **Stability.** Implementation churn no longer drifts the contract.
  Each spec is a versioned document; behaviour changes go through
  a pull request with a version bump.
- **Portability.** The specs are written implementation-agnostically
  so any voice-assistant project can adopt the same formats, grammar,
  and bus contracts, independent of any one codebase.

The specs cover formats and contracts only. They do not mandate
implementation choices — programming language, internal design,
storage, threading, transport — those are the implementer's. What
they fix is the **observable contract**.

---

## Authority

These specifications are **prescriptive, not descriptive**. They
define the intended architecture; they are not a transcript of how
any current implementation behaves. Where an implementation diverges
from a spec here, that divergence is a **bug in the implementation**,
not in the specification.

Anyone is free to **adopt** these specifications and free to
**propose changes** to them via pull request (see [contributing]
below). Adoption is voluntary; conformance, once adopted, is not.

[contributing]: #contributing

---

## Specifications

The **Version** column carries the specification's compatibility class
([VERSIONING.md](VERSIONING.md); full policy in [appendix/versioning.md](appendix/versioning.md)).

### Intent stack — what a skill defines

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-INTENT-1 | [Sentence Template Grammar](intent-1.md) | 2 | Draft |
| OVOS-INTENT-2 | [Locale Resource Formats](intent-2.md) | 2 | Draft |
| OVOS-INTENT-3 | [Intent Definition](intent-3.md) | 1 | Draft |
| OVOS-INTENT-4 | [Intent and Entity Registration Bus Contract](intent-4.md) | 2 | Draft |

### Bus stack — how components talk

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-MSG-1 | [Bus Message](msg-1.md) | 1 | Draft |
| OVOS-SESSION-1 | [Session](session-1.md) | 1 | Draft |
| OVOS-SESSION-2 | [Session Lifecycle and State Ownership](session-2.md) | 1 | Draft |
| OVOS-BRIDGE-1 | [Bus Bridge and Opaque Relay](bridge-1.md) | 2 | Draft |

### Orchestrator stack — what processes utterances

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-PIPELINE-1 | [Utterance Lifecycle and Pipeline](pipeline-1.md) | 2 | Draft |
| OVOS-TRANSFORM-1 | [Transformer Plugins](transformer.md) | 1 | Draft |
| OVOS-CONTEXT-1 | [Intent Context](intent-context.md) | 2 | Draft |
| OVOS-CONVERSE-1 | [Active Handlers and Interactive Response](converse.md) | 2 | Draft |
| OVOS-STOP-1 | [Stop Pipeline Plugin](stop-1.md) | 2 | Draft |
| OVOS-PERSONA-1 | [Persona Pipeline Plugin](persona.md) | 2 | Draft |
| OVOS-FALLBACK-1 | [Fallback Pipeline Plugin](fallback.md) | 2 | Draft |
| OVOS-COMMON-QUERY-1 | [Common Query Pipeline Plugin](common-query.md) | 2 | Draft |

### I/O stack — input and output surfaces

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-AUDIO-IN-1 | [Audio Input Service](audio-in.md) | 2 | Draft |
| OVOS-AUDIO-1 | [Audio Output Service](audio-out.md) | 2 | Draft |
| OVOS-GUI-1 | [GUI Display Subsystem](gui-1.md) | 1 | Draft |

### Media stack — playback and transport

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-OCP-1 | [Common Playback: the Virtual Media Player](ocp-1.md) | 1 | Draft |

Each spec carries its own scope statement, design rationale, and
conformance section in its header. Open the document for the full
picture — the tables above are an index.

**Reading order by role:**
- *Writing a skill?* INTENT-1 → INTENT-2 → INTENT-3. INTENT-4 only if you need the registration wire format.
- *Building a pipeline plugin?* PIPELINE-1, then SESSION-1 + SESSION-2, then the role spec (CONVERSE-1, CONTEXT-1, TRANSFORM-1, or STOP-1).
- *Building an orchestrator?* MSG-1 → SESSION-1 → SESSION-2 → PIPELINE-1, then INTENT-4, CONTEXT-1, CONVERSE-1, TRANSFORM-1, STOP-1.
- *Surveying the architecture?* [appendix/overview.md §1](appendix/overview.md) for the three-stack narrative.

For background — design rationale, comparisons with other systems,
implementation pointers, the catalogue of known divergences, and known
gaps — see [APPENDIX.md](APPENDIX.md) (index) or browse by topic under [appendix/](appendix/). For term definitions, see
[GLOSSARY.md](GLOSSARY.md). For the version history of each spec,
see [CHANGELOG.md](CHANGELOG.md).

---

## Contributing

Specifications are **versioned documents, not living wikis**. Any
change to a spec — however small — **MUST** be submitted as a pull
request, never committed directly.

Each PR that alters normative content **MUST**:

- add a corresponding entry to [CHANGELOG.md](CHANGELOG.md);
- set the spec's `Version` field to its compatibility class — the field is a
  class, not a per-revision counter (VERSIONING.md).

PRs that touch only the non-normative material —
[APPENDIX.md](APPENDIX.md) and [appendix/](appendix/) files,
[GLOSSARY.md](GLOSSARY.md), this README, examples — do not
require a version bump.

For the reference implementation, ecosystem tooling, and who this
corpus is produced for, see
[appendix/overview.md §1.4–1.5](appendix/overview.md).

---

## Credits

[![NGI0 Commons Fund](./ngi.png)](https://nlnet.nl/project/OpenVoiceOS)

This project was funded through the [NGI0 Commons Fund](https://nlnet.nl/commonsfund),
a fund established by [NLnet](https://nlnet.nl) with financial support from the
European Commission's [Next Generation Internet](https://ngi.eu) programme, under
the aegis of [DG Communications Networks, Content and Technology](https://commission.europa.eu/about-european-commission/departments-and-executive-agencies/communications-networks-content-and-technology_en)
under grant agreement No [101135429](https://cordis.europa.eu/project/id/101135429).
