# GUI Display Subsystem Specification

**Spec ID:** OVOS-GUI-1 · **Version:** 1 · **Status:** Draft

This specification defines the **GUI display subsystem** — the layer
through which a voice application declares **what** to display, and a
set of render backends decide **how** to display it. It decouples
applications from rendering technology: an application states its
display intent in terms of a fixed vocabulary of semantic templates,
and any number of independently-installed render backends turn those
intents into pixels (or a character grid, or a synthesized face, or
spoken supplementary cues).

The subsystem is a voice-OS peripheral, not the primary interaction
surface. Voice is primary; the display **accompanies** speech and
**never** drives interaction. A conforming application functions with
no display attached.

It builds on three companion specifications:

- **OVOS-MSG-1** — the Message envelope (`type` / `data` / `context`)
  and its `forward` / `reply` / `response` derivations;
- **OVOS-SESSION-1** — the `session` carrier and the reserved
  `session_id` value `"default"`;
- **OVOS-SESSION-2** — session lifecycle and the client-authority
  rule.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- the **display-decoupling model** (§2) — applications declare display
  intent semantically; render backends are interchangeable;
- the **closed template vocabulary** (§3) — the fixed, curated set of
  `SYSTEM_*` templates and each template's normative session-data
  keys;
- the **wire protocol** (§4) — the `gui.value.set` and `gui.page.show`
  Message shapes, the reserved keys `__from` / `__idle`, and the
  namespace lifecycle topics;
- **addressing and routing** (§5) — the rule that a GUI Message is
  routed solely by its `session_id`;
- the **adapter contract** (§6) — how a render backend is discovered,
  what events it receives, its handler and hook obligations, and
  capability-degradation guidance;
- the **interaction path** (§7) — the events a render backend emits
  back to the originating application;
- **conformance** (§8).

This specification does **not** define:

- the **rendering** of any template — layout, typography, colour,
  animation, the choice of toolkit or markup; these are owned by each
  render backend;
- the **transport, framing, authentication, or encryption** between a
  render backend and its clients — a backend-internal, layer-2
  concern;
- **session lifecycle** — when a session begins, ends, expires, or
  resumes (owned by OVOS-SESSION-2);
- **capability negotiation** as a wire protocol — §6.5 gives
  degradation *guidance*, not a negotiation handshake;
- **media acquisition or playback** — a display template carries only
  the metadata a player UI renders; the audio/video stream itself is
  owned by the media subsystem;
- the **internal API** of any application-side helper — method names,
  argument shapes, and language bindings of a GUI client library are
  non-normative; §9 gives one as an informative example.

---

## 2. Model

### 2.1 What and how are separated

An application declares display intent by naming a **template** from
the closed vocabulary (§3) and supplying that template's
**session data** — a flat key-value map describing the content to
render. The application states *what* (a weather card, a list, a
confirmation); it does not state *how* (no markup, no widget tree, no
layout, no asset bundle for any specific toolkit).

A **render backend** (an *adapter*, §6) receives the template name and
its session data and renders them in whatever technology it embodies.
Multiple adapters render the same intent simultaneously (§6.4):
multi-modal output is the default, not an exception.

The component sitting between the application and the adapters is the
**GUI service**: a state-and-dispatch hub that holds per-session
display state, fans template events out to every installed adapter,
and runs the namespace lifecycle (§4.3). The GUI service performs no
rendering of its own and operates no client transport.

### 2.2 Namespace

Every display intent is owned by a **namespace** — an opaque string
that scopes a stack of display state. By convention the namespace is
the originating application's identifier (its `skill_id`, INTENT-3 §3),
and this specification uses *namespace* and the producing identifier
interchangeably. An adapter receives the namespace with every event so
it can attribute, group, and tear down state per producer.

Within a session, namespaces form a **last-activated-on-top** stack
(§4.3). The top namespace is the one currently presented.

### 2.3 Voice-first invariants

- A render backend is **optional**. When none is installed, the wire
  protocol of §4 is still emitted; it is simply observed by nobody.
  An application **MUST NOT** require a display to function and
  **MUST NOT** block waiting for a GUI-originated event (§7).
