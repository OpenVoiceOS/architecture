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
> the specification (see *Authority* below). The notice will be
> removed when a spec reaches a stable status.

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

## Specifications

| ID | Document | Version | Status |
|----|----------|---------|--------|
| OVOS-INTENT-1 | [Sentence Template Grammar](sentence-template-grammar.md) | 2 | Draft |
| OVOS-INTENT-2 | [Locale Resource Formats](locale-resource-formats.md) | 2 | Draft |
| OVOS-INTENT-3 | [Intent Definition](intent-definition.md) | 1 | Draft |
| OVOS-MSG-1 | [Bus Message](message-object.md) | 1 | Draft |
| OVOS-INTENT-4 | [Intent and Entity Registration and Dispatch](intent-registration.md) | 1 | Draft |

Each spec carries its own scope statement, design rationale, and
conformance section in its own header. Open the document for the
full picture — the table above is just an index.

**Reading order.** The intent specs are numbered in dependency
order: OVOS-INTENT-1 defines the template grammar; OVOS-INTENT-2
builds on it to define the resource files; OVOS-INTENT-3 builds on
both to define what an intent is. OVOS-MSG-1 is the bus-layer
envelope and the routing / session model — readable at any point.
OVOS-INTENT-4 builds on both stacks (INTENT-3 + MSG-1) and is read
last.

For background — design rationale, comparisons with other systems,
the catalogue of known divergences from current code, and known
gaps — see [APPENDIX.md](APPENDIX.md). For term definitions, see
[GLOSSARY.md](GLOSSARY.md). For the version history of each spec,
see [CHANGELOG.md](CHANGELOG.md).

---

## Reference implementation

[**ovos-spec-tools**](https://github.com/OpenVoiceOS/ovos-spec-tools)
is a reference implementation — a dependency-light Python package
providing the sentence-template expander, the locale resource
loader, the dialog renderer, language matching, and the
`ovos-spec-lint` linter. Components that don't want to reimplement
the spec machinery themselves can depend on it. It is also the
intended home of the planned conformance corpus.

The bus stack (OVOS-MSG-1) does not yet have a comparable
ground-up reference implementation; `ovos-bus-client` is the
closest existing match but predates the spec.

---

## Contributing

Specifications are **versioned documents, not living wikis**. Any
change to a spec — however small — **MUST** be submitted as a pull
request, never committed directly.

Each PR that alters normative content **MUST**:

- bump the spec's `Version` field in its header;
- add a corresponding entry to [CHANGELOG.md](CHANGELOG.md).

A version identifies an exact, citable state of a document, so
implementations and conformance results can name the version they
target.

PRs that touch only the non-normative material —
[APPENDIX.md](APPENDIX.md), [GLOSSARY.md](GLOSSARY.md), this
README, examples — do not require a version bump.

---

## Credits

These specifications were produced as part of a documentation and
interoperability effort for OpenVoiceOS, funded by NLnet's
[NGI0 Commons Fund](https://nlnet.nl/project/OpenVoiceOS) under
grant agreement No
[101135429](https://cordis.europa.eu/project/id/101135429).

![NGI0 / NLnet](./ngi.png)
