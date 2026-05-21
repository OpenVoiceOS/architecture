# OVOS Formal Specifications

Formal, implementation-agnostic specifications for the OpenVoiceOS voice
assistant ecosystem. Each specification describes a format or contract
generically, so it can be implemented by any tool, in any language, and adopted
by voice assistants beyond OVOS.

> ⚠️ **Early draft — do not depend on these specs yet.** Every specification
> here is at **Draft** status. The grammar, file formats, and contracts are
> still under active review and may change in incompatible ways between
> versions. While you can see this notice, treat these documents as a request
> for comment, not a stable contract: do not build implementations that rely on
> them remaining unchanged. The notice will be removed when a spec reaches a
> stable status.

## Specifications

| ID | Document | Version | Status | Scope |
|----|----------|---------|--------|-------|
| OVOS-INTENT-1 | [Sentence Template Grammar](sentence-template-grammar.md) | 1.1 | Draft | The ASR input model, the sentence template grammar (expansion + named slots), expansion into training samples, the slot model, and the skill→pipeline training-data contract. |
| OVOS-INTENT-2 | [Locale Resource Formats](locale-resource-formats.md) | 1.1 | Draft | The `locale/` folder layout and the two resource file formats across five roles: `.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist`. |
| OVOS-INTENT-3 | [Intent Definition](intent-definition.md) | 1 | Draft | What an intent is, the two definition methods (keyword and template intents), registration, the intent-engine input contract, and the match result. |

**Reading order.** The specifications are numbered in dependency order and are
meant to be read that way. OVOS-INTENT-1 defines the template grammar;
OVOS-INTENT-2 builds on it to define the resource files; OVOS-INTENT-3 builds
on both to define what an intent is. Each depends on the ones before it.
OVOS-INTENT-3 version 1 builds on OVOS-INTENT-1 and OVOS-INTENT-2 at version
1.1.

## Design notes

- These specs define the **shape of training data and resource files**, not
  engine matching behaviour. A template generates training samples; a capable
  intent engine generalizes beyond them to recognize unseen utterances.
  Matching, scoring, and accept/reject decisions are intentionally left to each
  engine.
- This draft is deliberately **unopinionated about slot value types**. A slot
  value is an opaque sequence of words; numbers, dates, and other typed values
  depend on a prescribed normalization of ASR output, which is deferred to a
  future specification (see OVOS-INTENT-1 §5.3).
- OVOS-INTENT-3 adds the **intent model** on top: an intent is a
  developer-defined binding from a natural-language command to one handler, not
  a free-floating event. Every intent engine is conceptually a classifier plus
  a slot extractor, and the two definition methods — keyword intents and
  template intents — are non-interoperable but complementary.

Design rationale, comparisons with Home Assistant and Rhasspy, the pipeline
context, and known gaps are collected in [APPENDIX.md](APPENDIX.md) — a
non-normative companion document.

## Glossary

Terms defined across the three specifications, with where each is defined.

| Term | Meaning |
|------|---------|
| **Template** | A string in the OVOS-INTENT-1 grammar describing a set of sentences (INTENT-1 §3). |
| **Expansion** | Resolving `(a\|b)` / `[x]` into a finite set of concrete sentences (INTENT-1 §4). |
| **Sample / sample set** | A concrete sentence produced by expansion; the set of all of them for a template (INTENT-1 §4). |
| **Slot** | A named placeholder `{name}` filled with a value rather than written out (INTENT-1 §3.4, §5). |
| **Capture map** | The names→values mapping a match produces — slot names or vocabulary names as keys (INTENT-3 §7). |
| **Resource file / role** | A skill's plain-text files: `.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist` (INTENT-2 §1). |
| **Vocabulary** | A named slot-free phrase set; the unit a keyword intent constrains over (INTENT-3 §4.1). |
| **Occurrence** | A phrase appearing in an utterance as a contiguous whole-word subsequence (INTENT-2 §4.3, INTENT-3 §4.1). |
| **Skill** | An app — a self-contained unit of assistant functionality (INTENT-3 §1, §3). |
| **Skill id** | A skill's identifier, unique across the assistant (INTENT-3 §3). |
| **Intent** | A developer-defined binding from a natural-language command to one handler (INTENT-3 §1). |
| **Intent name / qualified name** | The intent's name, unique within its skill / the `skill_id:intent_name` pair (INTENT-3 §3). |
| **Keyword intent / template intent** | The two definition methods — keyword constraints, or sentence templates (INTENT-3 §2). |
| **Handler** | The code an intent triggers when its command is recognized (INTENT-3 §1, §6). |
| **Intent engine** | A classifier + slot extractor: consumes definitions, identifies the triggered intent (INTENT-3 §6.2). |
| **Host** | The intent system that owns the engines and routes match results to handlers (INTENT-3 §6.1). |
| **Registration** | Submitting an intent's definition and handler together, as one unit (INTENT-3 §6.1). |

## Planned

- Text normalization of ASR output (the basis for slot value typing).
- A machine-checkable conformance corpus for OVOS-INTENT-1 expansion.

## Changing a specification

Specifications are versioned documents, not living wikis. Any change to a spec —
however small — **MUST** be submitted as a pull request, never committed
directly. Each PR that alters normative content **MUST** bump the spec's
`Version` field in its header and add a corresponding entry to
[`CHANGELOG.md`](CHANGELOG.md). A version identifies an exact, citable state of
a document, so implementations and conformance results can name the version
they target.

## Implementations

These specifications are **prescriptive**: they define a target, not a
description of current software. No implementation fully conforms yet. The
projects below are the closest existing implementations and the basis these
specs were drawn from; bringing them into conformance is planned work.

- [padacioso](https://github.com/OpenVoiceOS/padacioso) — sentence template
  grammar and expansion.
- [ovos-workshop](https://github.com/OpenVoiceOS/ovos-workshop) — locale folder
  layout and resource file loaders.
