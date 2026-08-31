# Common Playback: the Virtual Media Player

**Spec ID:** OVOS-OCP-1 · **Version:** 2 · **Status:** Draft

This specification defines the **Virtual Media Player** — a single
logical media player, scoped to a session, that every media voice command
targets. It is the contract by which an orchestrator turns "play jazz",
"pause", "next", "louder", and "stop the music" into observable playback
state, and by which that state is mirrored to and from the host operating
system over open standards (MPRIS).

OCP stands for **Common Playback**: *common* because one player
arbitrates all media for a session regardless of which application,
provider, or output device ultimately serves a track — the same way the
intent stack gives one utterance one handler.

Dependencies: OVOS-MSG-1 (envelope and the `context.session` carrier),
OVOS-SESSION-1 (session field registry), OVOS-SESSION-2 (session
assignment and mutation boundaries), OVOS-PIPELINE-1 (a pipeline plugin
conformant to OVOS-PIPELINE-1 that classifies playback and control
utterances and dispatches into this surface), OVOS-STOP-1 (global stop
cascade, of which media stop is one subscriber).

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
the voice OS: if the player is bridged to an external OS source (§6), control
requests act on that source.

---

## 3. State model

The player exposes five orthogonal state axes: `PlayerState` (§3.1),
`MediaState` (§3.2), loop/shuffle (§3.3), track state (§3.4), and
`PlaybackType` (§3.5). Each axis has a fixed enumeration; an
implementation **MUST NOT** report a value outside its axis, and
**SHOULD** treat unknown received values as the axis's neutral member.

### 3.1 Player state

| `PlayerState` | Code | Meaning |
|---|---|---|
| `STOPPED` | `0` | No active playback. |
| `PLAYING` | `1` | A track is advancing. |
| `PAUSED` | `2` | A track is loaded and held at a position. |

Every axis member has a stable numeric **code**; the code, not the
symbolic name, is what travels on the wire (§4.4).

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

### 3.4 Track state

Track state reports where the now-playing track is in its per-backend
lifecycle. It is reported on `ovos.common_play.track.state` (§4.4) and
recorded on a media entry's `status` field (§4.5) once numeric codes are
assigned (§4.4, Open items).

