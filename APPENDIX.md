# Appendix — Design Notes and Context

**Non-normative.** This document is a companion to the formal
specifications. It records design rationale, comparisons with other
systems, the catalogue of *deliberate* divergences from the
reference implementation, and implementer-facing reference material that does not belong
in a normative specification body. Nothing here is binding — the
normative documents are the specifications listed in the
[README spec registry](README.md#specifications).

Pointers to specific reference-implementation code (file paths,
class names, function names) and to specific real projects are
deliberately kept *out* of the spec bodies and collected here,
because implementation code moves and specifications must not.

The appendix is organized into topic-specific files:

| File | Section | Topic |
|------|---------|-------|
| [appendix/overview.md](appendix/overview.md) | §1 | About the specifications — voice OS concept, formalization, stack taxonomy, compatibility, reference implementations |
| [appendix/comparisons.md](appendix/comparisons.md) | §2 | Comparison with other voice-assistant systems — HA/Rhasspy, Rasa, ASK/Dialogflow, summary |
| [appendix/patterns.md](appendix/patterns.md) | §3 | Architectural patterns — bus substrate, pipeline-plugin model, interop with external protocols |
| [appendix/rationale.md](appendix/rationale.md) | §4 | Design rationale, per specification — why each spec makes its choices |
| [appendix/divergences.md](appendix/divergences.md) | §5 | Where the specs differ from the reference implementation — divergences, renames, topic mapping |
| [appendix/reference.md](appendix/reference.md) | §6 | Implementer reference — session-field cheat-sheet, stamp rules, introspection patterns |
| [appendix/gaps.md](appendix/gaps.md) | §7 | Known gaps — deferred specs, tooling, corpora |
| [appendix/persona-flow.md](appendix/persona-flow.md) | §8 | Persona lifecycle — annotated bus sequences for summon, conversation, dismiss, and out-of-band query |
| [appendix/versioning.md](appendix/versioning.md) | §9 | Spec versioning policy — the V0/V1/V2 compatibility classes and the 1.0 release criterion |
