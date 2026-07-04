# OVOS Common Playback: the Virtual Media Player

**Spec ID:** OVOS-OCP-1 · **Version:** 1 · **Status:** Draft

This specification defines the **OVOS Virtual Media Player** — a single
logical media player, scoped to a session, that every media voice command
targets. It is the contract by which an orchestrator turns "play jazz",
"pause", "next", "louder", and "stop the music" into observable playback
state, and by which that state is mirrored to and from the host operating
system over open standards (MPRIS).

OCP stands for **OVOS Common Playback**: *common* because one player
arbitrates all media for a session regardless of which application,
provider, or output device ultimately serves a track — the same way the
intent stack gives one utterance one handler.

Dependencies: OVOS-MSG-1 (envelope and the `context.session` carrier),
OVOS-SESSION-1 (session field registry), OVOS-SESSION-2 (session
assignment and mutation boundaries), OVOS-PIPELINE-1 (the media pipeline
that matches playback and control utterances and dispatches into this
surface), OVOS-STOP-1 (global stop cascade, of which media stop is one
subscriber).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, **MAY**
are used as in RFC 2119.

---

## 1. Scope

This specification defines: the Virtual Media Player abstraction and its
session scoping; the player and media **state model**; the **bus surface**
that requests playback and transport control and that reports state; the
distinction between **playback requests** (start something) and **control
requests** (act on what is already playing); and the **MPRIS bridge** by
which the virtual player is exported to the host OS and by which
externally-initiated, standards-compliant players become controllable by
voice.

It does **not** define: how candidate media is discovered or ranked (a
provider concern), how a URI is turned into bytes on a speaker (a backend
concern), stream-extraction formats, the GUI rendering of now-playing, the
NLU that classifies an utterance as media (OVOS-PIPELINE-1), or the
on-disk/programming-language shape of any implementation. These are
implementer concerns; this spec fixes only the **observable contract**.

---

## 2. The Virtual Media Player

A conformant orchestrator **MUST** present, per session, exactly one
**Virtual Media Player**: a single addressable target that owns the
session's *now-playing* track, its *playback queue*, and its *transport
state*. "Virtual" because it is not a device and not an application — it is
an arbitration point. Behind it, any number of playback backends, remote
devices, or external OS players may do the actual work; in front of it, all
voice commands and all status consumers see one coherent player.

This mirrors the rest of the platform: as the intent stack gives one
utterance one winning handler (OVOS-PIPELINE-1), the media stack gives one
session one player. A request never names a backend by necessity; it names
*the player*, and the player routes.

Two request classes target the player and **MUST** be distinguished:

- **Playback requests** — "play X", "open X". The player acquires media
  (via the pipeline and providers) and begins playback. §4.2.
- **Control requests** — "pause", "resume", "next", "previous", "stop",
  "shuffle", seek, repeat. The player acts on its existing now-playing /
  queue without acquiring new media. §4.3.

A control request **MUST NOT** require the player to have been started by
OVOS: if the player is bridged to an external OS source (§6), control
requests act on that source.

---

## 3. State model

The player exposes three orthogonal state axes. Each axis has a fixed
enumeration; an implementation **MUST NOT** report a value outside its
axis, and **SHOULD** treat unknown received values as the axis's neutral
member.

### 3.1 Player state

| `PlayerState` | Meaning |
|---|---|
| `STOPPED` | No active playback. |
| `PLAYING` | A track is advancing. |
| `PAUSED` | A track is loaded and held at a position. |

### 3.2 Media state

`MediaState` describes the *loaded media*, independent of whether it is
advancing: at minimum `NO_MEDIA`, `LOADING`, `LOADED`, `BUFFERING`,
`END_OF_MEDIA`, `INVALID_MEDIA`. It is advisory for consumers (e.g. a GUI
spinner) and **MUST NOT** be conflated with `PlayerState`.

### 3.3 Loop / shuffle

The player **MUST** track a loop mode (at minimum `NONE`, `REPEAT`,
`REPEAT_TRACK`) and a boolean shuffle mode. Both are control-request
targets (§4.3).

The player is the **single writer** of its own state. It emits a state
event when state changes (§4.4); it **MUST NOT** derive its authoritative
state by subscribing to its own emitted events.

---

## 4. Bus surface

All player traffic is namespaced under **`ovos.common_play.`** and carried
in the OVOS-MSG-1 envelope. Every message **MUST** carry `context.session`
(OVOS-SESSION-1); the player it addresses is the one owning that
`session_id` (§5).

### 4.1 Namespace reservation

