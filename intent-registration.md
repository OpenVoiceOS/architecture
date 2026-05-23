# Intent and Entity Registration and Dispatch Bus Contract

**Spec ID:** OVOS-INTENT-4 · **Version:** 1 · **Status:** Draft

This document defines the **bus messages** that move an intent from a skill's
declaration to an executed handler. It covers three phases of the intent
lifecycle on the bus:

- **registration** — a skill submits its keyword intents, template intents,
  and entity hints to the host;
- **match** — the host announces which intent an utterance triggered;
- **dispatch** — the host invokes the bound handler over the bus and the
  skill reports the outcome.

It is the bus-level companion to OVOS-INTENT-3: where that specification
defines *what* an intent is, this one defines *how* an intent crosses the
wire from declaration to execution.

It builds on three companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope every
  message defined here travels in, the `destination` routing key, the
  `session` carrier (with its reserved `session_id == "default"` and
  its `lang`), and the `forward` / `reply` / `response` derivations;
- the *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  concept, identity model, the two definition methods, and the match
  result that this spec carries on the bus;
- the *Locale Resource Formats Specification* (OVOS-INTENT-2) and the
  *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the
  file-or-inline contract and grammar referenced by each registration
  payload.

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY**
are used as in RFC 2119.

---

## 1. Scope

This specification defines a small fixed set of message topics:

- the two **intent registration** messages — one per definition method
  (OVOS-INTENT-3 §2);
- the **entity registration** message — the `.entity` value-set hint
  (OVOS-INTENT-3 §5.2);
- the **deregistration** messages, for one intent, one entity, or a whole
  skill;
- the **enable** and **disable** messages — temporary suppression of a
  registered intent without losing its definition;
- the **introspection** messages — list and describe registrations;
- the **match-result** message — what the host emits when an intent matches
  an utterance;
- the **dispatch** message — how the host invokes the handler bound to a
  matched intent;
- the **handler-lifecycle** trio — how the skill reports that the handler
  has started, completed, or errored.

It does **not** define:

- the intent concept itself — see OVOS-INTENT-3;
- how the host or an engine *implements* registration storage, indexing, or
  matching — those are implementation choices;
- the **handler reference** itself (§9): the binding between a registered
  intent and the code that runs it is local to the skill and never crosses
  the bus; only the dispatch message and the handler-lifecycle messages do;
- **session lifecycle** — `session` is carried opaquely per
  OVOS-MSG-1;
- **language fallback** — when no registration matches the utterance
  language exactly. Fallback policy is a host concern, expected to be
  formalized in a future specification together with text normalization
  (OVOS-INTENT-1 §5.3 and the planned text-normalization spec);
- legacy topics from earlier Mycroft- and OVOS-derived code paths. These
  are collected in *Appendix A* as a non-normative implementer aid only.

---

## 2. Architectural model — host as the single consumer

This specification adopts a **host-mediated** model. There is exactly one
**host** per assistant; it is the **sole** bus consumer of every topic
defined here that originates from a skill. The host **MAY** delegate
matching to multiple intent engines (OVOS-INTENT-3 §6.2), but engines do
**not** subscribe to bus topics defined here. They receive registrations and
return match results through a host-internal interface whose form is not
specified.

This model has three consequences:

- **Engines are pluggable below the bus.** Adding, removing, or replacing an
  engine is a host-local change; no skill or external observer sees it on
  the bus.
- **Method-rejection is host-internal.** When a host has no engine accepting
  the registration's method, the host **MUST** reject the registration with
  `error_code: "no_compatible_engine"` (§3.2). Engines themselves never emit
  rejections on the bus.
- **One match per utterance is host-enforced.** When multiple engines
  produce candidate matches for the same utterance, the host selects at most
  one (OVOS-INTENT-3 §6.2) before emitting `ovos.intent.matched`.

---

## 3. Identity, responses, and error codes

### 3.1 Identity carried by every registration message

Every registration message carries the identity of what is being registered,
inside the Message's `data` (OVOS-MSG-1 §2.2). The identity fields restate
OVOS-INTENT-3 §3 at the bus layer.

For an **intent**:

| Field | Type | Required | Source |
|-------|------|----------|--------|
| `skill_id` | string | yes | INTENT-3 §3 — assistant-unique. |
| `intent_name` | string | yes | INTENT-3 §3 — unique within the skill. |
| `lang` | string | yes | BCP-47 (INTENT-2 §2). An intent is defined per language (INTENT-3 §3). This is `data.lang` — the language of the resource being registered, distinct from `session.lang` which is the user's preferred language (OVOS-MSG-1 §4.2). |

