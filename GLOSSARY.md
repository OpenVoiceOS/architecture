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
| **Template** | A string in the OVOS-INTENT-1 grammar describing a set of sentences ([INTENT-1 §3](intent-1.md)). |
| **Expansion** | Resolving `(a\|b)` / `[x]` into a finite set of concrete sentences ([INTENT-1 §4](intent-1.md)). |
| **Sample / sample set** | A concrete sentence produced by expansion; the set of all of them for a template ([INTENT-1 §4](intent-1.md)). |
| **Slot** | A named placeholder `{name}` filled with a value rather than written out ([INTENT-1 §3.4, §5](intent-1.md)). |
| **Slot map** | The names→values mapping a match produces — slot names or vocabulary names as keys ([INTENT-3 §7](intent-3.md)). |
| **Resource file / role** | A skill's plain-text files: `.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist` ([INTENT-2 §1](intent-2.md)). |
| **Vocabulary** | A named slot-free phrase set; the unit a keyword intent constrains over ([INTENT-3 §4.1](intent-3.md)). |
| **Occurrence** | A phrase appearing in an utterance as a contiguous whole-word subsequence ([INTENT-2 §4.3](intent-2.md), [INTENT-3 §4.1](intent-3.md)). |
| **Skill** | An app — a self-contained unit of assistant functionality ([INTENT-3 §1, §3](intent-3.md)). |
| **Skill id** | A skill's identifier, unique across the assistant ([INTENT-3 §3](intent-3.md)). |
| **Intent** | A developer-defined binding from a natural-language command to one handler ([INTENT-3 §1](intent-3.md)). |
| **Intent name / qualified name** | The intent's name, unique within its skill / the `skill_id:intent_name` pair ([INTENT-3 §3](intent-3.md)). |
| **Keyword intent / template intent** | The two definition methods — keyword constraints, or sentence templates ([INTENT-3 §2](intent-3.md)). |
| **Handler** | The code an intent triggers when its command is recognized ([INTENT-3 §1, §6](intent-3.md)). |
| **Intent engine** | A classifier + slot extractor: consumes definitions, identifies the triggered intent ([INTENT-3 §6.2](intent-3.md)). |
| **Orchestrator** | The component that coordinates intent matching and dispatch — owns the engines / pipeline plugins and routes match results to handlers ([INTENT-3 §6.1](intent-3.md)). Distinct from the messagebus (transport) and from individual engines / plugins. |
| **Registration** | Submitting an intent's definition and handler together, as one unit ([INTENT-3 §6.1](intent-3.md)). |
| **Message** | The unit of communication on the bus: a JSON object with `type`, `data`, `context` ([MSG-1 §2](msg-1.md)). |
| **Context** | The assistant-metadata object on a Message; an extensible JSON object whose keys are defined by companion specs ([MSG-1 §2.3](msg-1.md)). |
| **Session** | The per-conversation carrier in `context.session`; carries `session_id` (with `"default"` reserved for "interact with the device-local session") and `lang` (the user's preferred language, distinct from any `data.lang` describing the payload's own language) ([SESSION-1 §3.1, §3.2](session-1.md)). |
| **Listening lifecycle signal** | A payload-free bus signal the audio input service emits or consumes around voice-command capture and sleep mode — `ovos.listener.record.started` / `.record.ended`, `ovos.listener.sleep`, `ovos.listener.awoken` ([AUDIO-IN-1 §6](audio-in.md)). |
| **GUI template** | A member of the closed, curated `SYSTEM_*` vocabulary an application names to declare *what* to display; distinct from an INTENT-1 sentence template ([GUI-1 §3](gui-1.md)). |
| **Render backend / Adapter** | An additive plugin that turns GUI template intents into a concrete presentation (screen, browser, terminal, face); every installed adapter receives every event ([GUI-1 §6](gui-1.md)). |
| **GUI service** | The state-and-dispatch hub between applications and adapters: holds per-session display state, fans template events out to every adapter, runs the namespace lifecycle, and renders nothing itself ([GUI-1 §2.1](gui-1.md)). |
| **GUI namespace** | The opaque string (by convention the producing `skill_id`) that scopes a stack of display state within a session ([GUI-1 §2.2](gui-1.md)). |
| **Session data** | The flat key-value map describing a GUI template's content, accumulated per namespace and synced via `gui.value.set` ([GUI-1 §3.3](gui-1.md)). |
| **Common query** | A pipeline plugin that answers factual questions by holding a timed contest among skills — broadcast, collect competing answers, rank, speak the best ([COMMON-QUERY-1 §2](common-query.md)). |
| **Scatter-gather** | The contest pattern: one broadcast fans out to many skills (scatter), their answers are collected and ranked (gather) ([COMMON-QUERY-1 §2](common-query.md)). |
| **Wants-to-answer poll** | Common query's fast ping/pong phase — a cheap local filter where skills self-nominate before the expensive full-answer phase ([COMMON-QUERY-1 §6](common-query.md)). |
