# Appendix — Design Notes and Context

**Non-normative.** This document is a companion to the OVOS formal
specifications. It records design rationale, comparisons with other
systems, the catalogue of *deliberate* divergences from current OVOS
code, and implementer-facing reference material that does not belong
in a normative specification body. Nothing here is binding — the
normative documents are OVOS-INTENT-1, OVOS-INTENT-2,
OVOS-INTENT-3, OVOS-INTENT-4, OVOS-MSG-1, OVOS-SESSION-1,
OVOS-PIPELINE-1, OVOS-CONTEXT-1, OVOS-CONVERSE-1, OVOS-STOP-1,
OVOS-COMMON-QUERY-1, and OVOS-TRANSFORM-1.

Pointers to specific OVOS code (file paths, class names, function
names) and to specific real projects (HiveMind, Adapt,
padatious, ovos-audio, ovos-workshop, …) are deliberately kept
*out* of the spec bodies and collected here, because implementation
code moves and specifications must not.

> **⚠️ AI-generated draft — not yet fully reviewed.** The appendix
> content was produced by a large language model (Claude Code) and
> has not yet been fully reviewed for accuracy, completeness, or
> consistency with the specifications. The normative specifications
> themselves are human-reviewed; this appendix is supplementary
> context. Readers should verify claims before relying on them.

The appendix content has been split into topic-specific files:

| File | Section | Topic |
|------|---------|-------|
| [appendix/overview.md](appendix/overview.md) | §1 | About the OVOS specifications — voice OS concept, formalization, stack taxonomy, compatibility, reference implementations |
| [appendix/comparisons.md](appendix/comparisons.md) | §2 | Comparison with other voice-assistant systems — HA/Rhasspy, Rasa, ASK/Dialogflow, summary |
| [appendix/patterns.md](appendix/patterns.md) | §3 | Architectural patterns — bus substrate, pipeline-plugin model, interop with external protocols |
| [appendix/rationale.md](appendix/rationale.md) | §4 | Design rationale, per specification — why each spec makes its choices |
| [appendix/divergences.md](appendix/divergences.md) | §5 | Where the specs differ from current OVOS code — divergences, renames, topic mapping |
| [appendix/reference.md](appendix/reference.md) | §6 | Implementer reference — session-field cheat-sheet, stamp rules, introspection patterns |
| [appendix/gaps.md](appendix/gaps.md) | §7 | Known gaps and planned work — deferred specs, tooling, corpora |
| [appendix/persona-flow.md](appendix/persona-flow.md) | §8 | Persona lifecycle — annotated bus sequences for summon, conversation, dismiss, and out-of-band query |