The triple `(skill_id, intent_name, lang)` is the **registration key**
(INTENT-3 §6.1). Registering an intent whose key matches an existing
registration **replaces** the previous one (§7).

For an **entity**, `intent_name` is replaced by `entity_name` (same
uniqueness rule: unique within the skill).

When `data` and `context` (e.g. OVOS-MSG-1 keys) both carry an identity
field, `data` is **authoritative** for this specification — `context`
metadata is opaque to the registration semantics.

### 3.2 Responses

Every registration, deregistration, enable, disable, and introspection
topic defined here **MAY** have a `.response` reply (OVOS-MSG-1 §5.3).
Responses are correlated to requests by topic and `session`
(OVOS-MSG-1 §5.4) — no per-message identifier is used.

Success response:

```json
{ "ok": true }
```

Rejection response:

```json
{ "ok": false, "error_code": "malformed_payload", "error": "free-form human-readable detail" }
```

A host **MUST** emit a rejection response with `ok:false` and a normative
`error_code` (§3.3) for every malformed or unservable request it
receives. A host **MAY** emit `ok:true` on success or omit the response
entirely; producers **MUST NOT** require a success response.

### 3.3 `error_code` values

The `error_code` enum is normative; new codes **MUST** be added by a future
version of this specification, not invented per-deployment.

| `error_code` | Meaning |
|--------------|---------|
| `malformed_payload` | The Message's `data` violates the shape required by the topic (§§5–8, §11.2, §13). |
| `no_compatible_engine` | The host has no engine accepting the registration's definition method (§2). |
| `unknown_intent` | A deregister, enable, disable, describe, or introspection request references an `(skill_id, intent_name, lang)` triple that is not currently registered. |
| `unknown_entity` | Analogous to `unknown_intent`, for entities. |
| `unknown_skill` | A `skill_id`-only request references a skill with no current registrations. |

All five are **host-originated** rejections of skill requests. Dispatch
itself has no `.response` and therefore no host-rejection codes; handler
outcome is observed via the handler-lifecycle messages of §12, not via a
correlated reply.

---

## 4. Topics

Topics defined by this specification are lowercase, dot-separated, and
namespaced under `ovos.intent.`, `ovos.entity.`, and `ovos.skill.`:

| Topic | Direction | Purpose | §  |
|-------|-----------|---------|----|
| `ovos.intent.register.keyword` | skill → host | Register a keyword intent (INTENT-3 §4). | §5 |
| `ovos.intent.register.template` | skill → host | Register a template intent (INTENT-3 §5). | §6 |
| `ovos.intent.deregister` | skill → host | Remove one intent. | §8 |
| `ovos.intent.enable` | skill → host | Re-arm a previously disabled intent. | §8.5 |
| `ovos.intent.disable` | skill → host | Suppress an intent without removing its definition. | §8.5 |
| `ovos.entity.register` | skill → host | Register an `.entity` value-set hint (INTENT-3 §5.2). | §7 |
| `ovos.entity.deregister` | skill → host | Remove one entity. | §8 |
| `ovos.skill.deregister` | skill → host | Remove all intents and entities for one `skill_id`. | §8 |
| `ovos.intent.list` | observer → host | List registered intents (introspection). | §13 |
| `ovos.intent.describe` | observer → host | Describe one registered intent. | §13 |
| `ovos.intent.matched` | host → bus | Match-result notification. | §10 |
| `<skill_id>:<intent_name>` | host → skill | Dispatch a matched intent to its handler. The qualified intent name (INTENT-3 §3) is the topic. | §11 |
| `ovos.intent.handler.start` | skill → bus | Handler-lifecycle: handler has begun. | §12 |
| `ovos.intent.handler.complete` | skill → bus | Handler-lifecycle: handler has finished normally. | §12 |
| `ovos.intent.handler.error` | skill → bus | Handler-lifecycle: handler raised. | §12 |

---

## 5. Keyword intent registration

Topic: `ovos.intent.register.keyword`.

A keyword intent is defined by **keyword constraints over vocabularies**
(INTENT-3 §4). This message carries the constraints and the vocabularies in
one payload.

### 5.1 Vocabulary descriptor

A **vocabulary descriptor** is a JSON object identifying one vocabulary
(INTENT-3 §4.1). It uses the OVOS-INTENT-1 §6.1 file-or-inline contract:

Inline form (preferred — see §5.5):

```json
{ "name": "set", "samples": ["set", "change", "adjust"] }
```

File form:

```json
{ "name": "set", "file": "/abs/path/to/set.voc" }
```