- The display **accompanies** a spoken interaction. Templates that
  solicit a response (`SYSTEM_confirm`, `SYSTEM_select`) are visual
  companions to a concurrent spoken prompt; the spoken path **MUST**
  remain sufficient on a display-only or display-less client.

---

## 3. The template vocabulary

### 3.1 A closed set, by design

The template vocabulary is a **deliberately fixed, curated set**. It
is closed *on purpose*: a small, stable vocabulary lets a render
backend style each template once and have every application inherit a
single, consistent, glanceable presentation across a heterogeneous and
varying display surface. The same intent looks coherent whether it
lands on an embedded panel, a phone browser, a desktop window, or a
synthesized face.

This is the curated-surface model: the set does not grow to fit
individual applications. A template earns inclusion by **ubiquity** —
it represents a display need reimplemented across voice applications
generally, not a need specific to one. The set is closed against
ad-hoc extension and **grows only by amendment of this
specification** (a new version under the repository's versioning
policy). An application **MUST NOT** invent template names; a render
backend **MUST NOT** rely on template names outside this section.

### 3.2 Template name form

A template name is a string beginning with the reserved prefix
`SYSTEM_`. The prefix is reserved by this specification: it is the
discriminator the GUI service uses to recognise a conformant template
intent (§4.3). A page name that does not begin with `SYSTEM_` is **not
a template of this specification**; a conforming GUI service **MUST
NOT** dispatch it as a template (§4.3).

### 3.3 Session-data typing rules

Each template below lists its session-data keys. Unless stated
otherwise:

- string-typed keys carry plain text;
- numeric keys carry finite JSON numbers (OVOS-MSG-1 §6);
- an **omitted** optional key means "the render backend supplies its
  own default or omits the element"; a producer **MUST NOT** emit a
  key as JSON `null` to mean "absent" — it omits the key instead;
- an image-bearing key (`image`, `icon`) carries **either** an
  `http(s)` URL **or** a `data:` URI. A producer **MUST NOT** place a
  local filesystem path on the wire; a render backend **MUST NOT** be
  required to read the producer's filesystem (§3.5).

Session data is **cumulative within a namespace**: keys set by earlier
template events persist until the namespace is cleared (§4.3). A
render backend **MUST** treat the session-data map delivered with an
event as the authoritative current state for that namespace and
**MUST NOT** assume a key absent from one event has been deleted
unless the namespace was cleared.

### 3.4 Catalogue

The vocabulary comprises the following templates. "Keys" lists each
template's session-data keys; a key marked *(req)* is required for the
template to render meaningfully, all others are optional.

#### State and feedback

| Template | Keys | Meaning |
|----------|------|---------|
| `SYSTEM_idle` | — | A request to return to the resting state. *What* the resting display shows (clock, wallpaper, now-playing, nothing, …) is owned entirely by the rendering backend — see §6.9. An application **MUST NOT** request this template directly and **MUST NOT** provide a resting/home screen of its own. |
| `SYSTEM_loading` | `label` (string) | An indeterminate progress indication with an optional label. |
| `SYSTEM_status` | `label` (string), `success` (boolean) | A terminal success/failure result. `success` selects the affirmative or negative presentation. |
| `SYSTEM_error` | `label` (string, *req*), `detail` (string) | An error, with a short message and optional detail. |

#### Content primitives

| Template | Keys | Meaning |
|----------|------|---------|
| `SYSTEM_text` | `text` (string, *req*), `title` (string) | A scrollable plain-text view with an optional heading. |
| `SYSTEM_image` | `image` (string, *req*), `caption` (string), `title` (string), `fill` (string: `fit` \| `crop` \| `stretch`), `background_color` (string, hex) | A static image. `fill` is scaling guidance; `image` obeys §3.3 / §3.5. |
| `SYSTEM_animated_image` | same as `SYSTEM_image` | An animated image (e.g. GIF / WebP). Carries identical keys to `SYSTEM_image`. |
| `SYSTEM_list` | `title` (string), `items` (array of `{title (req), subtitle, image}`) | A scrollable list of labelled items; each item carries an optional thumbnail. |
| `SYSTEM_grid` | `title` (string), `items` (array of `{image (req), title}`) | An image-primary tile grid; tiles are equal-weight and the backend chooses the column count. |
| `SYSTEM_table` | `title` (string), `columns` (array of string, *req*), `rows` (array of array, *req*) | A columnar table. Each row **MUST** have the same length as `columns`. |
| `SYSTEM_html` | `html` (string, *req*), `resource_url` (string) | **Discouraged.** A raw-HTML escape hatch (§3.6). |
| `SYSTEM_url` | `url` (string, *req*) | **Discouraged.** A full web-page escape hatch (§3.6). |

