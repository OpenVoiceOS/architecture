# User Identity Resolution Specification

**Spec ID:** OVOS-USER-ID-1 · **Version:** 1 · **Status:** Draft

This specification defines the session fields that carry user identity
and authentication evidence for an utterance. It prescribes what each
field means, who is allowed to write it, how authentication strength is
expressed and how it expires, and what obligations apply to recognition
plugins, bridges, and skills.

It builds on **OVOS-SESSION-1** (field registry), **OVOS-SESSION-2**
(mutation boundaries), **OVOS-TRANSFORM-1** (transformer chain),
**OVOS-BRIDGE-1** (Layer-2 injection), and **OVOS-MSG-1** (session
carrier).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**MAY**, and **RECOMMENDED** are used as in RFC 2119.

---

## 1. Scope

This specification defines:

- **identity fields** (§2) — the session fields carrying resolved
  identity and per-signal evidence, who owns them, and the gate
  invariant that governs inbound values;
- **authentication level** (§3) — a derived integer summarising
  evidence strength, and how it expires;
- **enrolled signals** (§4) — the four data points a user may enroll
  and how they map to session fields;
- **resolution** (§5) — the normative order in which a recognition
  plugin consolidates signals into an identity, and how identity
  persists across utterances;
- **Layer-2 injection** (§6) — what a bridge may and may not supply;
- **skill use** (§7) — the guest baseline and how skills gate behavior
  on authentication level;
- **privacy** (§8) — exposure limits on identity fields;
- **conformance** (§9).

It does **not** define:

- recognition algorithms, models, or biometric processing;
- enrollment procedures or credential storage;
- how audio, video, or other sensor data is acquired;
- how a deployment authenticates the transport a Message arrives on —
  that is a layer-2 concern (**OVOS-MSG-1 §3.4**).

---

## 2. Identity fields

The fields below are claimed under **OVOS-SESSION-1 §2.2**. All are
OPTIONAL on the wire. The "Deployment default" column states the value
a consumer falls back to when the field is omitted, per SESSION-1 §2.2
item 3.

| Field | Type | Deployment default | Meaning |
|-------|------|--------------------|---------|
| `user_id` | string (opaque) | absent — the session is a guest session (§7.1) | The resolved user identity. |
| `speaker_id` | string (opaque) | absent — no behaviour; the signal was not established | ID of the enrolled voice-print record that matched this utterance. |
| `face_id` | string (opaque) | absent — no behaviour; the signal was not established | ID of the enrolled face-print record that matched. |
| `name_id` | string (opaque) | absent — no behaviour; the signal was not established | ID of the enrolled name record that the user's self-declaration matched. |
| `passphrase_id` | string (opaque) | absent — no behaviour; the signal was not established | ID of the enrolled secret-phrase record that matched. |
| `default_user_id` | string (opaque) | absent — no configured default identity applies (§6.1) | The deployer-configured identity for this site or session, used only when no runtime signal resolves. |
| `auth_level` | number (integer-valued) | `0` | Authentication strength (§3). |
| `authenticated_at` | number | absent — the level carries no verifiable freshness (§3.1) | Unix-seconds wall-clock time at which the evidence backing the current `auth_level` was obtained. MAY be a float; consumers MUST accept integer and float forms. |

`user_id` is the **consolidated** identity resolved from one or more
per-signal fields per §5.1. Per-signal fields (`speaker_id`, `face_id`,
`name_id`, `passphrase_id`) MAY be set even when `user_id` cannot be
resolved — they record what matched, not that a user was identified. An
absent `user_id` with a present per-signal field means recognition ran
but did not produce a confirmed identity.

`speaker_id` names the **enrolled speaker** whose voice print matched.
It is unrelated to **OVOS-TRANSFORM-1 §3.5**'s `voice_id`, a
`Message.context` hint naming the *synthetic* voice a downstream TTS
transformer should render with. The two never refer to the same thing:
`speaker_id` describes who spoke, `voice_id` describes how the
assistant answers.

### 2.1 The gate invariant

Identity fields arriving on an inbound Message are **unauthenticated
hints**. Any participant that can place a Message on the bus can assert
any value for them; the session object is client-authoritative
(**OVOS-SESSION-2 §2.5**) and carries no proof of its own contents.

Therefore, for every inbound Message from a governed participant, the
recognition plugin — or, in a managed deployment where the bridge
itself performs recognition, the bridge — **MUST**:

1. **re-derive** each identity field it is able to derive for this
   utterance, and **overwrite** whatever value the inbound session
   carried for that field; and
2. **clear** every identity field it did not itself derive, including
   `user_id`, `auth_level`, and `authenticated_at`.

