# Audio Output Service Specification

**Spec ID:** OVOS-AUDIO-1 · **Version:** 2 · **Status:** Draft

This specification defines the **audio output service** — the
pipeline's output-side counterpart that consumes natural-language
responses and renders them as audio. It covers two rendering modes
(`ovos.utterance.speak` for local playback and
`ovos.utterance.speak.b64` for remote-client delivery), a sequential
playback queue for speech and sound effects, fire-and-forget instant
sounds, and the output lifecycle signals that bookend audio playback.

It builds on three companion specifications:

- the *Utterance Lifecycle and Pipeline Specification*
  (OVOS-PIPELINE-1) — the pipeline iteration, the `Match` and
  dispatch contract, the handler-lifecycle trio, and the
  `ovos.utterance.speak` output exit point;
- the *Bus Message Specification* (OVOS-MSG-1) — the envelope,
  routing keys, session carrier, and derivations every Message
  defined here travels in;
- the *Transformer Plugins Specification*
  (OVOS-TRANSFORM-1) — the dialog-transformer and TTS-transformer
  chains that run before and after TTS synthesis.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
and **MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- **the audio output service role** (§2) — the component that
  receives natural-language responses and renders them as audio;
- **the rendering pipeline** (§3) — two rendering modes sharing the
  same TTS pipeline: `ovos.utterance.speak` enqueues for local
  playback; `ovos.utterance.speak.b64` emits synthesised audio as
  base64 for remote clients instead;
- **the playback model** (§4) — the scheduled queue for TTS
  speech and queued sounds, and fire-and-forget instant sounds
  for immediate playback;
- **output lifecycle signals** (§5) — the start/end markers that
  bookend audio playback;
- **stop integration** (§6) — how the audio service responds to
  stop signals;
- **bus surface** (§7);
- **conformance** (§8).

It does **not** define:

- **the internal machinery of TTS synthesis** — how a TTS plugin
  converts text to audio, including model inference, voice
  selection, and audio formatting, is entirely the plugin's
  business. The spec fixes only the observable bus contract;
- **the transformer plugin internals** — dialog and TTS
  transformer chains are defined by OVOS-TRANSFORM-1; this spec
  only fixes when they run in the output pipeline;
- **the audio-input pipeline** — microphone capture, wake-word
  detection, and speech-to-text are separate services covered by
  other specifications;
- **hardware access** — how the service accesses audio output
  hardware is a deployment concern;
- **volume control, audio routing, or hardware abstraction** —
  these are deployment-level concerns;
- **music and media playback** — long-form audio is managed by a
  separate media-playback service. This spec covers TTS speech and
  sound effects only.

---

## 2. The audio output service role

The **audio output service** is the component that receives
natural-language response text from the pipeline and renders it as
audible output. It:

- subscribes to `ovos.utterance.speak` (OVOS-PIPELINE-1 §9.6) and
  `ovos.utterance.speak.b64` (§3.4) and processes each through the
  same TTS rendering pipeline (§3), differing only in output stage;
- maintains a **scheduled playback queue** (§4.1) for TTS speech
  and queued sounds, ensuring that audio is played back in order
  without overlapping;
- plays **instant sounds** (§4.2) immediately on receipt,
  independently of the scheduled queue and without stopping it;
- emits **output lifecycle signals** (§5) around each playback
  session;
- responds to **stop signals** (§6) by clearing the queue and
  terminating in-progress playback.

A deployment **MAY** have no audio output service. The pipeline
and handler lifecycle are unaffected by its absence.

The handler does not block on audio output; playback may occur after
`ovos.utterance.handled` has fired (PIPELINE-1 §6.1).

---

## 3. Rendering pipeline

Both `ovos.utterance.speak` and `ovos.utterance.speak.b64` pass
through the same TTS pipeline. They differ only in the output stage:

```
ovos.utterance.speak        ovos.utterance.speak.b64
         │                           │
         ▼                           ▼
 [dialog transformers]      [dialog transformers]   ← TRANSFORM-1 §3.5
         │                           │
         ▼                           ▼
   TTS synthesis               TTS synthesis
         │                           │
         ▼                           ▼
  [tts transformers]         [tts transformers]      ← TRANSFORM-1 §3.6
         │                           │
         ▼                           ▼
 scheduled queue          ovos.audio.speech (§4.3)
  → local playback          (b64 for remote client)
```

All rendering stages execute in the audio output service, which MAY
run in the same process as the utterance orchestrator or separately.

### 3.1 Dialog transformer stage

Before TTS synthesis, the utterance text is passed through the
**dialog-transformer chain** (OVOS-TRANSFORM-1 §3.5) hosted by the
audio output service. Each transformer plugin in the chain receives
the text and the Message context and MAY mutate either.