`name` is the vocabulary name (INTENT-3 §4.1) — this is the key under which
the vocabulary's captured phrase appears in the match result (§10,
INTENT-3 §4.3). Exactly one of `samples` or `file` **MUST** be present.

`samples` entries are slot-free OVOS-INTENT-1 templates (INTENT-1 §1.1) and
**MUST** contain at least one entry.

### 5.2 Payload shape

```json
{
  "skill_id": "lighting.skill",
  "intent_name": "set_brightness",
  "lang": "en-US",
  "required": [
    { "name": "set", "samples": ["set", "change", "adjust"] },
    { "name": "brightness", "samples": ["brightness", "light level"] }
  ],
  "optional": [],
  "one_of": [
    [
      { "name": "up",   "samples": ["up",   "higher", "brighter"] },
      { "name": "down", "samples": ["down", "lower",  "dimmer"]  }
    ]
  ],
  "excluded": [
    { "name": "question", "samples": ["what is", "how"] }
  ]
}
```

Field reference:

| Field | Type | Required | Meaning (per INTENT-3 §4.2) |
|-------|------|----------|----|
| `required` | array of vocabulary descriptors | yes | Every required vocabulary MUST occur in the utterance. |
| `optional` | array of vocabulary descriptors | yes | Captured if it occurs; absence does not prevent a match. |
| `one_of` | array of arrays of vocabulary descriptors | yes | Each inner array is one **group**; at least one member of each group MUST occur. |
| `excluded` | array of vocabulary descriptors | yes | If any of these occurs, the intent MUST NOT match. |

Empty arrays are permitted. A producer **MUST** include all four keys, even
when empty, so the payload is shape-stable and consumers can rely on
positional semantics.

### 5.3 Constraint validity

The constraint rules of INTENT-3 §4.2 are restated here as bus-layer
malformed-payload rules:

- The combined `required` and `one_of` lists **MUST NOT** both be empty —
  an intent with only `optional` and `excluded` has nothing that must be
  present and is malformed (INTENT-3 §4.2).
- A vocabulary **MUST NOT** appear under more than one role within a single
  registration (INTENT-3 §4.2). Vocabulary identity for this check is by
  `name`.
- Every vocabulary descriptor **MUST** carry exactly one of `samples` or
  `file` (§5.1).
- Each `samples` array **MUST** be non-empty.

A host **MUST** reject any registration violating these rules with
`error_code: "malformed_payload"` (§3.2).

### 5.4 No `.blacklist`

A `.blacklist` is **not** used with keyword intents. The `excluded` role is
the keyword-intent suppression mechanism (INTENT-3 §4.2, §5.4).

### 5.5 File form is single-host only

A `file:` reference is a filesystem path that **MUST** be resolvable by the
host. In a deployment where producer and host share a filesystem (typical
single-machine installs), file form is fully equivalent to inline. In
distributed deployments — separate containers, separate hosts, or any case
where the producer's filesystem is not visible to the host — producers
**MUST** use the inline form (`samples`). A host that cannot resolve a
`file:` path **MUST** reject the registration with
`error_code: "malformed_payload"`.

The same rule applies to every `file:` field in §6 and §7.

---

## 6. Template intent registration

Topic: `ovos.intent.register.template`.

A template intent is defined by **example sentence templates** (INTENT-3 §5,
INTENT-1 §3).

### 6.1 Payload shape

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "samples": [
    "(play|put on) {query}",
    "(play|put on) {query} (on|using) {engine}",
    "i want to listen to {query}"
  ],
  "blacklist": ["trailer", "music video"]
}
```

Or, using the file form (INTENT-1 §6.1; single-host only, §5.5):

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "file": "/abs/path/to/play_music.intent",
  "blacklist_file": "/abs/path/to/play_music.blacklist"
}
```

Field reference:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `samples` | array of strings | exactly one of `samples`/`file` | OVOS-INTENT-1 templates with named slots (INTENT-1 §3, §5). |
| `file` | string | exactly one of `samples`/`file` | Absolute path to a `.intent` resource (INTENT-2 §4.1). Single-host only (§5.5). |
| `blacklist` | array of strings | no | Slot-free phrases (INTENT-2 §4.3) whose occurrence suppresses the match (INTENT-3 §5.4). |
| `blacklist_file` | string | no | Absolute path to a `.blacklist` file. At most one of `blacklist`/`blacklist_file` may be present. Single-host only (§5.5). |

### 6.2 Slot-consistency

Every template in `samples` (or in the file) **MUST** declare the same set of
named slots — the slot-consistency rule of INTENT-1 §5.5. A host **MUST**
reject a registration that violates it with `error_code: "malformed_payload"`.