The `ovos.common_play.` prefix is **reserved** for the Virtual Media
Player. Components other than the player and its pipeline **MUST NOT**
emit playback-mutating messages under this prefix; they observe state
(§4.4) and issue requests (§4.2, §4.3).

### 4.2 Playback requests

| Message | Meaning |
|---|---|
| `ovos.common_play.play` | Begin playback of a resolved result / queue. Payload in §4.2.1. |
| `ovos.common_play.search` | Acquire candidate media for a phrase (the pipeline's discovery step). Bracketed by the two Messages below. |
| `ovos.common_play.search.start` | The component orchestrating the search **MUST** emit this before querying providers. |
| `ovos.common_play.search.end` | The component orchestrating the search **MUST** emit this after result aggregation completes. |

A playback request **MAY** name a preferred output (a backend alias) in the
utterance; absent that, the player selects an output by its configured
preference order. Output selection is informative here and owned by the
implementation.

#### 4.2.1 `ovos.common_play.play` payload

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `media` | media entry (§4.5) | yes | The track to play now. |
| `playlist` | array of media entries | no | The playback queue. When absent, the queue is `[media]`. |
| `disambiguation` | array of media entries | no | The full candidate result set the queue was chosen from, kept for "play something else" style follow-ups. When absent, defaults to the playlist. |
| `repeat` | boolean | no | When `true`, the player enters loop mode `REPEAT` (§3.3). |

The search-bracketing Messages (`search`, `search.start`, `search.end`)
carry implementation-defined payloads in this version.

#### 4.2.2 `ovos.common_play.seek` payload

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `position` | number (ms) | yes | Absolute position within now-playing to move to. |

Seek is **absolute**: relative "skip forward N" commands are resolved
to an absolute position by the requester (which can read the state
reports of §4.4), not by the player — one addressing mode keeps
concurrent seekers from compounding each other's offsets.

### 4.3 Control requests

| Message | Acts on |
|---|---|
| `ovos.common_play.pause` | now-playing → `PAUSED` |
| `ovos.common_play.resume` | now-playing → `PLAYING` |
| `ovos.common_play.stop` | now-playing → `STOPPED` (an OVOS-STOP-1 subscriber, §7) |
| `ovos.common_play.next` | advance the queue |
| `ovos.common_play.previous` | retreat the queue |
| `ovos.common_play.seek` | move the position within now-playing |

Control requests are **idempotent with respect to absent media**: issuing
`pause` with nothing playing is a no-op, not an error.

### 4.4 State reports

The player **MUST** announce state transitions so that GUIs, satellites,
MPRIS exporters, and the pipeline's per-session tracking stay coherent:

| Message | Carries |
|---|---|
| `ovos.common_play.player.state` | the §3.1 value |
| `ovos.common_play.media.state` | the §3.2 value |
| `ovos.common_play.track.state` | now-playing track transitions |

All three share one payload shape:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `state` | number | yes | The new state value on the topic's axis (§3.1 `PlayerState`, §3.2 `MediaState`, or the track axis below). |

`ovos.common_play.track.state` reports where the now-playing track is
in its per-backend lifecycle. Its axis distinguishes *disambiguation*
(a result exists but is not queued), *queued* (waiting for a backend
to start), and *playing* (a backend confirmed playback), with the
queued/playing members qualified by backend kind (skill-internal,
audio, video, web view, external OS player). A `playing`-family value
implies `PlayerState.PLAYING`; pausing is a `PlayerState` /
`MediaState` concern and never a track-state value.

A consumer **MUST NOT** assume it can read player state synchronously; the
state reports are the contract.

### 4.5 The media entry

Playback requests and state consumers exchange tracks as **media
entry** objects:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `uri` | string | yes | Where the media lives; scheme selects the backend/extractor. |
| `title` | string | no | Display title. |
| `artist` | string | no | Display artist. |
| `image` | string | no | Artwork, delivered per the GUI image rules (OVOS-GUI-1 §3.5). |
| `playback` | number | no | Requested playback kind on the `PlaybackType` axis (skill-internal, audio, video, web view, external OS player). |
| `status` | number | no | The entry's current track-state value (§4.4). |
| `media_type` | number | no | Content classification (music, radio, podcast, video, …) used for result ranking. |
| `length` | number (seconds) | no | Track duration; `0` = unknown/stream. |
| `match_confidence` | number 0–100 | no | Provider's self-reported relevance for the originating query. |
| `skill_id` | string | no | The provider that produced this entry. |

Consumers **MUST** ignore unknown media-entry fields; providers ride
extra metadata on entries freely.

---

## 5. Session scoping

The Virtual Media Player is **per session** (OVOS-SESSION-2). A request's
`context.session.session_id` selects the player instance; co-located
single-user setups use `session_id: "default"`.

An orchestrator serving multiple concurrent sessions (e.g. a hub serving
satellites) **MUST** keep each session's now-playing, queue, and transport
state isolated: a `pause` for session A **MUST NOT** affect session B.
State reports (§4.4) **MUST** carry the originating session so consumers
can demultiplex.

---

## 6. OS integration — the MPRIS bridge

The Virtual Media Player **SHOULD** integrate with the host operating
system through **MPRIS** (the freedesktop `org.mpris.MediaPlayer2`
D-Bus interface). The bridge has two independent roles; an implementation
**MAY** provide either or both, and each **MUST** be separately
configurable.

### 6.1 Role A — export (the player as an MPRIS player)

The virtual player **MAY** publish itself on the session bus as an MPRIS
`MediaPlayer2` (e.g. `org.mpris.MediaPlayer2.OCP`), mapping §3 state and §4
transport onto the MPRIS `PlaybackStatus`, `LoopStatus`, `Metadata`,
`Position`, and the `Play`/`Pause`/`Next`/`Previous`/`Stop`/`Seek` methods.
This makes OVOS playback visible to and controllable by ordinary desktop
media keys and applets, with no knowledge of OVOS.

A Role A exporter **MUST** report only MPRIS-valid strings (e.g.
`PlaybackStatus ∈ {Playing, Paused, Stopped}`, `LoopStatus ∈ {None, Track,
Playlist}`) and **MUST** degrade gracefully — log and continue — when no
session D-Bus is available (headless hosts).

### 6.2 Role B — control external players

The virtual player **MAY** discover and control *other* MPRIS players on
the host (`org.mpris.MediaPlayer2.*`). This is the key consequence of the
common-playback model: **media playback that OVOS did not initiate is still
controllable by voice**, provided the source speaks an open standard. "Pause
the music" can pause a browser tab or a desktop player; "next" can skip the
system's current player.

Role B is **off by default** and gated by configuration, because it acts on
software OVOS does not own. When enabled, the player **MUST** maintain an
ignore-list (at minimum its own export name from Role A) and **SHOULD**
scope control to the most recently active external player to avoid
ambiguous broadcast.

### 6.3 Arbitration

When both OVOS-initiated playback and external MPRIS sources are present,
the virtual player is the single arbiter of "the current media" for control
requests (§4.3). The arbitration policy (prefer own playback, prefer most
recently active, etc.) is implementation-defined, but the player **MUST**
present one coherent answer per control request — a control request
**MUST NOT** act on two players at once unless the user explicitly requested
a global action.

---

## 7. Relationship to stop

Media stop is one subscriber to the OVOS-STOP-1 cascade. When the stop
pipeline dispatches to the media player's `…:stop` (the player being an
active handler) or broadcasts a global stop, the player **MUST** transition
now-playing to `STOPPED` (§3.1) and **MUST** scope the effect to the
inbound `session_id` (OVOS-STOP-1 §6). A global stop **MUST NOT** stop
another session's playback.