#### Media

| Template | Keys | Meaning |
|----------|------|---------|
| `SYSTEM_audio_player` | `title` (string, *req*), `artist`, `album`, `image`, `position` (number, seconds), `duration` (number, seconds; `0` = unknown/streaming), `playing` (boolean) | A now-playing card for audio. Visual only; the stream is owned by the media subsystem (§7.1). |
| `SYSTEM_video_player` | `uri` (string, *req*), `title` (string), `playing` (boolean) | A video surface; the render backend renders the stream. |
| `SYSTEM_media_player` | `media_title`, `media_artist`, `media_album`, `media_image`, `media_uri`, `media_position` (number, ms), `media_duration` (number, ms; `-1` = live), `media_playback_state` (string: `playing` \| `paused` \| `stopped` \| `loading` \| `error`), `media_playlist` (array of `{title, artist, image, uri, duration}`), `media_search_results` (array of `{title, artist, image, uri, skill_id, match_confidence}`), `media_playlist_position` (number) | The unified media-player UI (now-playing, queue, search results). Driven by the media subsystem, not by an ordinary application (§7.1). |

#### Domain cards

These templates are first-class **because they are reimplemented
across voice assistants generally** — ubiquity is the inclusion
criterion (§3.1).

| Template | Keys | Meaning |
|----------|------|---------|
| `SYSTEM_clock` | — | A self-updating clock. Requires no session data; the backend derives the time from the device clock. |
| `SYSTEM_timer` | `end_time` (number, Unix seconds, *req*), `label` (string), `count_up` (boolean) | A self-updating timer. `count_up` false counts down to `end_time`; true counts up from it. The backend derives the displayed value from the device clock — no polling by the producer. |
| `SYSTEM_weather` | `current_temp` (number, *req*), `min_temp` (number), `max_temp` (number), `condition` (string, *req*), `icon` (string), `location` (string) | A weather summary card. |
| `SYSTEM_map` | `latitude` (number, WGS-84, *req*), `longitude` (number, WGS-84, *req*), `zoom` (number, 1–20), `label` (string) | A geographic location; the backend chooses the map provider. |
| `SYSTEM_face` | `sleeping` (boolean) | An avatar face. `sleeping` true is the resting/closed-eyes state, false the awake state. For backends that render a character rather than a screen layout. |

#### Interactive companions

These templates accompany a concurrent spoken prompt (§2.3). They emit
an interaction event when the user acts on the visual element (§7.2);
the spoken path remains sufficient on its own.

| Template | Keys | Meaning |
|----------|------|---------|
| `SYSTEM_confirm` | `question` (string, *req*) | A yes/no companion to a spoken question. |
| `SYSTEM_select` | `prompt` (string), `items` (array of `{label (req), value (req)}`) | A choice companion to a spoken set of options; `value` is the machine-readable token returned on selection. |

### 3.5 Image delivery

An image-bearing key (`image`, `icon`, and the per-item `image` of
list / grid items) carries one of:

- an `http(s)` URL, used as-is by the render backend; or
- a `data:` URI (e.g. `data:image/png;base64,...`), used as-is.

A producer that holds a **local** asset **MUST** resolve it to a
`data:` URI before emission. A producer **MUST NOT** place a bare
filesystem path on the wire, and a render backend **MUST NOT** be
required to mount or read the producer's filesystem. This keeps the
wire self-contained: every adapter receives renderable image content
regardless of where it runs.

### 3.6 Raw-passthrough templates are discouraged