### 6.3 Malformed payloads

A host **MUST** reject (with `error_code: "malformed_payload"`) a template
registration in which:

- both `samples` and `file` are present, or neither is;
- both `blacklist` and `blacklist_file` are present;
- `samples` is empty;
- a template is not parsable as OVOS-INTENT-1 §3 grammar;
- the slot sets of the templates differ (§6.2).

---

## 7. Entity registration

Topic: `ovos.entity.register`.

An entity is an **optional value-set hint** for a template-intent slot
(INTENT-3 §5.2, INTENT-1 §5.4, INTENT-2 §4.4). Registering an entity is
**never** a precondition for an intent that references the slot name; a slot
with no entity still fills normally.

### 7.1 Payload shape

```json
{
  "skill_id": "music.skill",
  "entity_name": "engine",
  "lang": "en-US",
  "samples": ["spotify", "youtube music", "the radio"]
}
```

Or (single-host only, §5.5):

```json
{
  "skill_id": "music.skill",
  "entity_name": "engine",
  "lang": "en-US",
  "file": "/abs/path/to/engine.entity"
}
```

Field reference:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `entity_name` | string | yes | Unique within the skill. By convention matches the slot name a template intent references. |
| `samples` | array of strings | exactly one of `samples`/`file` | Slot-free value-set entries (INTENT-1 §5.4). |
| `file` | string | exactly one of `samples`/`file` | Absolute path to a `.entity` resource (INTENT-2 §4.4). Single-host only (§5.5). |

### 7.2 Malformed payloads

A host **MUST** reject an entity registration with
`error_code: "malformed_payload"` if both or neither of `samples`/`file` is
present.

---

## 8. Deregistration, enable, disable, and replacement

### 8.1 Replacement is implicit

Registering an intent whose `(skill_id, intent_name, lang)` triple matches
an existing registration **replaces** it — the previous definition is
discarded and wholly superseded by the new one (INTENT-3 §6.1). Replacement
does **not** require a prior deregister message and **MUST** preserve any
enabled/disabled state from §8.5 unless the producer explicitly resets it.

The same rule applies to entities, keyed on `(skill_id, entity_name, lang)`.

### 8.2 `ovos.intent.deregister`

Removes one intent. Payload:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

If `lang` is **omitted**, every language registered for that
`(skill_id, intent_name)` pair is removed.

### 8.3 `ovos.entity.deregister`

Removes one entity. Payload:

```json
{ "skill_id": "music.skill", "entity_name": "engine", "lang": "en-US" }
```

If `lang` is omitted, every language registered for that
`(skill_id, entity_name)` pair is removed.

### 8.4 `ovos.skill.deregister`

Removes **everything** owned by a skill — every intent and every entity
registered under that `skill_id`. Payload:

```json
{ "skill_id": "music.skill" }
```

This is the message a host emits, or that a skill sends to the host, when a
skill is unloaded (INTENT-3 §6.1).

Deregistering an intent, entity, or skill that is not currently registered
**MUST** be rejected with `error_code: "unknown_intent"`,
`"unknown_entity"`, or `"unknown_skill"` respectively — silent no-ops mask
bugs. A producer that wants idempotent removal can ignore those specific
error codes.

### 8.5 `ovos.intent.enable` and `ovos.intent.disable`

A registered intent is, by default, **enabled** — eligible for matching. A
skill **MAY** temporarily **disable** an intent without removing it; the
host retains the definition and handler binding but excludes the intent
from match candidacy until it is re-enabled. Both topics share the same
payload as `ovos.intent.deregister` (§8.2), and `lang` semantics:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

If `lang` is omitted, every language for that `(skill_id, intent_name)` is
affected.

Enabling an already-enabled intent, or disabling an already-disabled
intent, is **not** an error — the host **MAY** respond `{ "ok": true }`.
Re-registration (§8.1) preserves enabled/disabled state unless the producer
deregisters first.

---

## 9. The handler reference is not on the bus

Per INTENT-3 §6.1, the **handler reference** — the actual code object that
runs when the intent matches — is **not** part of any registration message
and **never** crosses the bus. It is held locally by the skill process.

The bus carries only:

- the intent **definition** (§§5–7);
- the match **notification** (§10);
- a dispatch **message** addressed to the owning skill (§11), whose
  payload tells the skill which handler to run;
- the **handler-lifecycle messages** the skill emits to report what its
  handler did (§12).

Together these allow a skill in a different process from the host to
host its handlers across the bus, without ever serializing the handler
itself. This is the key contract that lets local and remote skills look
the same from the host's perspective.