An inbound value is never carried through on the strength of having
been present. This is the gate invariant of **OVOS-BRIDGE-1 §4.1**
applied to identity: applying identity once, at connect time or on the
first utterance, is not conformant, because the participant may send a
session asserting any identity on any subsequent Message. A gate is not
a handshake.

`default_user_id` is exempt from step 2 only when it is supplied from
deployer configuration rather than from the inbound session (§6.1).

Identity persistence across utterances is not an exception to this
rule: a plugin that keeps identity across an utterance boundary does so
from **its own** retained state keyed on `session_id`, never from the
values the inbound session asserted (§5.3).

### 2.2 Ownership and mutation boundary

Every field in §2 is owned by the **user recognition plugin**. No other
component writes them, with the single narrow exception of a bridge
acting within the limits of §6.

The plugin mutates them at a **transformer boundary** — one of the
mutation boundaries enumerated in **OVOS-SESSION-2 §2.6**. Typically
this is the metadata-transformer hook (**OVOS-TRANSFORM-1 §3.3**),
which runs once per utterance before the pipeline. Writing identity
fields outside a §2.6 boundary is not conformant: the mutation is not
guaranteed to reach the utterance it was meant to describe.

---

## 3. Authentication level

`session.auth_level` summarises the strength of evidence backing
`session.user_id` for the current utterance.

| Level | Evidence |
|-------|---------|
| `0` | Anonymous — `user_id` absent. Per-signal fields MAY still be set (matched but not consolidated). |
| `1` | Configured default — `default_user_id` was used; no runtime signal resolved. |
| `2` | Self-declared — `name_id` matched; the user stated their identity, unverified. |
| `3` | Single passive biometric — exactly one of `speaker_id` or `face_id` matched. |
| `4` | Corroborated passive biometrics — `speaker_id` and `face_id` independently matched the same `user_id`. |
| `5` | Active credential — `passphrase_id` matched, with or without additional signals. |

The recognition plugin **MUST** set `auth_level` to `0` when `user_id`
is absent. When `user_id` is present, it **MUST** set `auth_level` to
the highest level whose criteria are met, and **MUST** set
`authenticated_at` to the time the evidence for that level was
obtained.

Skills **MUST** treat an absent `auth_level` as `0`.

### 3.1 Level lifetime and decay

An authentication level is evidence about a moment, not a property of
the session. A level of `3` or above **MUST** decay after a
deployer-defined TTL measured from `authenticated_at`. On decay, the
recognition plugin **MUST** lower `auth_level` to the strongest level
whose evidence is still valid — in practice the strongest *passive*
level it can re-derive for the current utterance, or `1` if only
`default_user_id` remains, or `0` if nothing does — and **MUST** update
`authenticated_at` accordingly.

A single TTL value is not prescribed. A deployment SHOULD use a shorter
TTL for level `5` than for levels `3` and `4`, because an active
credential attests to a single utterance while a passive biometric
attests to a presence.

A consumer that reads `auth_level` ≥ 3 with `authenticated_at` absent
cannot evaluate freshness. Such a consumer **SHOULD** treat the level
as expired and gate as though it were the highest level it can verify.

### 3.2 The default session

On a Message whose `session_id` is the reserved value `"default"`
(**OVOS-SESSION-1 §3.1**), `auth_level` **MUST** be capped at `1`.

The default session is the device-local session: it is persistent,
shared by every occupant of the physical device, and addressable by any
remote participant that chooses to impersonate it. Evidence obtained
from one occupant cannot be attributed to the next speaker on a shared
session. A recognition plugin MAY still resolve and write `user_id` and
the per-signal fields on the default session; it MUST NOT report a
level above `1` there.

---

## 4. Enrolled signals

A user may enroll any combination of four signal types. Each maps to
one session field. Enrollment procedures are out of scope; this section
defines only the runtime semantics.

### 4.1 Voice print — `speaker_id`

A voice-print recognizer (typically an audio transformer,
**OVOS-TRANSFORM-1 §3.1**) compares the utterance audio against
enrolled voice prints. On a match it writes the opaque `speaker_id` of
the matching enrollment record.

### 4.2 Face print — `face_id`

Face recognition operates outside the utterance lifecycle — for
example, a camera sensor plugin running a continuous recognition loop.
Because it is not triggered by the utterance, it is less temporally
precise than inline signals: it attests that a face was seen near the
device, not that the face spoke.

`session.site_id` (**OVOS-BRIDGE-1 §3.3**) SHOULD be used to select the
camera associated with the physical location where the utterance
originated, so the correct feed is queried in multi-device deployments.