`SYSTEM_html` and `SYSTEM_url` are **escape hatches** that hand a
render backend opaque content (raw markup, or an arbitrary web page)
instead of a semantic intent. They are retained for migration and
edge cases but are **discouraged**: they defeat the consistency the
closed vocabulary exists to provide (§3.1), they cannot be styled or
degraded uniformly, and a render backend that is not a full web
engine cannot honour them. A producer **SHOULD** prefer a semantic
template over `SYSTEM_html` / `SYSTEM_url` wherever one fits, and
**SHOULD NOT** treat raw passthrough as the default path. A render
backend **MAY** decline to render either and **SHOULD** degrade per
§6.5 when it does.

---

## 4. Wire protocol

GUI Messages are ordinary OVOS-MSG-1 envelopes. This section fixes
their `type` and `data`; the `context` (and therefore the `session`
carrier and routing, §5) follows OVOS-MSG-1 and OVOS-SESSION-1
unchanged.

### 4.1 Reserved data keys

Two keys carry protocol meaning inside a GUI Message's `data` and are
reserved by this specification:

- **`__from`** — string — the producing namespace (§2.2). Every GUI
  Message a producer emits **MUST** carry `__from`.
- **`__idle`** — controls how long the activated namespace persists
  (§4.3); meaningful on `gui.page.show`.

Both keys are **protocol metadata, not session data**. The GUI service
**MUST** strip `__from` and `__idle` (and any other reserved `__`-
prefixed key it defines) before delivering session data to an adapter:
an adapter receives content keys only. A producer **MUST NOT** use a
`__`-prefixed key as application session data.

### 4.2 Messages

#### `gui.value.set` — push session data

Emitted by a producer to update its namespace's session data. `data`
is the flat content map plus `__from`.

```json
{
  "type": "gui.value.set",
  "data": {
    "__from": "weather.openvoiceos",
    "current_temp": 22,
    "condition": "Sunny"
  },
  "context": { "session": { "session_id": "default" } }
}
```

On receipt, the GUI service merges the content keys into the named
namespace's cumulative session data (§3.3) and forwards the merged,
reserved-key-stripped map to every adapter as a session update
(§6.3).

#### `gui.page.show` — request a template

Emitted by a producer to present a template. The first entry of
`page_names` **MUST** be a template name (§3.2).

```json
{
  "type": "gui.page.show",
  "data": {
    "__from": "weather.openvoiceos",
    "page_names": ["SYSTEM_weather"],
    "index": 0,
    "__idle": 30
  },
  "context": { "session": { "session_id": "default" } }
}
```

| Field | Type | Meaning |
|-------|------|---------|
| `page_names` | array of string | Templates to present; entry `0` (or the entry at `index`) is the active one. The first entry **MUST** be a `SYSTEM_*` template name. |
| `index` | number | Which entry of `page_names` is the active template. Out-of-range values are clamped to the last entry. |
| `__from` | string | The producing namespace (§4.1). |
| `__idle` | boolean \| number \| omitted | Namespace persistence (§4.3). |

A `gui.page.show` whose first `page_names` entry is **not** a
`SYSTEM_*` template is not a template intent under this specification.
A conforming GUI service **MUST NOT** dispatch it to adapters as a
template; it MAY reject it or route it to a deployment-specific legacy
path outside this specification's scope.

#### `gui.clear.namespace` — clear a namespace

Emitted by a producer when it is done displaying. `data` carries
`__from`. The GUI service removes the namespace from the session's
active stack (§4.3) and notifies adapters of deactivation (§6.3).

### 4.3 Namespace lifecycle

For each session (§5) the GUI service maintains an ordered stack of
active namespaces.

- **Activate.** On a `gui.page.show` whose active template is valid,
  the GUI service ensures the producing namespace exists, dispatches
  the template to every adapter (§6.4), and — if the namespace is not
  already on top — moves it to the top of that session's stack,
  invoking the activation hook on every adapter (§6.3).
- **Persist.** `__idle` controls how long the activated namespace
  stays on top before automatic removal:

  | `__idle` | Behaviour |
  |----------|-----------|
  | `true` | Persistent — stays until cleared. |
  | a number *N* | Visible for *N* seconds, then auto-removed. |
  | omitted / absent | A deployment default applies. |

