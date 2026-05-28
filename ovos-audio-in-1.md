# Audio Input Service Specification

**Spec ID:** OVOS-AUDIO-IN-1 · **Version:** 1 · **Status:** Draft

This specification defines the **audio input service** — the component
that acquires audio, processes it through the pre-STT transformer
chain, transcribes it to text, and injects the result into the
utterance lifecycle.

How audio is acquired — microphone capture, file playback, remote
streaming, wake-word gating, voice-activity detection, push-to-talk,
or any other mechanism — is deployer-defined and out of scope.

It builds on two companion specifications:

- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the `ovos.utterance.handle` entry point (§9.1);
- the *Transformer Plugins Specification* (OVOS-TRANSFORM-1) — the
  audio-transformer chain (§3.1) that runs before STT.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- **the audio input role** (§2) — what the service produces;
- **the STT obligation** (§3) — that a transcription mechanism exists;
- **the audio-transformer obligation** (§4) — running the pre-STT
  transformer chain;
- **the utterance emission** (§5) — topic, payload shape, and language
  resolution.

It does **not** define:

- **audio capture** — microphone access, file reading, remote streaming,
  wake-word detection, VAD, push-to-talk, or any other acquisition
  mechanism;
- **STT engine selection** — which engine is used or how it is
  configured;
- **post-STT processing** — utterance transformers
  (OVOS-TRANSFORM-1 §3.2) and metadata transformers (§3.3) are
  deployer concerns; the service MAY run them before emission;
- **session lifecycle** — how sessions are created or identified.

---

## 2. The audio input role

The audio input service acquires audio by any deployer-defined
mechanism, processes it through the audio-transformer chain (§4),
transcribes it via a STT mechanism (§3), and emits the result on
`ovos.utterance.handle` (§5).

It is the **producer** of utterance lifecycle messages and the first
component in the utterance lifecycle per OVOS-PIPELINE-1 §9.

---

## 3. STT mechanism

The audio input service **MUST** have access to a speech-to-text
mechanism that converts processed audio into one or more candidate
transcription strings. The specific engine, model, API, or local
process is deployer-defined; this specification places no constraint
on it beyond the requirement that it exists and produces text.

---

## 4. Audio-transformer chain

Before passing audio to the STT mechanism, the audio input service
**MUST** run the audio-transformer chain (**OVOS-TRANSFORM-1 §3.1**).
The chain is ordered and configured per OVOS-TRANSFORM-1 §4; the
`context.session` is passed to each transformer.

Canonical audio transformer use cases include:

- **Language identification** — detecting the spoken language from
  the audio signal and writing it to `session.detected_lang`, so
  that §5.1 language resolution and the STT engine can use it.
- **Denoising and normalisation** — background noise reduction, gain
  normalisation, sample-rate or format conversion before STT.
- **Speaker recognition** — identifying the speaker from the audio
  and writing the result into `Message.context` (e.g. a `speaker_id`
  key) so that downstream pipeline stages and skills can personalise
  responses without the audio input service knowing their semantics.

A deployment with no audio transformers configured passes audio to
STT unchanged.

---

## 5. Utterance emission

After transcription the audio input service **MUST** emit:

`ovos.utterance.handle`

per **OVOS-PIPELINE-1 §9.1**, with `context.session` populated per
**OVOS-MSG-1 §4**.

Payload:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of string | yes | One or more candidate transcription strings. The first element is the primary candidate. |
| `lang` | string | yes | The BCP-47 language tag for the transcription. See §5.1. |

### 5.1 Language resolution

`data.lang` MUST be set to the language the STT mechanism transcribed
in. The service selects the STT language from these inputs in order:

1. `session.detected_lang` — the language a language-detection audio
   transformer classified the audio as (**OVOS-SESSION-1 §3.2.6**).
   Most specific signal; use it when present.
2. `session.request_lang` — a hint from the capture mechanism about
   the expected language (e.g. the wake word that triggered capture,
   or a UI language selector) (**OVOS-SESSION-1 §3.2.5**). A prior,
   not a guarantee.
3. `session.lang` — the session's general language preference
   (**OVOS-SESSION-1 §3.2.1**).

The first present and non-empty value wins. If none is present the
service SHOULD use a deployment-configured default language.

The service SHOULD write the selected input language to
`session.stt_lang` (**OVOS-SESSION-1 §3.2.4**) before or at the
point of STT invocation. `stt_lang` records the language the STT
model was **configured to assume** for the audio, which normally
matches `data.lang` but may differ when the STT model performs
speech translation — in that case `stt_lang` is the audio's
language and `data.lang` is the transcription's output language.
Downstream stages that need to know the audio's source language
(rather than the transcript's language) read `session.stt_lang`.

---

## 6. Conformance

### An audio input service **MUST**:

- have access to a STT mechanism (§3);
- run the audio-transformer chain (OVOS-TRANSFORM-1 §3.1) before
  passing audio to STT (§4);
- emit `ovos.utterance.handle` with `data.utterances` (array of
  strings) and `data.lang` (BCP-47 tag) after transcription (§5);
- populate `context.session` per OVOS-MSG-1 §4.

### An audio input service **SHOULD**:

- write `session.stt_lang` to the language STT decoded in, after
  transcription (§5.1).

### An audio input service **MAY**:

- acquire audio by any mechanism (§2);
- run the utterance-transformer chain (OVOS-TRANSFORM-1 §3.2) on the
  transcription before emission;
- emit multiple candidate transcriptions in `data.utterances`.

---

## See also

- **OVOS-PIPELINE-1** — utterance lifecycle entry point (§9.1).
- **OVOS-TRANSFORM-1** — audio-transformer chain (§3.1).
- **OVOS-SESSION-1** — `session.lang`, `session.stt_lang`,
  `session.detected_lang`.
- **OVOS-MSG-1** — session carrier (§4) and envelope.
- **OVOS-AUDIO-1** — the audio output service.