The transformed text replaces the original `utterance` field for
all downstream stages.

### 3.2 TTS synthesis

The audio output service synthesises the utterance text into audio.
Language is taken from `data.lang` in the received Message
(PIPELINE-1 §9.6); when absent, the service resolves it from the
session (OVOS-SESSION-1 §3.2).

When synthesis fails for a segment — the whole utterance, or one
sentence-segmented chunk of it (appendix §4.9) — the service **SHOULD**
attempt a fallback for that segment. Selection and fallback logic
are deployment concerns. Beyond that, this spec fixes no bus
contract for partial-utterance failure: whether a failed segment is
skipped, retried, or aborts the remaining segments is not
observable on the bus. The one guarantee is §5.2 — `ovos.audio.output.ended`
fires once the queue empties, whether it emptied by completing every
segment or by giving up on the failed ones.

For `ovos.utterance.speak`, the synthesised audio is enqueued for
local playback (§4). For `ovos.utterance.speak.b64`, the synthesised
audio is emitted as `ovos.audio.speech` (§3.4) instead — it is not
enqueued and does not play locally.

> **Note (non-normative):** See appendix §4.9 for a discussion of
> sentence-segmentation as a latency-reduction technique.

### 3.3 TTS transformer stage

After synthesis, the audio data and Message context are passed
through the **TTS-transformer chain** (OVOS-TRANSFORM-1 §3.6)
hosted by the audio output service. Each transformer plugin MAY mutate the audio data.

The transformed audio replaces the original for playback.

### 3.4 Remote-client rendering mode — `ovos.utterance.speak.b64`

The audio output service **MUST** subscribe to
`ovos.utterance.speak.b64`. A Message on this topic carries the same
`utterance` text as `ovos.utterance.speak` and passes through the
same dialog-transformer, TTS-synthesis, and TTS-transformer stages
(§3.1–§3.3). The output stage differs: instead of enqueueing for
local playback, the service **MUST** emit `ovos.audio.speech` (§4.3)
with the synthesised audio encoded as base64. The audio is not
enqueued and does not play on the local device.

The `listen` flag (§4.4) applies: if the originating Message carries
`listen: true`, the service **MUST** emit `ovos.mic.listen` after
emitting `ovos.audio.speech`.

---

## 4. Playback model

The audio output service has one scheduled queue and a separate
instant-sound mechanism:

- **Scheduled playback queue** (§4.1) — sequential, one-at-a-time
  playback for TTS speech and queued sound effects. Audio plays in
  FIFO order without overlapping.
- **Instant sounds** (§4.2) — fire-and-forget playback that starts
  immediately on receipt. Instant sounds are independent of the
  queue: they play over whatever is currently scheduled, MAY overlap
  each other, and are not stoppable.

### 4.0 Audio format

Every audio payload in this specification — whether referenced by
`uri` or carried inline as base64 in `audio` — follows one format
rule:

- **The container is the single source of truth.** Audio payloads
  SHOULD be delivered in a self-describing container (e.g. WAV, MP3,
  OGG, FLAC). Such containers identify themselves through their
  leading bytes, and WAV additionally carries sample rate and channel
  layout; the payload therefore needs no out-of-band format
  declaration.
- **`mime` is OPTIONAL and advisory.** A producer MAY set `mime` as a
  hint; a consumer MAY use it to skip container detection. When the
  hint and the payload's container disagree, the container is
  authoritative.
- **`sample_rate` is OPTIONAL.** It is meaningful only when the
  container does not carry the sample rate itself.
- **Headerless raw streams are the exception.** When the payload is a
  raw, containerless stream (e.g. raw PCM), nothing in the bytes
  describes the format, so the producer MUST set both `mime` and
  `sample_rate`.

These rules apply uniformly to `ovos.audio.queue` (§4.1),
`ovos.audio.play_sound` (§4.2), and `ovos.audio.speech` (§4.3).

### 4.1 Scheduled playback queue

This queue holds TTS speech (from `ovos.utterance.speak`, §3.2)
and queued sounds (from `ovos.audio.queue`, below).

**Session scope.** The audio output service MUST only enqueue items
whose `context.session.session_id` matches a session it is
configured to serve locally. A service co-located with the
orchestrator on a single device SHOULD serve only
`session_id: "default"` (**OVOS-SESSION-2 §5**) and MUST NOT
enqueue audio for named sessions — those sessions belong to remote
participants and their audio is delivered via
`ovos.utterance.speak.b64` / `ovos.audio.speech` (§3.4, §4.3).

