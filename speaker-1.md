# Speaker Enrollment Bus Contract Specification

**Spec ID:** OVOS-SPEAKER-1 · **Version:** 1 · **Status:** Draft

This specification defines the bus surface by which an administration
client enrolls a speaker, lists the enrolled speakers, deletes one, and
asks a one-shot verification question: four topics, their payloads,
their replies, the error vocabulary, and the gate that keeps the two
mutating topics closed by default.

A speaker verifier answers one question for the audio input service:
does this wake word come from a voice the household enrolled. It answers
it from a roster of enrolled profiles. Today that roster is built by a
command line tool over a shell account on the device. A control panel
has no shell, so it needs a bus surface, and a bus surface changes who
can add a trusted voice.

Dependencies: OVOS-MSG-1 (envelope, the `response` derivation, and the
opacity of `source`).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED**, **MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines: the four request topics and their replies,
the request and reply payloads, how enrollment audio travels, the
enablement gate over the two mutating topics, the error vocabulary, the
fan-out a client must expect, and what a verification answer says when
the roster is empty.

It does **not** define: how a profile is computed or stored, the
embedding model, the similarity measure, the acceptance threshold, where
a verifier runs, how the audio input service consults one
(OVOS-AUDIO-IN-1), audio capture, or any authorization beyond the single
gate of §5. Authorization of who may enroll is a layer-2 concern
(OVOS-MSG-1 §3.4).

---

## 2. The topics

| Topic | Meaning | Mutating |
|-------|---------|----------|
| `ovos.speaker.enroll` | Add or replace the profile of one named speaker. | yes |
| `ovos.speaker.delete` | Remove the profile of one named speaker. | yes |
| `ovos.speaker.list` | Name the enrolled speakers. | no |
| `ovos.speaker.verify` | Say which enrolled speaker, if any, one clip matches. | no |

All four are static dotted strings fixed by this specification, as
OVOS-MSG-1 §2.1.1 requires: none contains a `:`, and none embeds a
runtime identifier. A speaker name is a payload field, never a topic
component, because "nothing about a Message's target travels in a dotted
topic" (§2.1.1) and a name is not one of the two enumerable
runtime-shaped patterns that section admits.

A component that holds a roster and subscribes to any of the four
**MUST** subscribe to all four. A client cannot discover a partial
surface, and a request answered by silence is indistinguishable from an
absent service (§6).

---

## 3. Replies

Every request is answered by its `response` in the sense of OVOS-MSG-1
§5.3: the reply topic is the request topic suffixed with `.response`,
and the routing pair is the `reply` swap of §5.2. A responder **MUST**
derive the reply from the request rather than build it from its own
identity, so that `source`, `destination` and `session` follow §5.2 and
§4.1 without a separate rule here.

| Request | Reply |
|---------|-------|
| `ovos.speaker.enroll` | `ovos.speaker.enroll.response` |
| `ovos.speaker.delete` | `ovos.speaker.delete.response` |
| `ovos.speaker.list` | `ovos.speaker.list.response` |
| `ovos.speaker.verify` | `ovos.speaker.verify.response` |

A responder **MUST** answer every request it accepts, including one it
refuses. §5 and §7 say so for the two cases where silence is tempting.

---

## 4. Payloads

### 4.1 `ovos.speaker.enroll`

Request `data`:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `name` | string | yes | The speaker this profile belongs to. |
| `clips` | list of string | yes | The enrollment audio, §4.5. |

Reply `data`:

| Field | Type | Meaning |
|-------|------|---------|
| `name` | string | The name from the request. |
| `enrolled` | boolean | Whether the roster now holds the profile. |
| `clips` | integer | How many clips were used. |
| `error` | string or null | The error vocabulary of §7, or null. |

Enrollment of a name that is already enrolled **MUST** replace that
profile rather than fail, and the reply is the same. A client that wants
"add only" reads `ovos.speaker.list` first.

### 4.2 `ovos.speaker.delete`

Request `data`: `name`, a string, required.

Reply `data`: `name`, `removed` (boolean: whether a profile was there to
remove), and `error`.

Deleting a name that is not enrolled is not an error. `removed` is
`false` and `error` is null: the roster is in the state the client
asked for.

### 4.3 `ovos.speaker.list`

Request `data`: empty.

Reply `data`: `speakers`, a list of the enrolled names, and `error`. An
empty roster is an empty list, not an error.

### 4.4 `ovos.speaker.verify`

Request `data`: `clip`, one string, §4.5.

Reply `data`:

| Field | Type | Meaning |
|-------|------|---------|
| `speaker` | string or null | The best matching enrolled name, or null. |
| `score` | number or null | The similarity of that match, or null. |
| `accepted` | boolean | Whether the deployment accepts this voice. |
| `roster_empty` | boolean | Whether no profile is enrolled at all. |
| `error` | string or null | The error vocabulary of §7, or null. |

`accepted` is the answer the verifier would give the audio input
service, not a restatement of `score` against a fixed number: a
deployment that accepts every voice while the roster is empty answers
`accepted: true` with `speaker: null` and `roster_empty: true`, and one
configured to refuse until somebody is enrolled answers `accepted:
false` for the same roster. A client **MUST** read `roster_empty` rather
than infer an empty roster from a null `speaker`, which also means "the
roster holds nobody who matches".