> **Note (non-normative):** whether a face recognizer distinguishes a
> present person from a photograph or a replayed video is a property of
> the recognizer, not of this specification. A deployment that cannot
> make that distinction gets a weaker `face_id` than one that can; the
> level table (§3) does not model the difference, so a deployer who
> cares about it configures a shorter TTL (§3.1) or requires an active
> credential (level `5`) for the operations that matter.

### 4.3 Name — `name_id`

A self-declaration recognizer (typically an utterance transformer,
**OVOS-TRANSFORM-1 §3.2**) detects identity assertions in the
transcript ("I am Alice", "You're talking to Bob") and matches against
enrolled names. On a match it writes the opaque `name_id` of the
matching enrollment record.

If no names are enrolled, or the declared name matches no enrollment,
`name_id` is left absent. A matched name is unverified — the user
stated who they are and presented no credential — and **SHOULD NOT** be
the sole basis for any operation requiring verified identity.

### 4.4 Secret phrase — `passphrase_id`

A passphrase recognizer (typically an utterance transformer) detects a
secret phrase in the transcript and matches against enrolled
credentials. Because the user actively produced the credential,
`passphrase_id` alone is sufficient for `auth_level` `5` regardless of
which passive signals are also present.

---

## 5. Resolution

A **user recognition plugin** writes the identity fields into the
session. For each utterance it:

1. reads the per-signal evidence available for **this** utterance —
   from `Message.context`, from its own retained per-session state, and
   from out-of-band sources such as a camera feed — and **ignores** the
   identity fields the inbound session asserts (§2.1);
2. resolves each signal to its enrollment record ID;
3. consolidates the resolved signals into `user_id` per §5.1;
4. computes `auth_level` per §3 and sets `authenticated_at`;
5. writes what it derived and clears what it did not (§2.1), at a
   §2.6 mutation boundary (§2.2).

How the plugin is implemented — as a metadata transformer, as a
standalone service, or as a combination — is deployer-defined. The
normative constraints are the gate invariant (§2.1), the resolution
order (§5.1), and that the fields are present in `context.session` by
the time the utterance enters the pipeline.

### 5.1 Resolution order

When more than one signal resolves, the plugin **MUST** consolidate
`user_id` by taking the identity named by the strongest signal
available, in this order:

1. `passphrase_id` — an active credential the user produced;
2. `speaker_id` **and** `face_id` agreeing on one identity —
   corroborated passive biometrics;
3. a single passive biometric — `speaker_id` or `face_id`;
4. `name_id` — a stated name;
5. `default_user_id` — the deployer-configured identity (§6.1).

Ordering is by evidence class, not by a numeric score: this
specification defines no confidence field, and a plugin's internal
scores are not on the wire. Within one class a plugin uses its own
matching logic, and if that logic does not clear its threshold the
signal does not resolve at all and the next class applies.

### 5.2 Disagreement

When two or more **resolved** signals name **different** identities,
the plugin **MUST NOT** set `user_id`, and **MUST** set `auth_level`
to `0`. It MAY still write the per-signal fields — they record what
matched — and a skill reading them learns that recognition was
contradictory rather than absent.

Disagreement is not resolved by preferring the stronger signal. Two
signals naming different people means at least one of them is wrong,
and the specification does not offer a rule for guessing which. A
deployment that wants an identity here obtains an active credential
and re-runs resolution (§5.4).

`default_user_id` never participates in disagreement: it is outranked
by every runtime signal (§6.1), so a runtime signal that contradicts it
simply wins.

### 5.3 Identity across utterances

A recognition plugin MAY retain resolved identity in **its own** state,
keyed on `session_id`, and re-apply it to a later utterance on the same
session without re-running recognition — subject to the decay rule of
§3.1 and the default-session cap of §3.2.

This is not preservation of inbound fields. The plugin re-applies what
it previously derived; it never adopts what the session asserts. A
plugin MAY raise `auth_level` when a later utterance produces stronger
evidence, and **MUST** lower it when its evidence decays or when a new
signal disconfirms the retained identity.

### 5.4 Re-authentication

When a skill requires a higher `auth_level` than the session carries,
it prompts the user and captures the answer through
**OVOS-CONVERSE-1 §5** response mode. Per CONVERSE-1 §5.1 the handler
sets `session.response_mode` to `{skill_id, expires_at}` from within
its own dispatched handler, and the `ovos.utterance.speak` posing the
prompt MUST carry `listen: true`. Delivery of the answer happens
**inside the pipeline**: the converse plugin, positioned at the front
of `session.pipeline`, pre-empts normal matching and dispatches
`<skill_id>:response` (CONVERSE-1 §5.2).