**Discipline:**
- **FIFO**. Items are dequeued in the order they were enqueued.
- **Sequential**. Each item plays to completion before the next
  item begins.
- **Clearable**. On a stop signal (§6), the queue is emptied of
  all pending items and any in-progress playback is terminated.

**Queued sounds** use topic `ovos.audio.queue`:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `uri` | string | no | URI referencing the audio data. |
| `audio` | string | no | Base64-encoded audio data, used when the audio source is on a different host (alternative to `uri`). |
| `mime` | string | no* | Advisory MIME type of the audio (§4.0). |
| `sample_rate` | number | no* | Sample rate in Hz (§4.0). |
| `listen` | bool | no | When `true`, re-opens the user input channel after this item plays (§4.4). |

Exactly one of `uri` or `audio` MUST be present. \* `mime` and
`sample_rate` are REQUIRED when the payload is a headerless raw
stream (§4.0).

### 4.2 Instant sounds

Instant sounds are played via `ovos.audio.play_sound`. They start
immediately on receipt, play over any audio currently in progress
from the scheduled queue, MAY overlap each other, and are **not**
affected by stop signals (§6).

**Play-sound topic** `ovos.audio.play_sound`:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `uri` | string | no | URI referencing the audio data. |
| `audio` | string | no | Base64-encoded audio data, used when the audio source is on a different host (alternative to `uri`). |
| `mime` | string | no* | Advisory MIME type of the audio (§4.0). |
| `sample_rate` | number | no* | Sample rate in Hz (§4.0). |

Exactly one of `uri` or `audio` MUST be present. \* `mime` and
`sample_rate` are REQUIRED when the payload is a headerless raw
stream (§4.0).

### 4.3 Synthesised audio delivery — `ovos.audio.speech`

`ovos.audio.speech` is emitted by the audio output service when
processing an `ovos.utterance.speak.b64` Message (§3.4). It carries
the synthesised audio as base64; the receiving client is responsible
for decoding and playing it.

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `audio` | string | yes | Base64-encoded synthesised audio. |
| `mime` | string | no* | Advisory MIME type of the audio (§4.0). |
| `sample_rate` | number | no* | Sample rate in Hz (§4.0). |
| `listen` | bool | no | When `true`, the client SHOULD re-open its microphone after playback. |

\* `mime` and `sample_rate` are REQUIRED when the payload is a
headerless raw stream (§4.0).

The session is identified via `context.session` as usual. A bridge
(OVOS-BRIDGE-1 §4.2.5) subscribes by `session_id` or `destination`
and relays this message to the client.

### 4.4 Listen flag

The `listen` field on `ovos.utterance.speak` is defined by
OVOS-PIPELINE-1 §9.6. This timing — gated on `ovos.audio.output.ended`
— applies to the **local-playback path** (`ovos.utterance.speak`,
§3, §4.1). When a received Message carries `listen: true`, the audio
output service **MUST** emit `ovos.mic.listen` after all audio for
that utterance has completed and after `ovos.audio.output.ended`
(§5.2).

On a stop-initiated end (§6), `ovos.mic.listen` is **NOT** emitted
regardless of the `listen` flag.

The remote-client path (`ovos.utterance.speak.b64`) does not enqueue
into the scheduled queue and so never reaches `ovos.audio.output.ended`;
its `listen` handling is the exception fixed by §3.4, which emits
`ovos.mic.listen` immediately after `ovos.audio.speech`.

---

## 5. Output lifecycle signals

The audio output service emits lifecycle signals around playback
to notify other components of audio state.

### 5.1 Playback start

When the first item in a playback session begins (queue was empty,
first item dequeued), the audio output service **MUST** emit:

`ovos.audio.output.started`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

A playback session runs from the first item's start until the queue
is empty and the last item completes. `ovos.audio.output.started`
fires once per idle→active transition.

### 5.2 Playback end

When the queue becomes empty and the last item has completed
playback, the audio output service **MUST** emit:

`ovos.audio.output.ended`

Payload:

No payload. The session is identified by `context.session.session_id`
of this Message.

Components that subscribed to `ovos.audio.output.started` use this
signal to restore state.

If the last completed item carried `listen: true` (§4.4), the audio
output service emits `ovos.mic.listen` **after** `ovos.audio.output.ended`.
On a stop-initiated end, `ovos.mic.listen` is not emitted (§4.4).

### 5.3 Speaking-status query

A component MAY query whether the audio output service is
currently speaking by emitting:

`ovos.audio.is_speaking`

Request payload: none. To scope the query to a specific session,
the requester sets `context.session.session_id` in the request
Message; the service answers for that session only. An absent or
`"default"` `session_id` asks about the device-local default session
(OVOS-SESSION-1 §3.1); it is not a wildcard over all sessions.