`ovos.speaker.verify` answers a question. It **MUST NOT** change the
roster, and it **MUST NOT** be treated as the audio input service's own
verification path, which needs no bus round trip.

### 4.5 How audio travels

A clip is a base64 string of a complete WAV file, as defined by RFC 4648
§4 with padding.

A responder **MUST** reject a clip that is not a readable WAV with the
`bad clip` error rather than enrolling a profile from whatever the bytes
decode to.

Enrollment audio is large for a bus payload: five seconds of 16 kHz
16-bit mono is about 160 kB, about 213 kB once base64 encoded, and a
useful enrollment is several such clips. A deployment **SHOULD** bound
the total `data` size it accepts and answer `clip too large` beyond it,
and a client **SHOULD** send the fewest clips that enroll well rather
than the most it has.

This specification does **not** define a handle, a path or an upload as
an alternative carrier. One belongs here if the inline bound proves too
small in practice; until then a single carrier keeps every responder
interchangeable.

Opening a microphone to record enrollment audio is **not** part of this
surface. Capture belongs to the audio input service, and a verifier that
recorded its own audio would be a capture client with a second claim on
the device's microphone.

---

## 5. Enablement

`ovos.speaker.enroll` and `ovos.speaker.delete` are gated by a
deployment configuration key, and the RECOMMENDED default is
**disabled**. `ovos.speaker.list` and `ovos.speaker.verify` are not
gated: neither changes the roster.

A responder whose gate is closed **MUST NOT** change the roster, and
**MUST** answer the request with the `enrollment disabled` error rather
than staying silent. A silent refusal is indistinguishable from an
absent service and leaves a client waiting out its own timeout for an
answer that will never come.

The gate exists because of what the bus is. A speaker verifier exists to
refuse a wake word that does not come from an enrolled voice. An
enrollment topic means anything that can publish on the bus can add a
trusted voice, and a delete topic means anything on the bus can remove
one. OVOS-MSG-1 §3.4 makes `source` and `destination` opaque strings a
consumer compares by equality, so a responder cannot tell an
administration client from any other publisher, and **MUST NOT** try to:
a check against `source` is a check against a string the caller chose.

A deployment that opens the gate has decided that every component on its
bus is trusted to name a household member. That is a deployment's
decision to make and this specification does not forbid it; the default
is closed so that the decision is taken rather than inherited.

---

## 6. Fan-out

One roster is one responder. A deployment that runs two components
holding rosters — an audio input service on the device and a second
verifier elsewhere — produces two replies to every request, and the
client **MUST NOT** treat the first as the outcome.

This specification defines no payload field that addresses one responder
among several, because no deployment is known to run two. A field in the
shape of OVOS-INSTALL-1 §2.2 is the way to add one when a deployment
does.

A request that no component answers leaves the client waiting. A client
**MUST** bound its own wait.

---

## 7. Errors

`error` is null when the request succeeded, and otherwise one of:

| `error` | Meaning |
|---------|---------|
| `enrollment disabled` | The gate of §5 is closed. |
| `bad request` | A required field is absent, or has the wrong type. |
| `bad clip` | A clip is not a readable WAV. |
| `clip too large` | The audio exceeds the bound the deployment accepts. |
| `no audio` | `clips` is empty, or `clip` is absent. |
| `internal error` | The responder failed for a reason the client cannot act on. |

A responder **MUST** use these strings exactly for these cases, so that
a client can act on the value rather than display it. A responder
**MAY** add a human-readable `message` field beside `error`.

A reply carrying an `error` **MUST** report the outcome fields honestly:
`enrolled: false` on a failed enrollment, not `true` with an error
beside it.

---

## 8. Conformance

### A speaker roster responder **MUST**:

- subscribe to all four topics of §2 when it subscribes to any (§2);
- answer every request it accepts with the `response` derivation of
  OVOS-MSG-1 §5.3, derived from the request (§3);
- refuse a gated request with `enrollment disabled` rather than silence
  (§5);
- keep the gate over `ovos.speaker.enroll` and `ovos.speaker.delete`
  closed unless the deployment opened it (§5);
- leave the roster unchanged on `ovos.speaker.list` and
  `ovos.speaker.verify` (§4.3, §4.4);
- reject an unreadable clip with `bad clip` (§4.5);
- report `roster_empty` on every `ovos.speaker.verify.response` (§4.4);
- use the error strings of §7 exactly (§7).

### A speaker roster responder **MUST NOT**:

- decide who may enroll by reading `source` (§5, OVOS-MSG-1 §3.4);
- treat enrollment of an enrolled name as an error (§4.1);
- treat deletion of an absent name as an error (§4.2);
- open a microphone to acquire enrollment audio (§4.5).

### A client **MUST**:

- bound its own wait for a reply (§6);
- read `roster_empty` rather than infer it from a null `speaker` (§4.4).

### A client **SHOULD**:

- send the fewest enrollment clips that enroll well (§4.5).

---

## See also

- OVOS-MSG-1 — the envelope, the `response` derivation (§5.3), and the
  opacity of the routing pair (§3.4).
- OVOS-INSTALL-1 — the same gate shape over a mutating bus surface (§5),
  and the addressing field a multi-responder deployment would need
  (§2.2).
- OVOS-AUDIO-IN-1 — the audio input service, which consults a verifier
  without this surface.