- **Deactivate / clear.** On `gui.clear.namespace`, on auto-removal by
  the `__idle` timer, or when another namespace supersedes it, the
  namespace is removed from the session's active stack and the
  deactivation hook is invoked on every adapter (§6.3). When the stack
  becomes empty the session returns to its idle presentation.

The lifecycle is **per session**: each `session_id` owns an
independent stack (§5). A namespace activation in one session does not
affect another.

---

## 5. Addressing and routing

### 5.1 `session_id` is the sole routing key

A GUI Message is routed by **one** identifier: the `session_id` in its
`context.session` (OVOS-SESSION-1 §3.1). There is no separate site,
room, location, or audience dimension in this specification. The GUI
service reads the `session_id`, defaulting an absent or empty session
to the reserved value `"default"` per OVOS-SESSION-1 §3.1, and:

- maintains an independent namespace stack per `session_id` (§4.3);
- carries the `session_id` to every adapter on every per-session event
  (§6.3) so the adapter can deliver to the matching client(s).

A render backend's clients each declare the `session_id` they belong
to when they connect (by whatever backend-internal handshake the
backend defines). The backend delivers a per-session event only to the
clients whose declared `session_id` equals the event's `session_id`.
The reserved value `"default"` is **not** a wildcard: a client on
`"default"` receives only `"default"` events.

### 5.2 Shared and multi-room screens

A shared display — several physical screens that should show the same
thing, e.g. a multi-room setup — is expressed entirely at the client
layer: the participating clients **connect with the same
`session_id`**. They then receive the same per-session events and
render identically. This specification needs no group, site, or
location concept; sharing a `session_id` *is* the grouping mechanism.

The common arrangements:

