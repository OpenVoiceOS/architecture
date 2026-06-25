# Audio Input Service Specification

**Spec ID:** OVOS-AUDIO-IN-1 · **Version:** 2 · **Status:** Draft

This specification defines the **audio input service** — the component
that acquires audio, runs the pre-STT transformer chain, transcribes
to text, and injects the result into the utterance lifecycle. How
audio is acquired is deployer-defined and out of scope.

It builds on three companion specifications:

- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the `ovos.utterance.handle` entry point (§9.1);
- the *Transformer Plugins Specification* (OVOS-TRANSFORM-1) — the
  audio-transformer chain (§3.1) that runs before STT;
- the *Session Lifecycle and State Ownership Specification*
  (OVOS-SESSION-2) — session assignment as the originator of
  interactions.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification does **not** define:

- **audio capture** — acquisition mechanism is deployer-defined;
- **STT engine selection** — engine, model, or API is deployer-defined;
- **post-STT transformer chains** — utterance and all subsequent
  transformer stages are owned by the utterance lifecycle
  (OVOS-PIPELINE-1) and the audio output layer (OVOS-AUDIO-1);
- **session persistence and resumption** — owned by OVOS-SESSION-2;
  this spec defines only which session the emission carries (§5.2).

---

## 2. The audio input role

The audio input service acquires audio by any deployer-defined
mechanism, runs the audio-transformer chain (§4), transcribes via a
STT mechanism (§3), and emits the result on `ovos.utterance.handle`
(§5). It is the **producer** of utterance lifecycle messages per
OVOS-PIPELINE-1 §9.

---

## 3. STT mechanism

The audio input service **MUST** have access to a speech-to-text
mechanism. The engine, model, API, or local process is
deployer-defined.

---

## 4. Audio-transformer chain

Before passing audio to STT, the audio input service **MUST** run the
audio-transformer chain (**OVOS-TRANSFORM-1 §3.1**), configured per
OVOS-TRANSFORM-1 §4.

Canonical use cases:

- **Language identification** — writes `session.detected_lang` for
  §5.1 language resolution and STT engine selection.
- **Denoising and normalisation** — noise reduction, gain
  normalisation, format conversion.
- **Voice-print recognition** — writes an intermediate result to
  `Message.context` (e.g. `context.voice_match`) for consolidation
  by a metadata transformer into `session.voice_id` per
  OVOS-USER-ID-1 §4.1.

---

## 5. Utterance emission

After transcription the audio input service **MUST** emit:

`ovos.utterance.handle`

per **OVOS-PIPELINE-1 §9.1**.

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of string | yes | Transcription candidates; first element is primary. |
| `lang` | string | yes | BCP-47 output language of the transcription. See §5.1. |

### 5.1 Language resolution

Select the STT input language in this order:

1. `session.detected_lang` (**OVOS-SESSION-1 §3.2.6**) — audio
   transformer's language classification.
2. `session.request_lang` (**OVOS-SESSION-1 §3.2.5**) — hint from
   the capture mechanism (e.g. wake word, UI language selector).
3. `session.lang` (**OVOS-SESSION-1 §3.2.1**) — session's general
   language preference.

First present and non-empty value wins. If none is present use a
deployment-configured default.

The service SHOULD write the selected language to `session.stt_lang`
(**OVOS-SESSION-1 §3.2.4**) before STT invocation. `stt_lang`
records the model's assumed input language and normally matches
`data.lang`; they diverge in speech-translation models where the
audio and transcript languages differ.

### 5.2 Session assignment

The audio input service **MUST** assign a session to every emission,
placed in `context.session` (**OVOS-MSG-1 §4**).

- **Local device** — SHOULD use `session_id: "default"`
  (**OVOS-SESSION-2 §5**).
- **Satellite** — session is assigned by the bridge at the hub
  boundary (**OVOS-BRIDGE-1 §4.2.1**); the bridge relays or
  NAT-translates the `session_id` as needed.

---

## 6. Listening lifecycle signals

The audio input service emits lifecycle signals around voice-command
capture and sleep mode to notify other components of listener state.

### 6.1 Capture start

When voice-command capture begins, the audio input service **MUST**
emit:

`ovos.listener.record.started`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

### 6.2 Capture end

When capture ends, the audio input service **MUST** emit:

`ovos.listener.record.ended`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

This signal pairs with `ovos.listener.record.started` (§6.1); a component
that subscribed to the start signal uses this to restore state.

### 6.3 Sleep mode

A controller (e.g. a naptime skill) requests sleep mode by emitting:

`ovos.listener.sleep`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

On receipt the audio input service enters sleep mode and suspends
capture until it is awoken (§6.4).

### 6.4 Awoken

When the audio input service leaves sleep mode, it **MUST** emit:

`ovos.listener.awoken`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

This signal fires only on the sleep→awake transition; it is not
emitted when the service is already awake.

### 6.5 Bus surface

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.listener.record.started` | audio-input → broadcast | Voice-command capture began (§6.1). |
| `ovos.listener.record.ended` | audio-input → broadcast | Voice-command capture ended (§6.2). |
| `ovos.listener.sleep` | controller → audio-input | Enter sleep mode and suspend capture (§6.3). |
| `ovos.listener.awoken` | audio-input → broadcast | Left sleep mode (§6.4). |
| `ovos.mic.listen` | any component → audio-input | Re-open the user input channel; consumed here, defined in OVOS-AUDIO-1 §4.4. |

---

## 7. Conformance

### An audio input service **MUST**:

- have access to a STT mechanism (§3);
- run the audio-transformer chain (OVOS-TRANSFORM-1 §3.1) before
  STT (§4);
- assign a session in `context.session` per §5.2;
- emit `ovos.utterance.handle` with `data.utterances` and `data.lang`
  (§5);
- emit `ovos.listener.record.started` when voice-command capture begins and
  `ovos.listener.record.ended` when it ends (§6.1, §6.2);
- emit `ovos.listener.awoken` on the sleep→awake transition (§6.4).

### An audio input service **SHOULD**:

- use `session_id: "default"` when co-located with the orchestrator
  (§5.2);
- write `session.stt_lang` before STT invocation (§5.1).

### An audio input service **MAY**:

- emit multiple candidate transcriptions in `data.utterances`.

---

## See also

- **OVOS-PIPELINE-1** — utterance lifecycle entry point (§9.1);
  post-STT transformer chains are owned here.
- **OVOS-AUDIO-1** — audio output service; owns dialog and TTS
  transformer chains, and defines `ovos.mic.listen` (§4.4) which the
  audio input service consumes (§6.5).
- **OVOS-TRANSFORM-1** — audio-transformer chain (§3.1).
- **OVOS-SESSION-1** — `session.lang`, `session.stt_lang`,
  `session.detected_lang`, `session.request_lang`.
- **OVOS-SESSION-2** — session assignment and default-session rule.
- **OVOS-MSG-1** — session carrier (§4) and envelope.
- **OVOS-BRIDGE-1** — satellite session assignment (§4.2.1).
- **OVOS-USER-ID-1** — user identity resolution; voice-print
  recognition is an audio-transformer use case (§4.1).