The service **MUST** reply via the `response` derivation
(OVOS-MSG-1 §5.3) — on `ovos.audio.is_speaking.response` — with:

```json
{ "speaking": true }
```

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `speaking` | bool | yes | Whether audio is currently playing for the session identified by `context.session.session_id` of the request. |

`is_speaking` is deliberately session-scoped; this version defines
no aggregate query to ask whether any session is speaking.

---

## 6. Stop integration

When the audio output service receives a stop signal, it:

1. **clears** the scheduled playback queue of all pending items;
2. **terminates** any in-progress scheduled playback;
3. **emits** `ovos.audio.output.ended` if a playback session was
   active.

Instant sounds (§4.2) are not affected by stop signals — they play
to completion regardless.

The stop signal topics are:

| Topic | Purpose |
|-------|---------|
| `ovos.audio.stop` | Stop audio output. |
| `ovos.stop` | Universal stop broadcast (OVOS-STOP-1). |

Both signals carry `context.session.session_id` (OVOS-MSG-1 §4).
The audio output service **MAY** scope its response to that session.

**Barge-in is deployer policy.** Whether wake-word detection during
active playback interrupts that playback ("barge-in") is not fixed
by this spec. A controller MAY subscribe to `ovos.listener.wakeword`
(OVOS-AUDIO-IN-1 §6.5) and, on receipt while playback is in progress,
emit `ovos.audio.stop` to interrupt it. This spec mandates neither
that behaviour nor its absence — the audio output service's only
obligation on receiving a stop signal is the sequence above.

---

## 7. Bus surface

| Topic | Direction | Purpose |
|-------|-----------|---------|
| `ovos.utterance.speak` | handler → audio | Natural-language response text for TTS + local playback (PIPELINE-1 §9.6). |
| `ovos.utterance.speak.b64` | handler/bridge → audio | Natural-language response text for TTS + remote delivery via `ovos.audio.speech` (§3.4). |
| `ovos.audio.queue` | any component → audio | Queue a sound for scheduled playback (§4.1). |
| `ovos.audio.play_sound` | any component → audio | Play a sound immediately (§4.2). |
| `ovos.audio.stop` | any component → audio | Stop audio playback and clear queue (§6). |
| `ovos.audio.is_speaking` | any component → audio | Query whether audio is currently playing (§5.3). |
| `ovos.audio.output.started` | audio → broadcast | Playback session started (§5.1). |
| `ovos.audio.output.ended` | audio → broadcast | Playback session ended (§5.2). |
| `ovos.audio.speech` | audio → broadcast | Synthesised audio as base64 for remote clients (§4.3). |
| `ovos.mic.listen` | audio → broadcast | Request microphone re-open after `listen: true` (§4.4). |

---

## 8. Conformance

### An audio output service **MUST**:

- subscribe to `ovos.utterance.speak` and process each Message
  through the TTS rendering pipeline for local playback (§3);
- subscribe to `ovos.utterance.speak.b64` and process each Message
  through the same TTS pipeline, emitting `ovos.audio.speech`
  instead of enqueueing for local playback (§3.4);
- maintain a scheduled playback queue that plays one item at a
  time in FIFO order (§4.1);
- support queued sound playback via `ovos.audio.queue` (§4.1);
- play instant sounds immediately on `ovos.audio.play_sound` without
  queuing or stopping scheduled playback (§4.2);
- emit `ovos.audio.output.started` when a playback session begins
  (§5.1);
- emit `ovos.audio.output.ended` when a playback session ends (§5.2);
- clear the scheduled queue and terminate playback on stop signals (§6);
- emit `ovos.mic.listen` after playback when the last item carries
  `listen: true` (§4.4);
- suppress `ovos.mic.listen` when playback ends due to a stop signal (§4.4, §6).

### An audio output service **SHOULD**:

- pass utterance text through the dialog-transformer chain before
  TTS synthesis (§3.1);
- pass the synthesized audio through the TTS-transformer chain
  before enqueueing (§3.3);

### An audio output service **MAY**:

- scope stop responses to the `context.session.session_id` in the stop signal (§6).

---

## See also

- *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
  — the pipeline iteration, `ovos.utterance.speak`, and `ovos.utterance.handled`.
- *Bus Message Specification* (OVOS-MSG-1) — the envelope and
  derivations used for all bus communication.
- *Transformer Plugins Specification* (OVOS-TRANSFORM-1) —
  the dialog-transformer and TTS-transformer chains that plug into
  the rendering pipeline.
- *Stop Pipeline Plugin Specification* (OVOS-STOP-1) — the universal
  `ovos.stop` broadcast that the audio output service responds to.