| Arrangement | `session_id` |
|-------------|--------------|
| On-device display (the device's own screen) | `"default"` |
| Shared / multi-room screen group | one shared identifier, used by every screen in the group |
| Standalone remote display (e.g. a phone against a remote server) | that client's own session identifier |

### 5.3 System-wide signals

Some events are not per-session: device-wide status signals (e.g.
wake-word detected, the assistant is speaking) and certain lifecycle
notifications concern the device as a whole. These are delivered to
adapters as **status events** (§6.3) and a render backend **SHOULD**
broadcast them to all of its connected clients regardless of
`session_id`. The `session_id` accompanying a status event is
informational; it is not a delivery filter.

---

## 6. The adapter contract

A **render backend** is an *adapter*: an additive plugin that turns
template intents into a concrete presentation. Adapters are the only
components that render; the GUI service never does.

### 6.1 Discovery

Adapters are discovered through a named **entry-point group**. The GUI
service enumerates every installed adapter in that group at startup and
instantiates each one. The group name and packaging mechanism are
fixed by the platform's plugin-discovery convention and are not
restated normatively here; what this specification fixes is the
behaviour (§6.2–§6.5), not the registration syntax (see §9 for an
informative example).

Adapters are **additive**: installing one adds a render backend
without removing or reconfiguring others. A deployment with **zero**
adapters installed is valid and runs **headless** — the wire protocol
of §4 is still emitted and the GUI service still tracks per-session
state, but dispatch reaches no backend and is a silent no-op. The GUI
service **MUST NOT** fail or refuse to start because no adapter is
installed, and instantiating one adapter **MUST NOT** be able to
prevent others from loading: an adapter that raises during
construction **MUST** be skipped and logged, not allowed to abort
startup.

### 6.2 Construction

An adapter is constructed with its own configuration section and a
handle to the shared Message bus. Construction **MUST** be fast and
non-blocking: adapters are built during GUI-service startup, so a slow
constructor delays the whole subsystem. An adapter that runs a server
or render loop **MUST** start it on a background thread and return
from construction promptly.

### 6.3 Events received

The GUI service delivers events to **every** installed adapter. Each
event is one of:

**Template events** (per-session). Carry `(skill_id, data,
session_id)` where `skill_id` is the producing namespace, `data` is
the namespace's full current session data with reserved keys stripped
(§4.1), and `session_id` is the routing key (§5). One template event
is delivered per active template named in a `gui.page.show`.

**Session-data updates** (per-session). Carry `(skill_id, data,
session_id)` on every `gui.value.set` after the namespace data is
merged (§4.2).

**Lifecycle hooks.** Invoked as the namespace stack changes:

| Hook | When | Routing |
|------|------|---------|
| namespace-activated | a namespace moves to the top of a session's stack | per-session (`skill_id`, `session_id`) |
| namespace-deactivated | a namespace leaves the active stack | per-session (`skill_id`, `session_id`) |
| idle | the display returns to its resting state | device-wide |
| session-update | a `gui.value.set` was applied | per-session (`skill_id`, `data`, `session_id`) |
| status-event | a device-wide status signal occurred (§5.3) | broadcast (`event_name`, `data`, `session_id` informational) |

An adapter **MUST** treat per-session events as addressed to the
clients on the carried `session_id` (§5.1) and **SHOULD** broadcast
status events and device-wide signals to all clients (§5.3).

### 6.4 Multi-modal fan-out

Every installed adapter receives every event. There is no "active
adapter" and no arbitration between adapters: a screen-panel backend,
a browser backend, a character-face backend, and a terminal renderer
installed together all receive the same template event and each
renders it in its own medium. Fan-out is the default behaviour; an application cannot and
need not target a particular adapter.

Because fan-out is unconditional, an adapter **MUST** be isolated from
its peers: a handler or hook **MUST NOT** raise out of the adapter
(§6.6), and an adapter **MUST NOT** mutate shared GUI-service state.
The GUI service wraps every dispatch defensively, but an adapter
**MUST NOT** rely on that wrapping for its own correctness.

### 6.5 Capability and graceful degradation

Adapters are heterogeneous: a two-line character display cannot render
`SYSTEM_video_player`; a face cannot render `SYSTEM_table`; a
terminal cannot render `SYSTEM_map`. Because every adapter receives
every template, each adapter is responsible for handling — or
gracefully declining — every template.

- An adapter **MUST NOT** fail when it receives a template it cannot
  render; it either renders it or degrades (below). Declining is a
  no-op return, never an error.
- An adapter that cannot render a template **SHOULD degrade** to the
  nearest presentation it *can* render, preferring a degraded view
  over showing nothing. RECOMMENDED degradations:

  | Requested | Suggested degradation |
  |-----------|-----------------------|
  | `SYSTEM_video_player` | a still image (`image`/`icon`) plus the title as text |
  | `SYSTEM_media_player` / `SYSTEM_audio_player` | track title + artist as text |
  | `SYSTEM_table` | the rows rendered as a list |
  | `SYSTEM_grid` | the items rendered as a list |
  | `SYSTEM_map` | the `label`, or the coordinates, as text |
  | `SYSTEM_weather` | a one-line text summary |
  | `SYSTEM_html` / `SYSTEM_url` (§3.6) | a text or "content unavailable" placeholder |

- An adapter that supports a template only partially **SHOULD** render
  the keys it understands and ignore the rest, rather than refuse the
  template.

This guidance is degradation behaviour, not a wire negotiation:
nothing in §4 reports an adapter's capabilities back to the producer,
and a producer **MUST NOT** assume any template was actually rendered.

### 6.6 Exception safety and non-blocking

- **Exception-safe.** An adapter **MUST NOT** let an exception escape
  a handler or a hook. It catches and logs internally. An escaping
  exception must never reach the GUI service's dispatch or affect
  another adapter (§6.4).
- **Non-blocking.** Handlers and hooks are invoked on the GUI
  service's dispatch path. An adapter **MUST NOT** perform blocking
  work (network round-trips, disk I/O, sleeps, waiting on a client) on
  that path; it offloads such work to its own background thread or
  queue and returns promptly.
- **No shared-state mutation.** An adapter keeps only its own
  per-`session_id` / per-namespace state and **MUST** release that
  state on namespace-deactivated. It **MUST NOT** mutate GUI-service
  state, and **SHOULD NOT** cache the `data` map across events — it
  re-reads current state from the GUI service's read-only query
  surface (§6.7) when it needs state later.

### 6.7 State query (recovery)

The GUI service **MAY** expose a read-only query surface for an
adapter that must reconstruct current state — for example a client
that connects after a template was already shown. Such queries return
**copies** of current per-session namespace state and **MUST NOT**
mutate it. An adapter using them **MUST** tolerate a `None`/empty
result, since the queried namespace may have been removed between query
and use. The exact query surface is backend-facing and non-normative.

### 6.8 Connection status

An adapter **MAY** report whether it currently has at least one
connected client. The GUI service aggregates these reports to answer a
connectivity query: the subsystem reports "a display is connected" if
**any** installed adapter reports a connected client. An adapter that
does not implement the report is treated as having no connected
client. A producer **MAY** consult this aggregate to decide whether to
bother composing a display, but **MUST NOT** condition correctness on
it (§2.3).

### 6.9 Idle / resting display ownership

The resting display — what is shown when no application is actively
presenting (a clock, wallpaper, now-playing summary, an avatar, or
nothing at all) — is owned **entirely by the render backend**. It is a
property of the surface, not of any application.

- An application **MUST NOT** provide, register, or render a resting /
  home screen, and **MUST NOT** assume one exists. `SYSTEM_idle` is a
  request to *return to* the resting state (§3, §4.3); it carries no
  content and does not let the caller dictate what the resting state
  looks like.
- A render backend **MAY** present any resting display it chooses, or
  none. Two backends driven by the same subsystem **MAY** show entirely
  different idle surfaces.
- There is no subsystem-level "home screen" concept, no home-screen
  registry, and no home-screen application type. A backend that wants a
  configurable or app-like resting surface implements that **privately**,
  behind its own boundary; it is never exposed as an application contract.

This keeps applications free of display ownership: they declare intents
(§2), and the surface decides how to rest.

---

## 7. Interaction (render backend → application)

The subsystem is output-primary; the input path is deliberately
narrow. A render backend emits an event back to the originating
application only for the templates that define one, and only as a
**shortcut** for a path the spoken interaction already covers (§2.3).

### 7.1 Media transport

`SYSTEM_audio_player`, `SYSTEM_video_player`, and `SYSTEM_media_player`
are visual surfaces over a stream owned by the media subsystem. A
render backend's transport controls (play / pause / next / seek) act
on the **media subsystem**, not on the producing application; the
template's session data is a reflection of media state, not a command
channel. The media-control wire surface is owned by the media
subsystem and is out of scope here.

### 7.2 Interactive companions

For the interactive templates (§3.4), when the user acts on the visual
element the render backend **SHOULD** emit an interaction event back to
the originating namespace, carrying the originating `session_id` in the
Message context (so the application can attribute the answer to the
session it asked in):

- **`SYSTEM_confirm`** → an event reporting the boolean answer
  (whether the user confirmed).
- **`SYSTEM_select`** → an event reporting the `value` of the chosen
  item (§3.4).

The originating application **MUST** treat such an event as a
*shortcut*: it registers a handler for it **and** independently
handles the spoken response, and **MUST NOT** block waiting for the
GUI event (§2.3). The exact event topic and payload schema are owned
by the producing-side interface and are non-normative in this version;
what this specification fixes is that the response **MUST** carry its
originating `session_id` so the application can route the answer back
to the correct session.

---

## 8. Conformance

### 8.1 A conforming producer (application-side interface) MUST

- name only templates from the closed vocabulary (§3); never invent
  template names;
- emit `gui.page.show` with a `SYSTEM_*` template as the first
  `page_names` entry (§4.2);
- include `__from` on every GUI Message it emits (§4.1);
- deliver image content as an `http(s)` URL or a `data:` URI, never a
  bare filesystem path (§3.5);
- carry the `session_id` of the interaction in the Message context so
  routing (§5) and interaction-response attribution (§7.2) work;
- omit absent optional keys rather than emit them as `null` (§3.3);
- function with no display attached and never block on a GUI event
  (§2.3).

### 8.2 A conforming producer SHOULD

- prefer a semantic template over the `SYSTEM_html` / `SYSTEM_url`
  escape hatches (§3.6);
- batch related session-data changes into a single `gui.value.set`
  rather than one Message per key;
- clear its namespace (`gui.clear.namespace`) when done displaying
  (§4.2).

### 8.3 A conforming GUI service MUST

- recognise a template intent by the `SYSTEM_` prefix and dispatch
  only `SYSTEM_*` page names as templates (§3.2, §4.2);
- strip reserved `__`-prefixed keys from session data before
  delivering it to adapters (§4.1);
- route by `session_id` alone, defaulting an absent/empty session to
  `"default"` (§5.1), and maintain an independent namespace stack per
  `session_id` (§4.3);
- deliver every template event, session-data update, and lifecycle
  hook to **every** installed adapter (§6.3, §6.4);
- start and operate with zero adapters installed (headless no-op) and
  skip-and-log any adapter that fails to construct, without aborting
  startup (§6.1);
- run no client transport and perform no rendering of its own (§2.1).

### 8.4 A conforming adapter MUST

- be discoverable via the adapter entry-point group (§6.1);
- construct quickly and non-blocking, starting any server/render loop
  on a background thread (§6.2);
- never let an exception escape a handler or hook (§6.6);
- never block the dispatch path and never mutate GUI-service state
  (§6.6);
- treat per-session events as addressed to the clients on the carried
  `session_id`, with `"default"` not a wildcard (§5.1);
- never fail on a template it cannot render — render or decline as a
  no-op (§6.5);
- release per-namespace state on namespace-deactivated (§6.6).

### 8.5 A conforming adapter SHOULD

- degrade an unrenderable template to the nearest renderable
  presentation rather than show nothing (§6.5);
- broadcast status events and device-wide signals to all clients
  regardless of `session_id` (§5.3);
- emit an interaction event carrying the originating `session_id` for
  the interactive templates it presents (§7.2);
- report connection status so the connectivity aggregate is accurate
  (§6.8);
- re-read state from the GUI service's query surface rather than cache
  the `data` map (§6.6, §6.7).

### 8.6 Non-goals

The following are explicitly outside this specification and **MUST
NOT** be inferred from it: the rendering of any template; the
transport, framing, authentication, or encryption between a render
backend and its clients; capability negotiation as a wire handshake;
session lifecycle (OVOS-SESSION-2); media acquisition and the
media-control wire surface; and the internal API, method names, and
language bindings of any application-side GUI helper.

---

## 9. Informative example (non-normative)

The following sketches one application-side helper and one minimal
adapter. Names and language are illustrative; nothing here is
normative.

A producer showing a weather card:

```python
gui["current_temp"] = 22          # emits gui.value.set {__from, current_temp}
gui.show_weather(                 # sets keys, then emits gui.page.show
    current_temp=22, min_temp=14, max_temp=24,
    condition="Sunny", location="Lisbon",
)                                 # page_names=["SYSTEM_weather"]
# ... later
gui.release()                     # emits gui.clear.namespace
```

A minimal terminal adapter that renders two templates and degrades the
rest by ignoring them:

```python
class TerminalAdapter(AbstractGUIAdapter):
    def handle_show_text(self, skill_id, data, session_id="default"):
        print(f"[{skill_id}@{session_id}] "
              f"{data.get('title', '')}: {data.get('text', '')}")

    def handle_show_weather(self, skill_id, data, session_id="default"):
        # degrade the weather card to one line of text
        print(f"[{skill_id}@{session_id}] "
              f"{data.get('location', '')}: "
              f"{data['current_temp']}° {data['condition']}")

    def on_status_event(self, event_name, data, session_id="default"):
        print(f"[status] {event_name}")   # broadcast; ignore session_id

    def any_client_connected(self) -> bool:
        return True
```

Registration via the adapter entry-point group (illustrative):

```toml
[project.entry-points."opm.gui_adapter"]
"my-terminal" = "my_package:TerminalAdapter"
```

---

## See also

- **OVOS-MSG-1** — the Message envelope and the `forward` / `reply` /
  `response` derivations that carry `context.session`.
- **OVOS-SESSION-1** — the `session` carrier, `session_id`, and the
  reserved `"default"` value used as the routing key (§5).
- **OVOS-SESSION-2** — session lifecycle and the client-authority
  rule a remote display relies on.
- **OVOS-INTENT-3** — the `skill_id` an application uses as its
  namespace (§2.2).
