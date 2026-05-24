# Glossary

Terms defined across the OVOS specifications, with where each is
defined. This document is **non-normative** — each term's
authoritative definition lives in the spec section linked from its
entry. The glossary exists so a reader who encounters a term in
one spec can find where it was introduced without grepping the
whole repository.

If a term used in a spec is missing here, that's a bug — please
open a PR adding it.

---

## Terms

| Term | Meaning |
|------|---------|
| **Template** | A string in the OVOS-INTENT-1 grammar describing a set of sentences ([INTENT-1 §3](sentence-template-grammar.md)). |
| **Expansion** | Resolving `(a\|b)` / `[x]` into a finite set of concrete sentences ([INTENT-1 §4](sentence-template-grammar.md)). |
| **Sample / sample set** | A concrete sentence produced by expansion; the set of all of them for a template ([INTENT-1 §4](sentence-template-grammar.md)). |
| **Slot** | A named placeholder `{name}` filled with a value rather than written out ([INTENT-1 §3.4, §5](sentence-template-grammar.md)). |
| **Capture map** | The names→values mapping a match produces — slot names or vocabulary names as keys ([INTENT-3 §7](intent-definition.md)). |
| **Resource file / role** | A skill's plain-text files: `.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist` ([INTENT-2 §1](locale-resource-formats.md)). |
| **Vocabulary** | A named slot-free phrase set; the unit a keyword intent constrains over ([INTENT-3 §4.1](intent-definition.md)). |
| **Occurrence** | A phrase appearing in an utterance as a contiguous whole-word subsequence ([INTENT-2 §4.3](locale-resource-formats.md), [INTENT-3 §4.1](intent-definition.md)). |
| **Skill** | An app — a self-contained unit of assistant functionality ([INTENT-3 §1, §3](intent-definition.md)). |
| **Skill id** | A skill's identifier, unique across the assistant ([INTENT-3 §3](intent-definition.md)). |
| **Intent** | A developer-defined binding from a natural-language command to one handler ([INTENT-3 §1](intent-definition.md)). |
| **Intent name / qualified name** | The intent's name, unique within its skill / the `skill_id:intent_name` pair ([INTENT-3 §3](intent-definition.md)). |
| **Keyword intent / template intent** | The two definition methods — keyword constraints, or sentence templates ([INTENT-3 §2](intent-definition.md)). |
| **Handler** | The code an intent triggers when its command is recognized ([INTENT-3 §1, §6](intent-definition.md)). |
| **Intent engine** | A classifier + slot extractor: consumes definitions, identifies the triggered intent ([INTENT-3 §6.2](intent-definition.md)). |
| **Orchestrator** | The component that coordinates intent matching and dispatch — owns the engines / pipeline plugins and routes match results to handlers ([INTENT-3 §6.1](intent-definition.md)). Distinct from the messagebus (transport) and from individual engines / plugins. |
| **Registration** | Submitting an intent's definition and handler together, as one unit ([INTENT-3 §6.1](intent-definition.md)). |
| **Message** | The unit of communication on the bus: a JSON object with `type`, `data`, `context` ([MSG-1 §2](message-object.md)). |
| **Context** | The assistant-metadata object on a Message; an extensible JSON object whose keys are defined by companion specs ([MSG-1 §2.3](message-object.md)). |
| **Session** | The per-conversation carrier in `context.session`; carries `session_id` (with `"default"` reserved for "originates from the device itself") and `lang` (the user's preferred language, distinct from any `data.lang` describing the payload's own language) ([MSG-1 §4](message-object.md)). |
| **Registration message** | A bus message that broadcasts a keyword intent, template intent, or entity for any pipeline plugin to consume ([INTENT-4 §§5–7](intent-registration.md)). |
| **Pipeline plugin** | A black-box component identified by an opaque `pipeline_id` that consumes utterances (and optionally registration messages) and returns intent matches. Loaded by the orchestrator at startup; ordered list lives in `session.pipeline` (OVOS-PIPELINE-1). |
| **Dispatch message** | A Message on the per-intent topic `<owner_id>:<intent_name>` — the orchestrator's instruction to a skill (when `owner_id` is a `skill_id`) or to a pipeline plugin (when `owner_id` is a `pipeline_id`) to run its handler for a matched intent (OVOS-PIPELINE-1). |
| **Match-result message** | `ovos.intent.matched` — the orchestrator's notification that an intent matched an utterance (OVOS-PIPELINE-1). |
