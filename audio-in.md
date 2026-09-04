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
OVOS-TRANSFORM-1 §4. Chain input metadata carries at minimum
sample rate, sample width, and channel count (OVOS-TRANSFORM-1
§3.1); this is the pre-STT audio-format vocabulary, and it is the
input-side counterpart to the `mime` and `sample_rate` fields
OVOS-AUDIO-1 defines on the output side.

When capture is in-process (audio input service and STT sharing a
process, no bus hop between them), the chain's `lang` parameter
machinery (**OVOS-TRANSFORM-1 §3.0**) degrades gracefully: there is
no Message yet whose `data.lang` could seed it, so `lang` simply
starts unset for the chain and the §3.0 writeback step is a no-op
(there is no `Message.data.lang` to reflect a value into). This is
not a gap — the chain's actual output reaches the pipeline through
`session.detected_lang` (below), which is the channel §5.1 language
resolution already reads.

Canonical use cases:

- **Language identification** — writes `session.detected_lang` for
  §5.1 language resolution and STT engine selection.
- **Denoising and normalisation** — noise reduction, gain
  normalisation, format conversion.
- **Voice-print recognition** — writes an intermediate result to
  `Message.context` (e.g. `context.voice_match` — an illustrative
  key name, not claimed by this specification) for downstream
  consolidation by a metadata transformer.

---

## 5. Utterance emission

After transcription the audio input service **MUST** emit:

`ovos.utterance.handle`

per **OVOS-PIPELINE-1 §9.1**.

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `utterances` | array of string | yes | Transcription candidates; first element is primary. |
| `lang` | string | no | BCP-47 output language of the transcription. See §5.1. |

Per **OVOS-PIPELINE-1 §9.1**, `lang` on this topic is present only
when the producer authoritatively knows the content language. The
audio input service satisfies that condition by construction: it
selected the STT decoder (§5.1) and therefore knows, without
inference, which language the decoder was run in. It therefore
always sets `lang`.

The service **MUST NOT** emit `ovos.utterance.handle` when STT
produced no usable transcription — an empty result, a decode
failure, or a confidence rejection. `utterances` is defined as
non-empty; a phantom emission with no real content is
non-conformant.

Instead the service **MUST** emit `ovos.stt.failed`, carrying the
session in `context.session` like every emission here (§5.2). The
payload **MAY** be empty; `lang` **MAY** be included under the §9.1
condition. The user spoke and nothing will answer — that fact must
be observable on the bus, or every client is left to guess between
"still transcribing" and "gave up". What a client does with it —
an error earcon, a retry prompt, nothing — is client policy, not
this specification's concern. No handler lifecycle follows: the
failure event is terminal, and it is the only Message the failed
capture produces.

### 5.1 Language resolution

The service **MUST** select the STT input language by this
precedence — a deterministic order is what lets every producer of a
language hint predict which language the transcription will assume:

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
  boundary (**OVOS-BRIDGE-1 §3.4.2**); the bridge relays or
  NAT-translates the `session_id` as needed (**OVOS-BRIDGE-1
  §3.2**).

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
`ovos.listener.record.ended` carries no completion guarantee about
what capture produced — the arrival (or absence) of a subsequent
`ovos.utterance.handle` (§5) is the only reliable signal of whether
usable audio resulted.

### 6.3 Sleep mode

A controller (e.g. a naptime skill) requests sleep mode by emitting:

`ovos.listener.sleep`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

On receipt the audio input service enters sleep mode and suspends
capture until it is awoken (§6.4). Sleep entry is **unacknowledged
by design**: no confirmation Message is emitted on entering sleep.
The only sleep-related emission is `ovos.listener.awoken` on the
sleep→awake transition (§6.4).

**Sleep is device-scoped.** Although the `ovos.listener.sleep`
request rides a session like every Message, sleep mode is a
**physical device state**: a sleeping audio input service captures
nothing for any session. Entering or leaving sleep affects the whole
device, not only the session that carried the request.

