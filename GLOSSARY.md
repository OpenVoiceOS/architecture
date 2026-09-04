# Glossary

Terms defined across the specifications, with where each is
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
| **Slot map** | The names→values mapping a match produces — slot names or vocabulary names as keys ([PIPELINE-1 §4.3](pipeline-1.md)). |
| **Resource file / role** | A skill's plain-text files: `.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist`, `.prompt` ([INTENT-2 §1](intent-2.md)). |
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
| **Session** | The per-conversation carrier in `context.session` ([MSG-1 §4](msg-1.md)); its claimable field set — `session_id`, `lang`, and every other field a specification registers — is owned by [SESSION-1 §2.2/§3](session-1.md). |
| **Dispatch-shaped topic** | A topic containing `:`, assembled from identifiers to address a specific registered handler — canonically `<skill_id>:<intent_name>`; the `:` is the marker, and only a specification in this family may define a colon-bearing shape ([MSG-1 §2.1.1](msg-1.md)). |
| **Dotted addressed topic** | An ordinary `:`-free dotted topic (`<x>.<y>.<verb>`) that names a specific recipient in one of its segments — e.g. `<skill_id>.common_query.request` — an addressed message, not a dispatch ([MSG-1 §2.1.1](msg-1.md), [COMMON-QUERY-1 §7](common-query.md)). |
| **Recency-targeted stop** | See [STOP-1 §4.1](stop-1.md) — the stop plugin's no-answer fallback. |
| **Listening lifecycle signal** | A payload-free bus signal the audio input service emits or consumes around voice-command capture and sleep mode — `ovos.listener.wakeword`, `ovos.listener.record.started` / `.record.ended`, `ovos.listener.sleep`, `ovos.listener.awoken` ([AUDIO-IN-1 §6](audio-in.md)). |
| **GUI template** | A member of the closed, curated `SYSTEM_*` vocabulary an application names to declare *what* to display; distinct from an INTENT-1 sentence template ([GUI-1 §3](gui-1.md)). |
| **Render backend / Adapter** | An additive plugin that turns GUI template intents into a concrete presentation (screen, browser, terminal, face); every installed adapter receives every event ([GUI-1 §6](gui-1.md)). |
| **GUI service** | The state-and-dispatch hub between applications and adapters: holds per-session display state, fans template events out to every adapter, runs the namespace lifecycle, and renders nothing itself ([GUI-1 §2.1](gui-1.md)). |
| **GUI namespace** | The opaque string (by convention the producing `skill_id`) that scopes a stack of display state within a session ([GUI-1 §2.2](gui-1.md)). |
| **Session data** | The flat key-value map describing a GUI template's content, accumulated per namespace and synced via `gui.value.set` ([GUI-1 §3.3](gui-1.md)). |
| **Common query** | A pipeline plugin that answers factual questions by holding a timed contest among skills — broadcast, collect competing answers, rank, speak the best ([COMMON-QUERY-1 §2](common-query.md)). |
| **Scatter-gather** | The contest pattern: one broadcast fans out to many skills (scatter), their answers are collected and ranked (gather) ([COMMON-QUERY-1 §2](common-query.md)). |
| **Wants-to-answer poll** | Common query's fast ping/pong phase — a cheap local filter where skills self-nominate before the expensive full-answer phase ([COMMON-QUERY-1 §6](common-query.md)). |
| **Manifest** | The orchestrator-owned index of every registration it has observed, served on query so a late-subscribing consumer isn't limited to what it caught live ([INTENT-4 §10](intent-4.md)). |
| **Registration key** | The tuple identifying one registration for replacement/indexing purposes — `(skill_id, intent_name, lang, method)`, extended to `(session_id, skill_id, intent_name, lang, method)` under session-scoped registration ([INTENT-4 §3.2, §11.1](intent-4.md)). |
| **Effective intent pool** | The set of intents available to a session: everything registered under `"default"` plus everything registered under that session's own `session_id`, minus blacklisted entries ([INTENT-4 §11.2](intent-4.md)). |
| **Session-scoped registration** | Keying an INTENT-4 registration by the producer's `session_id` (from `context`, never `data`) rather than always writing to `"default"` ([INTENT-4 §11](intent-4.md)). |
| **Vocabulary descriptor** | The JSON object naming one vocabulary and its samples in an INTENT-4 registration payload ([INTENT-4 §5.1](intent-4.md)). |
| **Effective handler pool** | The fallback plugin's ordered candidate list for a `match` call, built from registered skills filtered by session preference, stage range, availability, and policy ([FALLBACK-1 §5](fallback.md)). |
| **Fallback skill** | A skill that declares no intent patterns and instead evaluates the raw utterance itself when polled by a fallback pipeline plugin ([FALLBACK-1 §2](fallback.md)). |
| **Fallback pipeline plugin** | A pipeline plugin that maintains a registry of fallback skills and queries them in order until one claims the utterance ([FALLBACK-1 §2](fallback.md)). |
| **Match** | The object a pipeline plugin's `match` function returns to claim an utterance — `skill_id`, `intent_name`, `lang`, `slots`, `utterance`, and optionally `updated_session` ([PIPELINE-1 §4.1](pipeline-1.md)). |
| **Pipeline plugin** | A component occupying a stage in `session.pipeline`, identified by an opaque `pipeline_id`, that may claim an utterance via `match` ([PIPELINE-1 §3](pipeline-1.md)). |
| **`pipeline_id`** | The opaque string matching `[A-Za-z0-9_-]` — no `:`, no `.` — that keys a pipeline plugin instance in the orchestrator's loaded-plugin set ([PIPELINE-1 §3](pipeline-1.md)). |
| **Transformer** | A black-box component that consumes one artifact at a fixed point in the utterance lifecycle and produces an artifact of the same shape for the next stage ([TRANSFORM-1 §1](transformer.md)). |
| **Transformer chain** | An ordered set of transformers of one type that all run, unconditionally, when their injection point is reached — no claim, no first-result-wins ([TRANSFORM-1 §1](transformer.md)). |
| **Injection point** | One of the six fixed places in the utterance lifecycle where a transformer chain runs ([TRANSFORM-1 §2](transformer.md)). |
| **Layer-2 system** | Authentication, authorization, multi-tenant routing, or remote participation built on top of `source`/`destination` opacity, without the assistant core learning about peers ([MSG-1 §3.4](msg-1.md)). |
| **Assistant core** | The side of the `source`/`destination` boundary opposite third-party handler code and other external bus participants ([MSG-1 §3](msg-1.md)). |
| **Derivation (`forward` / `reply` / `response`)** | The two normative operations — `forward` and `reply` — that produce a new Message from an existing one while propagating or rewriting its routing keys and session carrier, plus `response`, a shorthand naming convention layered on `reply` rather than a third derivation ([MSG-1 §5](msg-1.md)). |
| **Dispatch topic** | A colon-bearing topic `<skill_id>:<intent_name>` reserved by the colon-vs-dot convention for addressing a specific registered handler ([MSG-1 §2.1.1](msg-1.md)). |
| **Persona** | A complete conversational agent — its own identity (`persona_id`), personality, and capabilities — that a persona pipeline plugin hosts ([PERSONA-1 §2](persona.md)). |
| **Summon** | Activating a persona for a session by setting `persona_id`, whether by self-summon, one-off query, or an external component ([PERSONA-1 §5](persona.md)). |
| **Dismiss** | Deactivating the active persona for a session by clearing `persona_id`, returning the pipeline to no-persona mode ([PERSONA-1 §6](persona.md)). |
| **No-persona mode** | The pipeline state with no active persona (`persona_id` absent); only deterministic intent-matching and fallback stages handle utterances ([PERSONA-1 §4](persona.md)). |
| **Persona-fallback** | A persona plugin's secondary `fallback_pipeline_id` stage, positioned after all skill stages, that claims utterances when no persona is active and nothing else matched ([PERSONA-1 §7.1, §9](persona.md)). |
| **`persona_id`** | The session field naming the active persona; absent or empty means no persona is active ([PERSONA-1 §3](persona.md)). |
| **Virtual Media Player** | The single addressable, per-session arbitration point that owns the now-playing track, playback queue, and transport state, regardless of which backend does the work ([OCP-1 §2](ocp-1.md)). |
| **Media entry** | The object playback requests and state consumers exchange to describe one track — `uri`, `title`, `artist`, `status`, and related fields ([OCP-1 §4.5](ocp-1.md)). |
| **`PlayerState`** | The transport axis of the Virtual Media Player — `STOPPED` / `PLAYING` / `PAUSED` ([OCP-1 §3.1](ocp-1.md)). |
| **`MediaState`** | The loaded-media axis, independent of transport — `NO_MEDIA`, `LOADING`, `LOADED`, `BUFFERING`, `END_OF_MEDIA`, `INVALID_MEDIA` ([OCP-1 §3.2](ocp-1.md)). |
| **`transformer_id`** | The opaque, deployment-unique string identifying a transformer instance within its type in the orchestrator's per-type registry ([TRANSFORM-1 §1.1](transformer.md)). |
| **Utterance cancellation** | The transformer-plugin contract for aborting an in-flight utterance from within a chain by setting the `canceled` / `cancel_reason` context keys — the only sanctioned way to abort a lifecycle mid-flight ([TRANSFORM-1 §8](transformer.md)). |
| **Scheduler** | The single service that keeps schedules and fires their events ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Owner** | The component (skill, plugin, service) that created a schedule, identified by its component id ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Schedule** | One record: an owner, a target event, a timing rule, a payload, and policies ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Occurrence (scheduler)** | One instant at which a schedule is due ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Fire** | Emitting the target event for one occurrence ([SCHEDULER-1 §2](scheduler-1.md)). |
| **One-shot** | A schedule with exactly one occurrence ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Recurring** | A schedule whose occurrences are generated by a recurrence rule ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Misfire** | An occurrence whose fire happens after the due instant plus the grace period, or not at all ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Replay** | The scheduler's evaluation of every stored schedule after it starts ([SCHEDULER-1 §2](scheduler-1.md)). |
| **Ephemeral (schedule)** | A schedule that is never persisted and does not survive a scheduler restart ([SCHEDULER-1 §2](scheduler-1.md)). |