---

## 10. The match-result message

Topic: `ovos.intent.matched`.

When the host has selected a single matching intent (§2), it emits this
message to the bus as a **notification** of which intent matched and with
which captures. It is the bus-level representation of OVOS-INTENT-3 §7.

### 10.1 Routing

The Message's `context` (OVOS-MSG-1):

- `session` is propagated from the originating utterance, when one is
  established;
- `source` identifies the host (the producer of the notification);
- `destination` is absent — `ovos.intent.matched` is a broadcast
  notification.

### 10.2 Payload shape

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "utterance": "put on the beatles using spotify",
  "captures": { "query": "the beatles", "engine": "spotify" },
  "engine": "padatious",
  "confidence": 0.92
}
```

Field reference:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The owning skill (INTENT-3 §3). |
| `intent_name` | string | yes | Together with `skill_id`, the qualified intent name (INTENT-3 §3). |
| `lang` | string | yes | The language the intent was matched in. |
| `utterance` | string | yes | The ASR-normalized utterance that matched (INTENT-1 §2). |
| `captures` | object (string→string) | yes | The capture map (INTENT-3 §7). Keys are slot names (template intents, INTENT-3 §5.2) or vocabulary names (keyword intents, INTENT-3 §4.3). MAY be empty. |
| `engine` | string | no | Non-normative engine identifier (INTENT-3 §7). |
| `confidence` | number | no | Non-normative engine confidence (INTENT-3 §7). |

Per INTENT-3 §7, captured values are **opaque sequences of words**: strings,
with no value-typing applied by this layer. Future versions **MAY** extend
captures to richer forms (multi-valued, typed, with span offsets); under
version 1 the `string → string` shape is the conformance contract.

Correlation back to the originating utterance is via `session`
(OVOS-MSG-1 §4) on the `context`.

### 10.3 Notification, not dispatch

`ovos.intent.matched` exists so observers — loggers, transcript viewers,
fallback chains, analytics, training-data collectors — can see what the
intent layer selected. A consumer **MUST NOT** treat receipt of
`ovos.intent.matched` as permission or instruction to run the handler;
handler invocation is the host's job, via the qualified-name dispatch
topic `<skill_id>:<intent_name>` (§11).

### 10.4 At most one per utterance

Per INTENT-3 §6.2 and §2 above, an utterance produces at most one matched
intent. The host therefore emits at most one `ovos.intent.matched` per
utterance. When no intent matches, no `ovos.intent.matched` is emitted.

---

## 11. Dispatch

Topic: **`<skill_id>:<intent_name>`** — the qualified intent name of
OVOS-INTENT-3 §3 used directly as the bus topic.

Immediately after emitting `ovos.intent.matched`, the host emits a Message
on the qualified-name topic of the matched intent. This is the message
that causes the skill to run its bound handler.

A per-intent topic is used rather than a single addressed topic for three
reasons:

- the qualified name `<skill_id>:<intent_name>` is **already unique
  per-handler** (INTENT-3 §3), so the topic itself selects exactly one
  handler — no separate addressing field is needed;
- skills subscribe only to **their own** handler topics, so an overheard
  dispatch is impossible by construction — the bus delivers the Message
  only to the owning skill;
- it matches existing OVOS practice; legacy code already dispatches this
  way, so a conformant v1 host requires no rename to existing handler
  subscriptions.

### 11.1 Routing

The Message's `context` (OVOS-MSG-1) **MUST** propagate `session`
from the originating `ovos.intent.matched` unchanged, so the handler runs
inside the same conversational session as the utterance that triggered
it (and any subsystem keyed on `session_id == "default"`, e.g.
`ovos-audio`, continues to fire correctly — OVOS-MSG-1 §4.1).

`destination` **MAY** be set to the `skill_id` for clarity, but the topic
name is authoritative — only the owning skill subscribes to it (§11.3).

`source` identifies the host (the producer of the dispatch). The skill
does **not** address a directed reply back to the host: dispatch outcome
is announced through the broadcast lifecycle trio of §12, which is
produced via `forward` (OVOS-MSG-1 §5.1) and therefore preserves the
dispatch's `source` rather than swapping it. Skill identity in the trio
is conveyed by the payload's `skill_id` field, not by `context.source`.

### 11.2 Payload shape

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "lang": "en-US",
  "utterance": "put on the beatles using spotify",
  "captures": { "query": "the beatles", "engine": "spotify" }
}
```

