---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

> **⚠️ AI-generated draft — not yet fully reviewed.** This content
> was produced by a large language model (Claude Code) and
> has not yet been fully reviewed for accuracy, completeness, or
> consistency with the specifications. The normative specifications
> themselves are human-reviewed; this appendix is supplementary
> context. Readers should verify claims before relying on them.

## 7. Known gaps and planned work

- **Per-plugin behavioural specs.** OVOS-PIPELINE-1 defines
  the plugin contract (the `match` shape, the orchestrator's
  iteration semantics) but explicitly defers what each
  non-trivial plugin type actually *does*. `converse` (OVOS-CONVERSE-1),
  `stop` (OVOS-STOP-1), and `common_query` (OVOS-COMMON-QUERY-1)
  have their own specs. Remaining candidates: `fallback`, `ocp`,
  `persona`. Each defines its own internal behaviour and its
  own bus emissions beyond the universal lifecycle PIPELINE-1
  prescribes.
- **Session preference fields not yet claimed.** SESSION-1
  defines the wire shape and OVOS-SESSION-2 (in flight at
  PR #27) defines the lifecycle and state-ownership model;
  what remains deferred is the full set of session
  preferences current OVOS already carries (`persona_id`,
  `time_format`, `date_format`, `system_unit`,
  `tts_preferences`, `location`, …) — these need to be
  claimed under SESSION-1 §2.1's field registry by their
  respective owning specs (a future preferences spec,
  OCP / persona / locale specs as appropriate).
- **Text normalization of ASR output.** The basis for slot
  value typing (INTENT-1 §5.3). Deferred to its own
  specification.
- **A machine-checkable conformance corpus** of `template →
  sample set` pairs for INTENT-1 expansion, so expander
  conformance can be verified automatically. A parallel
  corpus of bus-message fixtures for MSG-1 would be the
  equivalent at the bus layer.
- **An end-to-end worked example.** The specs have local
  examples; none shows a single skill defining one keyword
  intent and one template intent through the whole path —
  files, registration, match, handler.
- **Conversation-level evaluation infrastructure.** Rasa
  has story-based testing and end-to-end success metrics;
  the OVOS specs do not currently have a counterpart.
- **OVOS-INTENT-2 ↔ hassil `intents` translation tool.**
  The grammar lineage (§2.1) makes a mechanical translator
  between OVOS-INTENT-2 locale resources and HA's `intents`
  YAML feasible. Such a tool would let the two corpora
  cross-pollinate without either format changing. Sits at
  injection point 3 of §3.3 conceptually but is
  build-time rather than runtime tooling.
- **i18n corpus.** OVOS-INTENT-2 defines the locale file
  format, and `ovos-localize` (§1.4) provides the
  operations layer; what remains is the *scale* of the
  translated corpus.