---

## 8. Conformance

### A Virtual Media Player implementation **MUST**:

- present exactly one player per session (§2, §5) and keep sessions
  isolated (§5);
- distinguish playback requests from control requests, and honour control
  requests against externally-sourced media when bridged (§2, §4.3, §6.2);
- be the single writer of its own state and announce transitions on
  `ovos.common_play.player.state` / `…media.state` / `…track.state` (§3.3,
  §4.4);
- treat control requests as no-ops when no media is present (§4.3);
- scope a stop to the inbound session (§7).

### A Virtual Media Player implementation **SHOULD**:

- integrate with the host OS via MPRIS Role A (§6.1);
- degrade gracefully without a session D-Bus (§6.1).

### A Virtual Media Player implementation **MAY**:

- control external MPRIS players via Role B, off by default and gated by
  configuration (§6.2);
- accept a user-named output preference in a playback request (§4.2).

---

## See also

- **OVOS-PIPELINE-1** — the media pipeline that classifies playback vs.
  control utterances and dispatches into the §4 surface.
- **OVOS-STOP-1** — global stop cascade; media stop is a subscriber (§7).
- **OVOS-SESSION-1 / OVOS-SESSION-2** — the `context.session` carrier and
  per-session ownership that scope the player (§5).
- **OVOS-MSG-1** — envelope and session carrier for all §4 traffic.
- **OVOS-AUDIO-1** — audio *output* service (TTS/dialog); distinct from
  media playback, which this spec owns.
