---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

> **⚠️ AI-generated draft — not yet fully reviewed.** This content
> was produced by a large language model (Claude Code) and
> has not yet been fully reviewed for accuracy, completeness, or
> consistency with the specifications. The normative specifications
> themselves are human-reviewed; this appendix is supplementary
> context. Readers should verify claims before relying on them.

# Persona lifecycle — annotated bus sequences

This section shows the full observable bus sequence for the three
main persona lifecycle events: summon, a two-turn conversation, and
dismiss. All events are on the shared messagebus unless noted.
Session state changes are shown inline.

---

## Summon via utterance ("hey alice")

```
ovos.utterance.handle            [utterances=["hey alice"], session={persona_id: absent}]
  │
  ├─ pipeline: stop_high → None
  ├─ pipeline: converse  → None  (no active handler)
  ├─ pipeline: skill_high → None
  ├─ pipeline: persona   → Match (route 1: embedded summon command detected)
  │    Match.updated_session = {persona_id: "alice"}
  │
  ovos.intent.matched
  <pipeline_id>:persona           [dispatch; session now has persona_id="alice"]
    ovos.intent.handler.start
    ovos.utterance.speak          ["Sure, I'm Alice. How can I help?"]
    ovos.persona.activated        [persona_id="alice", session_id="..."]
    ovos.intent.handler.complete
  ovos.utterance.handled
```

Session state after: `{persona_id: "alice"}` — carried on all
subsequent utterances in this session.

---

## Active-persona conversation turn

```
ovos.utterance.handle            [utterances=["what's the weather?"], session={persona_id: "alice"}]
  │
  ├─ pipeline: stop_high → None
  ├─ pipeline: converse  → None  (no active response-mode)
  ├─ pipeline: skill_high → None  (or may match — skills run first)
  ├─ pipeline: persona   → Match (route 2: persona_id="alice" supported)
  │
  ovos.intent.matched
  <pipeline_id>:persona           [dispatch]
    ovos.intent.handler.start
    ovos.utterance.speak          ["It's 18 degrees and sunny."]
    ovos.intent.handler.complete
  ovos.utterance.handled
```

---

## Multi-turn (persona asks a follow-up question)

```
ovos.utterance.handle            [utterances=["tell me a story"], session={persona_id: "alice"}]
  │
  ├─ pipeline: persona → Match (route 2)
  │
  <pipeline_id>:persona
    ovos.intent.handler.start
    ovos.utterance.speak          ["What kind of story? (listen: true)"]
                                  ← listen:true re-opens mic after TTS
    [handler sets session.response_mode = {skill_id: pipeline_id, expires_at: ...}]
    ovos.intent.handler.complete
  ovos.utterance.handled

  [user speaks reply]

ovos.utterance.handle            [utterances=["a dragon story"], session={persona_id: "alice", response_mode: {...}}]
  │
  ├─ pipeline: converse → Match (response_mode held by persona plugin)
  │    dispatches <pipeline_id>:response
  │
  <pipeline_id>:response
    ovos.intent.handler.start
    ovos.utterance.speak          ["Once upon a time, a dragon..."]
    [handler clears session.response_mode]
    ovos.intent.handler.complete
  ovos.utterance.handled
```

---

## Dismiss via self-release ("goodbye alice")

```
ovos.utterance.handle            [utterances=["goodbye alice"], session={persona_id: "alice"}]
  │
  ├─ pipeline: persona → Match (route 1: release command detected)
  │    Match.updated_session = {persona_id: absent}
  │
  <pipeline_id>:persona
    ovos.intent.handler.start
    ovos.utterance.speak          ["Goodbye! Switching back to normal mode."]
    ovos.persona.dismissed        [persona_id="alice", session_id="..."]
    ovos.intent.handler.complete
  ovos.utterance.handled
```

Session state after: `{persona_id: absent}` — no-persona mode
resumes. The next utterance goes through the full deterministic
pipeline.

---

## Dismiss via stop cascade

This is deployment-specific. A deployment that wires stop to clear
`persona_id` typically does so in its stop pipeline plugin's
cascade step, setting `persona_id` to absent in the session before
or after emitting the stop dispatch. The persona plugin then detects
the cleared field on the next utterance (if any) and emits
`ovos.persona.dismissed`. There is no dedicated stop↔persona bus
event; the session field change is the signal.

---

## Out-of-band query (skill asks persona directly)

```
ovos.persona.ask                 [persona_id="alice", utterance="summarise X", source=<skill>]
  │
  ├─ persona plugin (supports "alice") receives it
  │    generates reply internally
  │
  ovos.persona.ask.response      [persona_id="alice", utterance="summarise X", response="..."]
                                  routed via reply() back to <skill>
```

No pipeline interaction, no `ovos.utterance.handle`, no
handler-lifecycle trio. Session state is unchanged.