| `TrackState` member | Meaning |
|---|---|
| `disambiguation` | A result exists (§4.2.1's `disambiguation` set) but is not queued to any backend. |
| `queued` | The track is waiting for a backend to start it. Qualified by backend kind (§3.5): skill-internal, audio, video, web view, external OS player. |
| `playing` | A backend has confirmed playback. Qualified by backend kind (§3.5). A `playing`-family value implies `PlayerState.PLAYING` (§3.1). |

Pausing is a `PlayerState` (§3.1) / `MediaState` (§3.2) concern and
**MUST NOT** be represented as a track-state value.

The lifecycle implied by this axis is `disambiguation` → `queued` →
`playing`. Transitions not listed here are not defined by this version
of the spec; an implementation **MUST NOT** treat them as valid.

### 3.5 Playback type

`PlaybackType` names the backend kind a media entry targets or a
queued/playing track state is qualified by (§3.4, §4.5 `playback`
field):

| `PlaybackType` member | Meaning |
|---|---|
| `skill-internal` | The originating skill renders the media itself (no shared backend). |
| `audio` | An audio backend (OVOS-AUDIO-1-adjacent playback service). |
| `video` | A video-capable backend / display surface. |
| `web view` | An embedded web view renders the media. |
| `external OS player` | An MPRIS-bridged external player (§6) is the target. |

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

When `playlist` is present, `media` **MUST** be a member of it: the
player locates `media`'s position within `playlist` by matching `uri`
(§4.5). `ovos.common_play.next` and `ovos.common_play.previous` (§4.3)
move relative to that located position, not to an externally supplied
index.

The `ovos.common_play.search` payload and the payloads of its
bracketing Messages (`search.start`, `search.end`) are all
implementation-defined in this version.

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

#### 4.3.1 Delegated control of skill-rendered playback

When the now-playing entry's backend kind is skill-internal (§3.5), the
originating skill renders the media and the player does not own the
output. The player remains the single addressee of §4.3 control
requests, and **MUST** forward the verbs it cannot act on itself to the
rendering skill as

`ovos.common_play.{skill_id}.{verb}`, verb ∈ `pause`, `resume`, `next`,
`previous`, `stop`, `seek`

where `{skill_id}` is the identifier the skill registered under.
Skills join the player's roster by emitting a registration announcement
on `ovos.common_play.announce`; its payload is implementation-defined in
this version except for the `can_seek` declaration below.
Delegation messages are emitted by the
player itself, consistent with the §4.1 reservation; a skill subscribes
to its own family and **MUST NOT** emit under it.

The `seek` delegation carries the §4.2.2 payload: one absolute
`position` in milliseconds.

Seek is the one delegated verb a rendering skill may be unable to
honour. A skill that can reposition its own rendering declares
`can_seek: true` in its registration announcement. Absent that
declaration the player **MUST** treat the skill's rendering as
non-seekable: it does not delegate `seek` to the skill and **MUST NOT**
advertise seekability for that media to OS integrations (§6.1).

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
| `state` | number | yes | The numeric code of the new state on the topic's axis (§3.1 `PlayerState`, §3.2 `MediaState`, or the track axis below). |

Currently, numeric codes are assigned only for `PlayerState` (0/1/2);
`MediaState` and track-state values remain symbolic pending future
assignment (§4.6 Open items).

`ovos.common_play.track.state` reports the §3.4 `TrackState` value.

A consumer **MUST NOT** assume it can read player state synchronously; the
state reports are the contract.

### 4.6 Open items

This version of the spec leaves the following unresolved; implementers
**MUST NOT** assume a numeric encoding for them beyond what is stated:

- **`MediaState` numeric codes** (§3.2) — members are named but no
  stable numeric code is assigned yet, unlike `PlayerState` (§3.1).
- **Track-state numeric codes** (§3.4) — the `TrackState` members
  (`disambiguation`, `queued`, `playing`, each qualified by
  `PlaybackType`, §3.5) are named but not yet numbered.

Until these are assigned, `MediaState` and track-state values travel
symbolically; a future version of this spec fixes their numeric codes
the way §3.1 already fixes `PlayerState`'s.

### 4.5 The media entry

Playback requests and state consumers exchange tracks as **media
entry** objects:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `uri` | string | yes | Where the media lives; scheme selects the backend/extractor. |
| `title` | string | no | Display title. |
| `artist` | string | no | Display artist. |
| `image` | string | no | Artwork, delivered per the GUI image rules (OVOS-GUI-1 §3.5). |
| `playback` | number | no | Requested playback kind on the `PlaybackType` axis (§3.5). |
| `status` | *reserved* | no | The entry's current track-state value (§3.4, §4.4). Reserved in this version pending the track-state numeric assignment (§4.6 Open items): producers **SHOULD** omit this field, and consumers **MUST** ignore any value present until numeric `TrackState` codes are assigned. |
| `media_type` | number | no | Content classification (music, radio, podcast, video, …) used for result ranking. |
| `length` | number (ms) | no | Track duration in milliseconds; `-1` = unknown/live. One time convention across the media surface: durations and positions count in milliseconds with `-1` meaning unknown/live (§4.2.2) — the convention OVOS-GUI-1 v2 adopts uniformly for its media templates (OVOS-GUI-1 §3.4). |
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
The MPRIS capability properties (`CanSeek`, `CanPause`, …) **MUST** report
the player's actual ability to honour the verb for the current media — in
particular, `CanSeek` is `false` while the rendering source cannot be
repositioned (§4.3.1).
This makes the virtual player's playback visible to and controllable by
ordinary desktop media keys and applets, with no knowledge of the voice OS.

A Role A exporter **MUST** report only MPRIS-valid strings (e.g.
`PlaybackStatus ∈ {Playing, Paused, Stopped}`, `LoopStatus ∈ {None, Track,
Playlist}`) and **MUST** degrade gracefully — log and continue — when no
session D-Bus is available (headless hosts).

### 6.2 Role B — control external players

The virtual player **MAY** discover and control *other* MPRIS players on
the host (`org.mpris.MediaPlayer2.*`). This is the key consequence of the
common-playback model: **media playback that the virtual player did not
initiate is still controllable by voice**, provided the source speaks an
open standard. "Pause the music" can pause a browser tab or a desktop
player; "next" can skip the system's current player.

Role B is **off by default** and gated by configuration, because it acts on
software the virtual player does not own. When enabled, the player **MUST**
maintain an ignore-list (at minimum its own export name from Role A) and **SHOULD**
scope control to the most recently active external player to avoid
ambiguous broadcast.

### 6.3 Arbitration

When both player-initiated playback and external MPRIS sources are present,
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
- forward control verbs for skill-rendered media to the rendering skill
  and honour its declared seek capability (§4.3.1);
- scope a stop to the inbound session (§7);
- reject a `ovos.common_play.play` request whose `media` entry lacks a
  `uri` before dispatching it to any backend (§4.2.1, §4.5);
- on acquisition failure (the resolved media cannot be loaded), announce
  `MediaState` `INVALID_MEDIA` (§3.2, §4.4);
- if the playback backend dies mid-track, announce `PlayerState`
  `STOPPED` (§3.1, §4.4).

### A Virtual Media Player implementation **SHOULD**:

- integrate with the host OS via MPRIS Role A (§6.1);
- degrade gracefully without a session D-Bus (§6.1).

### A Virtual Media Player implementation **MAY**:

- control external MPRIS players via Role B, off by default and gated by
  configuration (§6.2);
- accept a user-named output preference in a playback request (§4.2).

---

## See also

- **OVOS-PIPELINE-1** — a pipeline plugin conformant to OVOS-PIPELINE-1
  that classifies playback vs. control utterances and dispatches into the
  §4 surface.
- **OVOS-STOP-1** — global stop cascade; media stop is a subscriber (§7).
- **OVOS-SESSION-1 / OVOS-SESSION-2** — the `context.session` carrier and
  per-session ownership that scope the player (§5).
- **OVOS-MSG-1** — envelope and session carrier for all §4 traffic.
- **OVOS-AUDIO-1** — audio *output* service (TTS/dialog); distinct from
  media playback, which this spec owns.