Field reference:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The owning skill (INTENT-3 §3). MUST equal the topic's `skill_id` prefix. |
| `intent_name` | string | yes | The matched intent name. MUST equal the topic's `intent_name` suffix. |
| `lang` | string | yes | The language the intent was matched in. |
| `utterance` | string | yes | The ASR-normalized utterance (INTENT-1 §2). |
| `captures` | object (string→string) | yes | The capture map (INTENT-3 §7). MAY be empty. |

The dispatch payload is intentionally a **subset** of §10.2: the
non-normative engine metadata (`engine`, `confidence`) is not propagated
to the skill, because skill code is engine-agnostic by design
(INTENT-3 §6.2). The redundant `skill_id` and `intent_name` in the payload
let a skill confirm the topic the handler is actually being asked to
serve, guarding against a misrouted subscription.

### 11.3 Subscription discipline

Per INTENT-3 §6.1, an intent has exactly one handler. The owning skill
**MUST** subscribe to `<skill_id>:<intent_name>` for every intent it has
registered, and **MUST NOT** subscribe to any qualified-name topic
belonging to a different `skill_id`. The host **MUST** emit on the
qualified-name topic exactly once per match.

Because the topic itself encodes the `(skill_id, intent_name)` pair, a
correctly-subscribed skill cannot receive a dispatch belonging to
another skill; the bus delivers it only to the owning subscriber. A skill
that nevertheless receives a Message whose payload `skill_id` differs
from its own (a configuration bug) **MUST NOT** run the handler and
**SHOULD** log the discrepancy.

### 11.4 In-process equivalence

When the owning skill runs in the same process as the host, the host
**MAY** invoke the handler directly without serializing the
`<skill_id>:<intent_name>` Message over a transport — provided every
external observer sees the same `<skill_id>:<intent_name>` dispatch and
the same `ovos.intent.handler.{start,complete,error}` lifecycle traffic
(§12) it would have seen for a remote skill. This uniformity is what
makes a deployment portable across in-process and out-of-process skill
arrangements.

---

## 12. Handler-lifecycle messages

Dispatch has **no `.response` reply**. Instead, the skill announces the
handler's progress through three lifecycle topics — the trio
**`ovos.intent.handler.start`**, **`ovos.intent.handler.complete`**,
and **`ovos.intent.handler.error`**. These messages are broadcast
notifications (no `destination`); any observer (the host, loggers,
transcript viewers, the fallback chain, analytics) can subscribe.

The trio mirrors the start/complete/error shape that `ovos-workshop`
already emits for every skill handler today, under legacy topic names
(`mycroft.skill.handler.{start,complete,error}`) that this specification
renames into the `ovos.intent.` namespace for uniformity with the rest
of the topics defined here. The legacy names are tabled in Appendix A.

### 12.1 Order and obligations

For each accepted dispatch (§11), the skill **MUST** emit exactly one of
the following sequences:

- on normal completion: `ovos.intent.handler.start` followed by
  `ovos.intent.handler.complete`;
- on handler exception: `ovos.intent.handler.start` followed by
  `ovos.intent.handler.error`.

A skill **MUST NOT** emit both `complete` and `error` for the same
dispatch, and **MUST NOT** emit `complete` or `error` without a
preceding `start`. A skill **MUST NOT** run the bound handler more than
once per dispatch.

A skill that has decided not to accept a dispatch (for example because
the payload `skill_id` does not match its own — §11.3) emits **none** of
the trio; silence indicates "this dispatch was not mine."

### 12.2 Routing

All three messages are produced via OVOS-MSG-1 §5.1 `forward` from
the originating dispatch Message, which means the `context` is preserved
wholesale. In particular:

- `session` (OVOS-MSG-1 §4) is preserved unchanged — every observer
  keyed on `session_id == "default"` (e.g. `ovos-audio` deciding whether
  to play TTS locally) continues to fire correctly through the
  lifecycle;
- `source` is **preserved** from the dispatch (i.e. it still names the
  host, not the skill) — `forward` does not rewrite `source`
  (OVOS-MSG-1 §5.1). The skill that emitted the lifecycle message
  identifies itself through the payload's `skill_id` field (§12.3), not
  through `context.source`;
- `destination` is preserved; in the typical case where the dispatch
  had no `destination` (or had only the skill as destination), the
  lifecycle messages are broadcast, which matches their notification
  semantics.

### 12.3 Payloads

All three messages carry the identity of the handler being reported on.
The payload is intentionally compact — the dispatch Message already
broadcast the full context (utterance, captures) via §11.2, and
observers correlate the lifecycle messages with the dispatch by
`session` and `intent_name`.

`ovos.intent.handler.start`:

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music"
}
```

`ovos.intent.handler.complete`:

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music"
}
```

`ovos.intent.handler.error`:

