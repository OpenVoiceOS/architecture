# OVOS Formal Specifications

Formal, implementation-agnostic specifications for the OpenVoiceOS voice
assistant ecosystem. Each specification describes a format or contract
generically, so it can be implemented by any tool, in any language, and adopted
by voice assistants beyond OVOS.

## Specifications

| ID | Document | Version | Status | Scope |
|----|----------|---------|--------|-------|
| OVOS-INTENT-1 | [Sentence Template Grammar](sentence-template-grammar.md) | 1 | Draft | The ASR input model, the sentence template grammar (expansion + named slots), expansion into training samples, the slot model, and the skill→pipeline training-data contract. |
| OVOS-INTENT-2 | [Locale Resource Formats](locale-resource-formats.md) | 1 | Draft | The `locale/` folder layout and the two resource file formats across five roles: `.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist`. |

OVOS-INTENT-2 **depends on** OVOS-INTENT-1: every resource file is a list of
sentence templates whose syntax and expansion are defined by the grammar spec.
Read OVOS-INTENT-1 first.

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

## Planned

- Text normalization of ASR output (the basis for slot value typing).
- The skill manifest (metadata, intent→handler binding, language fallback).
- A machine-checkable conformance corpus for OVOS-INTENT-1 expansion.

## Reference implementations

- [padacioso](https://github.com/OpenVoiceOS/padacioso) — sentence template
  grammar and expansion.
- [ovos-workshop](https://github.com/OpenVoiceOS/ovos-workshop) — locale folder
  layout and resource file loaders.