The answering utterance traverses the recognition plugin like any
other, so it is subject to §2.1 and §5.1 in full. If the new
`auth_level` meets the requirement the skill proceeds; if it does not,
the skill declines.

---

## 6. Layer-2 injection

A Layer-2 bridge (**OVOS-BRIDGE-1 §4.1**) sits at a boundary where it
may know something about the participant that no recognizer can see —
a network-layer login, a chat account, a terminal provisioned for one
household member.

Such a bridge **MAY** set:

- `user_id`, and
- `default_user_id`, and
- `auth_level`, **capped at `1`**.

Such a bridge **MUST NOT** set `speaker_id`, `face_id`, `name_id`,
`passphrase_id`, `authenticated_at`, or an `auth_level` above `1`. A
per-signal field asserts that a specific enrolled record matched a
specific recognizer. A bridge that ran no recognizer has nothing to
assert, and a fabricated per-signal field is indistinguishable
downstream from a real one — it would let a transport-level login
present itself to skills as a biometric match.

A bridge that **does** run a recognizer is, for these fields, acting as
a recognition plugin, and is bound by §5 in full — including the gate
invariant, the resolution order, and the decay rule. The role, not the
process boundary, determines the obligations.

Whatever a bridge sets is itself subject to §2.1 downstream: a
recognition plugin later in the chain re-derives and overwrites.

### 6.1 `default_user_id`

`default_user_id` is **deployer configuration**, not a runtime signal:
"this site or session belongs to this user, absent any other evidence."
Its authoritative source is the deployment's own configuration.

The recognition plugin **MAY** mirror the configured value onto the
session so that skills and observers can see which default applied. No
other component writes it, and a value that arrives on an inbound
session from a governed participant carries no authority — the plugin
replaces it with the configured value or clears it.

Any runtime signal that resolves outranks `default_user_id` (§5.1). A
plugin that resolves no runtime signal and finds a configured default
sets `user_id` to it with `auth_level` `1`.

---

## 7. Skill use

### 7.1 Guest baseline

Skills and pipeline plugins **MUST NOT** fail or error when
`session.user_id` is absent, and **SHOULD** treat an absent `user_id`
(equivalently, `auth_level` `0`) as a guest session. Every feature that
does not need an identity works without one.

### 7.2 Gating on level

Skills gate sensitive operations on `session.auth_level`. The table
below is informative guidance, not requirements; the threshold for any
given operation is the skill author's choice.

| `auth_level` | Suitable for |
|-------------|--------------|
| 0 | Fully anonymous features: weather, timers, general knowledge. |
| ≥ 1 | Personalised features based on a configured profile. |
| ≥ 2 | Low-trust personal features: preferences, non-sensitive reminders. |
| ≥ 3 | User-specific data: contacts, calendar, media history. |
| ≥ 4 | Sensitive personal data: private notes, location history. |
| 5 | High-trust operations: financial transactions, access control, privileged commands. |

A skill that requires a minimum level the session does not meet
**SHOULD** prompt for stronger evidence (§5.4) rather than failing
silently or returning another user's data.

### 7.3 Examples (non-normative)

**Guest-safe feature.** The handler reads no identity field:

```
handler(message):
    speak("It is currently 22°C and sunny.")
```

**Personalised feature.** `user_id` at any level is enough; the risk of
the wrong radio station is low:

```
handler(message):
    user_id = message.context["session"].get("user_id")
    if not user_id:
        speak("I don't know who you are yet.")
        return
    speak(f"Playing {load_preferences(user_id).favourite_station}.")
```

**Sensitive data.** A biometric match is required; a stated name
(level 2) or a configured default (level 1) is not enough:

```
handler(message):
    session = message.context["session"]
    if session.get("auth_level", 0) < 3:
        speak("I need to recognise your voice before reading your messages.")
        return
    speak(summarise(fetch_messages(session["user_id"])))
```

**High-trust operation, with re-authentication.** The handler enters
response mode per §5.4 — it mutates the session and its prompt carries
`listen: true` — so the passphrase arrives at `<skill_id>:response`
after the recognition plugin has re-derived identity for that utterance:

```
handler(message):
    session = message.context["session"]
    if session.get("auth_level", 0) < 5:
        session["response_mode"] = {"skill_id": self.skill_id,
                                    "expires_at": now() + 30}
        speak("Please say your secret phrase to authorise this transfer.",
              listen=True)          # forwarded, carries the mutated session
        return
    execute_transfer(session["user_id"], amount, recipient)

response_handler(message):
    session = message.context["session"]
    if session.get("auth_level", 0) < 5:
        speak("That did not authorise the transfer.")
        return
    execute_transfer(session["user_id"], amount, recipient)
```