`context.session.session_id` on `ovos.listener.sleep` and
`ovos.listener.awoken` (§6.4) is therefore **informational only**:
it identifies the requester for logging and correlation, but the
device-scope effect — capture suspended or resumed for every
session — is identical no matter which session's identifier the
Message carries.

No topic in this specification lets a component query current
listener state (awake, asleep, capturing) on demand. This is a
**deliberate omission**: sleep and record signals (§6.1–§6.4) are
edge-triggered notifications, not a queryable state store. A
deployment that needs to synchronize to current state on connect
(e.g. a bridge attaching mid-session) derives it from the last-seen
lifecycle signal rather than polling for one.

### 6.4 Awoken

When the audio input service leaves sleep mode, it **MUST** emit:

`ovos.listener.awoken`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

This signal fires only on the sleep→awake transition; it is not
emitted when the service is already awake.

### 6.5 Wake-word detection

When a wake word triggers voice-command capture, the audio input
service **MUST** emit:

`ovos.listener.wakeword`

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `wake_word` | string | yes | The wake-word phrase that was detected, as configured (human-readable, space-separated). |
| `lang` | string | no | BCP-47 tag associated with the detected wake word, when the deployment binds wake words to languages. |

The session is identified by `context.session.session_id` of this
Message. This signal is the observable event behind a
wake-word-derived `session.request_lang` (**OVOS-SESSION-1
§3.2.5**): in a multi-wakeword deployment where each wake word is
bound to a language, the `lang` of the detected wake word is the
hint the emitter reports as `request_lang`.

The signal precedes `ovos.listener.record.started` (§6.1) — detection
is what opens capture. Deployments that open capture without a wake
word (push-to-talk, `ovos.mic.listen`) emit no wake-word signal.

### 6.6 Bus surface

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.listener.wakeword` | audio-input → broadcast | Wake word detected; capture opening (§6.5). |
| `ovos.listener.record.started` | audio-input → broadcast | Voice-command capture began (§6.1). |
| `ovos.listener.record.ended` | audio-input → broadcast | Voice-command capture ended (§6.2). |
| `ovos.listener.sleep` | controller → audio-input | Enter device-wide sleep mode and suspend capture (§6.3). |
| `ovos.listener.awoken` | audio-input → broadcast | Left sleep mode (§6.4). |
| `ovos.mic.listen` | any component → audio-input | Re-open the user input channel; consumed here, defined in OVOS-AUDIO-1 §4.4. |
| `ovos.stt.failed` | audio-input → broadcast | Capture yielded no usable transcription; terminal, no lifecycle follows (§5). |

---

## 7. Conformance

### An audio input service **MUST**:

- have access to a STT mechanism (§3);
- run the audio-transformer chain (OVOS-TRANSFORM-1 §3.1) before
  STT (§4);
- assign a session in `context.session` per §5.2;
- emit `ovos.utterance.handle` with `data.utterances` and set
  `data.lang` to the language the decoder ran in (§5.1); never a
  synthesized value (OVOS-PIPELINE-1 §9.1);
- emit `ovos.stt.failed` instead when STT produced no usable
  transcription, and no `ovos.utterance.handle` for that capture
  (§5);
- emit `ovos.listener.wakeword` when a wake word triggers capture
  (§6.5);
- emit `ovos.listener.record.started` when voice-command capture begins and
  `ovos.listener.record.ended` when it ends (§6.1, §6.2);
- treat sleep mode as device-scoped — suspend capture for all
  sessions while asleep (§6.3);
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
  audio input service consumes (§6.6).
- **OVOS-TRANSFORM-1** — audio-transformer chain (§3.1).
- **OVOS-SESSION-1** — `session.lang`, `session.stt_lang`,
  `session.detected_lang`, `session.request_lang`.
- **OVOS-SESSION-2** — session assignment and default-session rule.
- **OVOS-MSG-1** — session carrier (§4) and envelope.
- **OVOS-BRIDGE-1** — satellite session assignment (§3.4.2) and
  session `session_id` NAT translation (§3.2).