```json
{
  "skill_id": "music.skill",
  "intent_name": "play_music",
  "exception": "RuntimeError: Spotify is not configured"
}
```

Field reference (all three messages):

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `skill_id` | string | yes | The skill whose handler is running (INTENT-3 §3). |
| `intent_name` | string | yes | The intent the handler was dispatched for. |
| `exception` | string | `error` only | Human-readable description of the failure (typically `repr(e)`). |

Implementations **MAY** include additional fields (timing, handler
function name, etc.) but consumers **MUST NOT** require them.

### 12.4 Host timeout

A host **MAY** wait for `ovos.intent.handler.complete` or
`ovos.intent.handler.error` matching the dispatched
`(skill_id, intent_name, session)` for a deployment-defined time bound.
If neither arrives within the bound, the host **MAY** treat the dispatch
as failed for its own accounting purposes (logging, fallback routing,
user notification), but **MUST NOT** re-emit the dispatch Message for
the same match. Re-dispatch is not defined by this specification.

A host **MUST NOT** synthesize a `ovos.intent.handler.error` of its
own to stand in for a missing one — error messages come from the skill
that owns the handler.

### 12.5 Long-running handlers

A handler that needs longer than a typical host timeout to complete
**SHOULD** still eventually emit `ovos.intent.handler.complete` or
`ovos.intent.handler.error` to close the lifecycle. Interim progress
messages (acknowledgements, streaming updates, partial speech) are a
deployment concern outside the scope of this specification, and may be
specified separately.

---

## 13. Introspection

Two read-only topics let an observer (debugger, UI, conformance test, an
external orchestrator) query the host's current registration state.

### 13.1 `ovos.intent.list`

Lists registered intents. Request payload:

```json
{ "skill_id": "music.skill", "lang": "en-US" }
```

Both fields are **optional filters**: omitting `skill_id` returns every
skill's intents; omitting `lang` returns every language.

Response (`ovos.intent.list.response`):

```json
{
  "ok": true,
  "intents": [
    {
      "skill_id": "music.skill",
      "intent_name": "play_music",
      "lang": "en-US",
      "method": "template",
      "enabled": true
    }
  ]
}
```

Each entry carries `skill_id`, `intent_name`, `lang`, a `method` of
`"keyword"` or `"template"` (INTENT-3 §2), and an `enabled` boolean (§8.5).

### 13.2 `ovos.intent.describe`

Returns the full definition of one intent. Request payload:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

Response (`ovos.intent.describe.response`):

- On success, `{ "ok": true, "method": "keyword"|"template", "definition": {...} }`
  where `definition` is the §5 or §6 payload (file forms expanded to inline
  where possible).
- On unknown intent, `{ "ok": false, "error_code": "unknown_intent", "error": "no such (skill_id, intent_name, lang) registered" }`.

A host **MAY** restrict who is permitted to call introspection topics;
authorization is not defined by this specification.

---

## 14. Conformance

### A **skill** (producer of registration and handler-lifecycle messages) **MUST**:

- emit each registration through the topic that matches its definition
  method (§5 for keyword, §6 for template), never both for a single intent
  (INTENT-3 §2);
- include the identity fields of §3.1 in every registration's `data`;
- conform every registration's payload to §5 (keyword), §6 (template), or
  §7 (entity), respectively, including §5.5's single-host rule for `file:`
  references;
- emit `ovos.intent.deregister` / `ovos.entity.deregister` /
  `ovos.skill.deregister` to retract its registrations, paired with the
  local release of the handler (§9, INTENT-3 §6.1);
- conform its underlying templates, vocabularies, and entities to
  OVOS-INTENT-1 and OVOS-INTENT-2;
- subscribe only to `<skill_id>:<intent_name>` topics for which the
  `skill_id` is its own (§11.3); silently ignore any dispatch whose
  payload `skill_id` does not match;
- run the bound handler at most once per dispatch (§12.1);
- emit the handler-lifecycle trio per §12 for every dispatch it
  accepted — `ovos.intent.handler.start` followed by exactly one of
  `ovos.intent.handler.complete` or `ovos.intent.handler.error`
  (§12.1, §12.3);
- use an `error_code` from §3.3 on every registration `ok:false`
  response.

### A **host** (consumer of skill messages, producer of match and dispatch messages) **MUST**:

- be the sole bus consumer of every skill-originated topic defined here
  (§2);
- delegate matching to its intent engines via a host-internal interface,
  selecting at most one match per utterance (§2, INTENT-3 §6.2);
- reject malformed or unservable requests with an `error_code` from §3.3
  carried in the `.response` (§3.2);