**Specific-signal requirement.** Some operations cannot be expressed as
a level threshold — here physical access needs both a face and a
passphrase regardless of the consolidated level:

```
handler(message):
    session = message.context["session"]
    if not (session.get("face_id") and session.get("passphrase_id")):
        speak("Face and passphrase both required to unlock.")
        return
    unlock_door(session["user_id"])
```

**Configured default.** A terminal provisioned for one household
member. The bridge sets `default_user_id` and `auth_level: 1`; no
recognizer runs, so nothing above level 1 is available on this device:

```
# inbound session: {"user_id": "alice", "default_user_id": "alice",
#                   "auth_level": 1}

handler(message):
    session = message.context["session"]
    if session.get("auth_level", 0) >= 1:
        speak("Good morning, setting your usual temperature.")
```

---

## 8. Privacy

Identity fields describe a person. They travel on every Message that
carries the session, so their exposure is the exposure of the session
itself.

- The per-signal fields (`speaker_id`, `face_id`, `name_id`,
  `passphrase_id`) and `authenticated_at` **SHOULD NOT** cross a bridge
  boundary. They exist so that the components inside one deployment can
  reason about evidence; a remote peer has no use for them and cannot
  verify them. Of the identity fields, only `user_id` and `auth_level`
  **SHOULD** be relayed outward.
- Every id in §2 **MUST** be **opaque**: a consumer MUST NOT parse it
  or ascribe structure to its value beyond string equality.
- Every id **MUST** be **per-deployment**. The same person enrolled in
  two deployments gets two unrelated ids, so that ids observed in one
  deployment do not correlate with another.
- An id **MUST NOT** be derived from a biometric template, and
  **MUST NOT** be usable as one. `speaker_id` and `face_id` name an
  enrollment record; they never encode, hash, or embed the voice print
  or face print itself. A leaked session must not leak a biometric.

---

## 9. Conformance

### A user recognition plugin **MUST**:

- re-derive and overwrite the identity fields it can derive, and clear
  those it did not derive, on **every** inbound Message from a governed
  participant (§2.1);
- write the fields at an OVOS-SESSION-2 §2.6 mutation boundary (§2.2);
- consolidate `user_id` in the order of §5.1;
- leave `user_id` absent and set `auth_level` `0` when resolved signals
  disagree (§5.2);
- set `auth_level` `0` when `user_id` is absent, and otherwise the
  highest applicable level with `authenticated_at` (§3);
- decay `auth_level` ≥ 3 to the strongest still-valid level after the
  deployer-defined TTL (§3.1);
- cap `auth_level` at `1` on `session_id: "default"` (§3.2);
- leave `user_id` absent rather than use a sentinel value when identity
  cannot be resolved (§2, §7.1).

### A Layer-2 bridge **MUST**:

- set no `auth_level` above `1` (§6);
- set no per-signal field or `authenticated_at` it did not derive from
  a recognizer it ran (§6);
- meet every recognition-plugin requirement above for any field it does
  derive from a recognizer it ran (§6).

### Skills and pipeline plugins **MUST**:

- treat an absent `user_id` as a guest session (§7.1);
- treat an absent `auth_level` as `0` (§3);
- never fail or error on absent identity fields (§7.1).

---

## See also

- **OVOS-SESSION-1** — claims the §2 fields under its §2.2 registry
  mechanism; §3 rosters them; §3.1 defines the reserved `"default"`
  session capped by §3.2 here.
- **OVOS-SESSION-2** — §2.5 client authority, which the §2.1 gate
  invariant answers; §2.6 mutation boundaries, at which §2.2 requires
  the fields to be written.
- **OVOS-BRIDGE-1** — §4.1 states the gate invariant this
  specification applies to identity; §6 bounds what a bridge may
  inject; §3.3 defines `site_id`, used to select a camera in §4.2.
- **OVOS-TRANSFORM-1** — §3.1 audio and §3.2 utterance transformers as
  signal injection points, §3.3 metadata transformer as the usual
  consolidation point; §3.5's `voice_id` TTS hint is a different field
  from §2's `speaker_id`.
- **OVOS-CONVERSE-1** — §5 response mode, the mechanism §5.4 uses to
  collect a re-authentication utterance.
- **OVOS-MSG-1** — §3.4 layer-2 extension point on `source` /
  `destination`, where transport-level authentication attaches; §4 the
  session carrier.
- **OVOS-AUDIO-IN-1** — the audio-transformer chain in which
  voice-print recognition runs.