- treat a re-registration with the same key as replacement (§8.1);
- honour `ovos.intent.enable` / `ovos.intent.disable` by excluding disabled
  intents from match candidacy (§8.5);
- emit `ovos.intent.matched` per §10 when an intent matches, and
  immediately follow it with a dispatch on `<skill_id>:<intent_name>`
  (§11);
- observe the handler-lifecycle trio (§12) to learn the outcome of a
  dispatch; correlate by `(skill_id, intent_name, session)`;
- emit at most one `ovos.intent.matched` and one `<skill_id>:<intent_name>`
  dispatch per utterance (§10.4, §11.3).

### An **intent engine** (host-internal, off the bus) **MUST**:

- accept at least one of the two definition methods and reject
  registrations for methods it does not accept (INTENT-3 §6.2) — the
  rejection surfaces on the bus as the host's `no_compatible_engine` error
  (§2);
- expand vocabularies and templates with an OVOS-INTENT-1-conformant
  expander;
- honour the keyword constraint semantics of §5.3 (INTENT-3 §4.2) and the
  template `blacklist` suppression of §6.1 (INTENT-3 §5.4);
- produce the result that the host uses to populate §10.2.

Matching, generalization, scoring, and the ranking of competing intents
are deliberately **unconstrained** — an engine MAY use any strategy
(INTENT-3 §8).

---

## Appendix A — Legacy topic mapping (non-normative)

This appendix is **non-normative**. It documents the legacy Mycroft- and
OVOS-derived topics that predate this specification, mapping each to its v1
replacement. It exists only to help implementers migrate. New code
**SHOULD** emit the v1 topics directly.

| Legacy topic | v1 replacement | Notes |
|--------------|---------------|-------|
| `register_vocab` | folded into `ovos.intent.register.keyword` (§5) | Vocabularies in v1 are inline `samples` or `file`-by-path inside the registration; not separate messages. |
| `register_intent` (Adapt parser) | `ovos.intent.register.keyword` (§5) | The Adapt `IntentBuilder` `.__dict__` payload is replaced by the structured shape of §5.2. |
| `padatious:register_intent` | `ovos.intent.register.template` (§6) | The `samples` / `file` / `blacklist*` payload of §6.1 supersedes the Padatious-specific shape. |
| `padatious:register_entity` | `ovos.entity.register` (§7) | Entities are not Padatious-specific (INTENT-3 §5.2). |
| `detach_intent` | `ovos.intent.deregister` (§8.2) | Identity now expressed as the structured `(skill_id, intent_name, lang)` triple, not the `skill_id:intent_name` munged string. |
| `detach_skill` | `ovos.skill.deregister` (§8.4) | |
| `enable_intent` / `disable_intent` | `ovos.intent.enable` / `ovos.intent.disable` (§8.5) | First-class topics under v1; previously informal. |
| `<skill_id>:<intent_name>` (handler dispatch) | **unchanged** (§11) | The per-intent qualified-name dispatch topic of legacy OVOS is the prescribed v1 topic. Skills already subscribed to it require no change. |
| `mycroft.skill.handler.start` | `ovos.intent.handler.start` (§12) | Renamed for uniformity with the `ovos.intent.*` namespace. Payload shape is unchanged in spirit (skill + handler identity), tightened to a normative `(skill_id, intent_name)` pair. |
| `mycroft.skill.handler.complete` | `ovos.intent.handler.complete` (§12) | Same rename. |
| `mycroft.skill.handler.error` | `ovos.intent.handler.error` (§12) | Same rename; `exception` field is normative. |
| `ovos.utterance.handled` | (subsumed by `ovos.intent.handler.complete`) | Legacy end-of-intent marker emitted alongside `complete`; the trio of §12 is sufficient on its own. |
| `add_context` / `remove_context` | (out of scope) | Adapt conversational context is not part of the intent registration contract. A separate specification may define it. |

Per the repository's *Authority* section, current OVOS code that still emits
the legacy topics is a known implementation bug being worked through, not a
defect in this specification.

---

## See also

- *Bus Message Specification* (OVOS-MSG-1) — the envelope every message
  here travels in, the `destination` and `session` keys used throughout,
  and the `forward` / `reply` / `response` derivations.
- *Intent Definition Specification* (OVOS-INTENT-3) — the intent concept,
  identity, definition methods, and match result that this specification
  carries on the bus.
- *Locale Resource Formats Specification* (OVOS-INTENT-2) — the resource
  files referenced by the `file` form of each registration.
- *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the grammar
  of the inline `samples` form, and the file-or-inline training-data
  contract.
