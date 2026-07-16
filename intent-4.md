# Intent and Entity Registration Bus Contract

**Spec ID:** OVOS-INTENT-4 · **Version:** 2 · **Status:** Draft

This document defines the **bus messages** a skill uses to declare its
intents and entities. It is the wire format for intent registration —
nothing else. Registrations are broadcast on the bus; pipeline plugins
(OVOS-PIPELINE-1) consume what they want; the orchestrator passively
indexes everything for introspection. The bus-level utterance
lifecycle (matching, dispatch, handler lifecycle, terminal events) is
owned by OVOS-PIPELINE-1.

It is the bus-level companion to OVOS-INTENT-3: where that
specification defines *what* an intent is, this one defines *how* a
skill puts that intent on the bus so a plugin can match against it.

It builds on three companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope every
  message defined here travels in, the `destination` routing key, the
  `session` carrier, and the `forward` / `reply` / `response` derivations;
- the *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  concept, identity model, the two definition methods, and the match
  result that this spec carries on the bus;
- the *Locale Resource Formats Specification* (OVOS-INTENT-2) and the
  *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the
  authoring file formats and template grammar a skill loader expands
  before emitting a registration payload (file paths never cross the
  bus; see §5.1).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT** and **MAY**
are used as in RFC 2119.

---

## 1. Scope

This specification defines a small fixed set of registration topics
and the orchestrator-provided introspection interface:

- the two **intent registration** messages — one per definition method
  (OVOS-INTENT-3 §2);
- the **entity registration** message — the `.entity` value-set hint
  (OVOS-INTENT-3 §5.2);
- the **deregistration** messages, for one intent, one entity, or a
  whole skill;
- the **enable** and **disable** messages — temporary suppression of a
  registered intent without losing its definition;
- the **introspection** messages — list and describe registrations,
  served by the orchestrator's passive registration index (§10);
- the **session-scoped registration** model (§11) — how every
  registration is automatically keyed to the registering session,
  enabling per-session skill sets and distributed satellite
  deployments.

It does **not** define:

- the intent concept itself (OVOS-INTENT-3) or the handler reference,
  which never crosses the bus (§9);
- how plugins *implement* registration storage, matching, or
  consumption — black box (OVOS-PIPELINE-1);
- the **utterance lifecycle**, **dispatch**, **handler-lifecycle
  trio**, **match-result notification**, or
  `ovos.intent.unmatched` — all owned by OVOS-PIPELINE-1;
- **session lifecycle** — `session` is carried opaquely per
  OVOS-MSG-1;
- **language fallback** — when no registration matches the utterance
  language exactly. Deferred to a future spec.

---

## 2. Architectural model — registrations are broadcast

Registration messages defined here are **broadcast** on the bus.
There is no central party that owns, validates, or routes them;
whether any loaded pipeline plugin (OVOS-PIPELINE-1) consumes a
given registration is a plugin concern, out of scope here. A
registration no plugin consumes is silently dropped — the skill's
intent will not match, and the deployment is responsible for
diagnosing why (typically: wrong plugin loaded for the
registration method).

The **orchestrator** (OVOS-INTENT-3 §6.1) maintains the manifest
(§10): a passive index built from observed registrations,
observability-only. It does not gate matching, influence
consumption, or block re-registration. Plugins are observably
pluggable — adding or removing one is a deployment concern; bus
traffic and the manifest are unaffected.

Registrations are **fire-and-forget**: there is no `.response`
reply, no acknowledgement, no error event. A producer that needs
to verify a registration landed queries the manifest (§10);
manifest presence is the only signal this specification defines.

---

## 3. Identity

### 3.1 Skills self-identify on every emission

A skill **MUST** set `Message.context["skill_id"]` to its own
`skill_id` (OVOS-INTENT-3 §3) on every Message it originates or
mutates before placing it on the bus. This covers fresh emissions
(registration messages of §§5–8, ad-hoc skill-defined topics,
etc.) and any Message whose `context`, `data`, or session the
skill modifies before emission.

For a skill handler running under dispatch, conformance is
structural: the orchestrator stamps `context["skill_id"]` on the
dispatch Message (OVOS-PIPELINE-1 §7.1), and all Messages the
handler derives from it via the OVOS-MSG-1 §5 derivation
semantics inherit that value automatically. No extra stamp step
is needed on the dispatch path.

`Message.context["skill_id"]` is the **authoritative attribution
key** for skill-originated bus traffic — observers **MUST NOT**
infer the originating skill from topic names or `data` fields. A
Message arriving without `context["skill_id"]` is either not
skill-originated or is from a non-conformant skill.

#### Enforcement

On the dispatch path enforcement is structural — the orchestrator
stamps `context["skill_id"]` per OVOS-PIPELINE-1 §7.1 and MSG-1
derivation propagates it to handler-derived Messages. For
emissions outside the dispatch path, the component that loads
skills **SHOULD** intercept the emit pathway so non-conformant
handler code cannot escape. A Message whose `context["skill_id"]`
disagrees with the `<skill_id>` of the dispatch it derives from
is malformed; the orchestrator **MUST** log the drift at WARN.

### 3.2 Identity carried by every registration message

Every registration message carries the identity of what is being registered,
inside the Message's `data` (OVOS-MSG-1 §2.2). The identity fields restate
OVOS-INTENT-3 §3 at the bus layer.

For an **intent**:

| Field | Type | Required | Source |
|-------|------|----------|--------|
| `skill_id` | string | yes | INTENT-3 §3 — assistant-unique. |
| `intent_name` | string | yes | INTENT-3 §3 — unique within the skill. |
| `lang` | string | yes | BCP-47 (INTENT-2 §2), case-insensitive (SESSION-1 §3.2). The language of the resource being registered — distinct from `session.lang`. |

The triple `(skill_id, intent_name, lang)` identifies an **intent**
(INTENT-3 §6.1). For manifest indexing and replacement (§8.1), the
**registration key** is the quadruple
`(skill_id, intent_name, lang, method)` — `method` being `keyword`
(§5) or `template` (§6). Registering a quadruple that matches an
existing entry **replaces** that entry only; the other-method
registration for the same triple is untouched. Replacement is also
per-language: other languages of the same `(skill_id, intent_name)`
are unaffected.

A single intent **MAY** be registered under both methods — they are
two training-data representations of the same handler. Different
pipeline plugins consume different methods; a match from either
dispatches to the same `<skill_id>:<intent_name>` topic. The wire
contract makes no claim about which representation should "win" when
both produce a match — that is a pipeline policy concern
(OVOS-PIPELINE-1). Producers **MAY** ship divergent suppression
vocabularies between the two methods (different `excluded` for
keyword vs different `blacklist` for template); each plugin honours
only its own method's suppression.

For an **entity**, `intent_name` is replaced by `entity_name` (same
uniqueness rule: unique within the skill). Entity registrations have
no `method` axis.

Other specifications **MAY** reserve specific `intent_name`
values; the authoritative registry is OVOS-PIPELINE-1 §7.3. A
registration naming a reserved `intent_name` is **malformed** —
the orchestrator and every consuming plugin treat it under the
malformed-payload rules of §5.3 / §6.3 (log at WARN, do not index).

---

## 4. Topics

Topics defined by this specification are lowercase, dot-separated,
and namespaced under `ovos.intent.`, `ovos.entity.`, and
`ovos.skill.`. All registration topics are **broadcast** — any
component (typically pipeline plugins) may subscribe. The orchestrator
also subscribes to all of them passively, to maintain the
introspection index of §10.

| Topic | Direction | Purpose | §  |
|-------|-----------|---------|----|
| `ovos.intent.register.keyword` | skill → bus (broadcast) | Register a keyword intent (INTENT-3 §4). | §5 |
| `ovos.intent.register.template` | skill → bus (broadcast) | Register a template intent (INTENT-3 §5). | §6 |
| `ovos.intent.deregister` | skill → bus (broadcast) | Remove one intent. | §8 |
| `ovos.intent.enable` | skill → bus (broadcast) | Re-arm a previously disabled intent. | §8.5 |
| `ovos.intent.disable` | skill → bus (broadcast) | Suppress an intent without removing its definition. | §8.5 |
| `ovos.entity.register` | skill → bus (broadcast) | Register an `.entity` value-set hint (INTENT-3 §5.2). | §7 |
| `ovos.entity.deregister` | skill → bus (broadcast) | Remove one entity. | §8 |
| `ovos.skill.deregister` | skill → bus (broadcast) | Remove all intents and entities for one `skill_id`. | §8 |
| `ovos.intent.list` | observer → orchestrator | Query registered intents (introspection; served by the orchestrator). | §10 |
| `ovos.intent.describe` | observer → orchestrator | Query one registered intent (introspection; served by the orchestrator). | §10 |

Match notification, dispatch, and handler-lifecycle topics live in
OVOS-PIPELINE-1 §§7–9, not here.

---

## 5. Keyword intent registration

Topic: `ovos.intent.register.keyword`.

A keyword intent is defined by **keyword constraints over vocabularies**
(INTENT-3 §4). This message carries the constraints and the vocabularies in
one payload.

### 5.1 Vocabulary descriptor

A **vocabulary descriptor** is a JSON object identifying one vocabulary
(INTENT-3 §4.1):

```json
{ "name": "set", "samples": ["set", "change", "adjust"] }
```

`name` is the vocabulary name (INTENT-3 §4.1) — this is the key under
which the vocabulary's captured phrase appears in the match result
(OVOS-PIPELINE-1 `Match.slots`; INTENT-3 §4.3). `samples` entries are
slot-free OVOS-INTENT-1 templates (INTENT-1 §1.1) and **MUST** contain
at least one entry.

Locale resource files (`.voc`, `.intent`, `.entity`, `.blacklist`;
OVOS-INTENT-2) are a producer-side authoring convenience: a skill
loader reads them and inlines their expanded content into the
registration payload. File paths never appear on the wire.

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

- The `intent_name` **MUST NOT** be one reserved by another spec
  (§3.2; the authoritative registry is OVOS-PIPELINE-1 §7.3).
- The combined `required` and `one_of` lists **MUST NOT** both be empty —
  an intent with only `optional` and `excluded` has nothing that must be
  present and is malformed (INTENT-3 §4.2).
- A vocabulary **MUST NOT** appear under more than one role within a single
  registration (INTENT-3 §4.2). Vocabulary identity for this check is by
  `name`.
- Every vocabulary descriptor **MUST** carry a non-empty `samples`
  array (§5.1).
- A vocabulary descriptor's `samples` **MUST** include at least one
  template that expands to a non-empty sample (OVOS-INTENT-1 §3.6).
  A descriptor that yields zero non-empty samples is malformed.

A producer **MUST** include all four top-level keys (`required`,
`optional`, `one_of`, `excluded`); a payload missing any of them is
malformed.

A consuming plugin **MUST NOT** index a registration that violates
these rules. The rejecting plugin **MUST** log the rejection at
WARN, including `skill_id`, `intent_name`, `lang`, the rejecting
topic, and a one-line reason — this is the only debugging signal a
producer receives, since the bus is fire-and-forget (§2). The
topic is part of the actionable signal because the same
`(skill_id, intent_name, lang)` may be valid as keyword and
malformed as template (or vice versa, §3.2). Structured logging is
**RECOMMENDED**.

Within a vocabulary descriptor, an individual sample that is not
parsable as OVOS-INTENT-1 §3 grammar, or that expands to zero
non-empty samples, does **not** malform the registration. A consuming
plugin **MUST NOT** reject the registration on its account: it
**MUST** skip the offending sample, **MUST** log each skipped sample
at WARN with the fields above plus the sample itself, and **MUST**
index the remaining valid samples. Only a descriptor in which no
sample expands to a non-empty sample is malformed (the zero-yield rule
above), and only then is the registration rejected.

### 5.4 No `.blacklist`

A `.blacklist` is **not** used with keyword intents. The `excluded` role is
the keyword-intent suppression mechanism (INTENT-3 §4.2, §5.5).

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
  "blacklist": ["trailer", "music video"],
  "required_slots": ["query"]
}
```

Field reference:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `samples` | array of strings | yes | OVOS-INTENT-1 templates with named slots (INTENT-1 §3, §5). |
| `blacklist` | array of strings | no | Slot-free phrases (INTENT-2 §4.3) whose occurrence suppresses the match (INTENT-3 §5.5). |
| `required_slots` | array of strings | no | Slot names the engine MUST extract for a match to be valid (INTENT-3 §5.3). |

### 6.2 Slot sets

Templates in `samples` MAY declare **different sets of named slots**;
the engine extracts only the slots declared by the template that best
matches (INTENT-1 §5.5, INTENT-3 §5.1). A consuming plugin MUST accept
registrations with differing slot sets across templates.

### 6.3 Malformed payloads

A consuming plugin **MUST NOT** index a template registration in which:

- the `intent_name` is reserved by another spec (§3.2);
- `samples` is missing or empty;
- no template in `samples` is both parsable as OVOS-INTENT-1 §3
  grammar and expands to at least one non-empty sample
  (OVOS-INTENT-1 §3.6);
- `required_slots` names a slot that is not declared by any valid
  template in `samples` (INTENT-3 §5.3).

An individual template that is not parsable as OVOS-INTENT-1 §3
grammar, or that expands to zero non-empty samples, does **not**
malform the registration by itself. A consuming plugin **MUST NOT**
reject the registration on its account: it **MUST** skip that
template, **MUST** log each skipped template at WARN with the §5.3
fields plus the template itself, and **MUST** index the remaining
valid templates. The registration is rejected only when no valid
template remains (third bullet above).

The §5.3 WARN-log rule applies: the rejecting plugin **MUST** log
the rejection with `skill_id`, `intent_name`, `lang`, and a
one-line reason.

---

## 7. Entity registration

Topic: `ovos.entity.register`.

An entity is an **optional value-set hint** for a template-intent slot
(INTENT-3 §5.2, INTENT-1 §5.4, INTENT-2 §4.3). Registering an entity is
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

Field reference:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `entity_name` | string | yes | Unique within the skill. By convention matches the slot name a template intent references. |
| `samples` | array of strings | yes | Slot-free value-set entries (INTENT-1 §5.4). |

### 7.2 Malformed payloads

A consuming plugin **MUST NOT** index an entity registration whose
`samples` is missing or empty, or in which no entry yields a non-empty
value. The §5.3 WARN-log rule applies: the rejecting plugin **MUST**
log the rejection with `skill_id`, `entity_name`, `lang`, and a
one-line reason.

An individual entry that is not parsable as OVOS-INTENT-1 §3 grammar,
or that yields no non-empty value, does **not** malform the
registration by itself. A consuming plugin **MUST NOT** reject the
registration on its account: it **MUST** skip the offending entry,
**MUST** log each skipped entry at WARN with the §5.3 fields plus the
entry itself, and **MUST** index the remaining valid entries.

---

## 8. Deregistration, enable, disable, and replacement

### 8.1 Replacement is implicit

Registering an intent whose `(session_id, skill_id, intent_name, lang, method)`
quintuple matches an existing registration **replaces** it
(INTENT-3 §6.1) — no prior deregister needed. Replacement preserves
enabled/disabled state (§8.5); a producer that wants to reset that
state deregisters first. The same rule applies to entities, keyed on
the quadruple `(session_id, skill_id, entity_name, lang)` — entities
have no `method` axis. The `session_id` is read from
`context.session.session_id` (§11.1) — never from `Message.data`.

### 8.2 `ovos.intent.deregister`

Removes one intent. Payload:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

If `lang` is **omitted**, every language registered for that
`(skill_id, intent_name)` pair is removed. Deregistration targets
the `(skill_id, intent_name, lang)` triple and removes **all
methods** under it — both the keyword and template registrations
of the same intent (§3.2), if both exist. There is no per-method
deregistration; a skill that wants to remove only one method
re-registers the other.

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

An optional `session_id` field narrows the removal to registrations
scoped to that session (§11):

```json
{ "skill_id": "music.skill", "session_id": "satellite-abc" }
```

This is the message an orchestrator emits, or that a skill sends to
the orchestrator, when a skill is unloaded (INTENT-3 §6.1). A bridge
SHOULD emit `ovos.skill.deregister` with the satellite's `session_id`
for every skill the satellite registered when the satellite
disconnects (OVOS-BRIDGE-1 §3).

Deregistering an intent, entity, or skill that is not currently
registered is a **no-op**: registrations are fire-and-forget, every
plugin processes the message independently, and any plugin without a
matching record simply has nothing to remove. This makes the
shutdown sequence — where every plugin the skill ever talked to
receives every deregistration — naturally idempotent.

Races between a deregistration and an in-flight match (a match
emitted before the deregister was processed, dispatched after) are
the responsibility of the utterance lifecycle owner — see
OVOS-PIPELINE-1.

### 8.5 `ovos.intent.enable` and `ovos.intent.disable`

A registered intent is, by default, **enabled** — eligible for matching. A
skill **MAY** temporarily **disable** an intent without removing it; the
orchestrator retains the definition in the manifest but marks it disabled,
and plugins exclude it from match candidacy until it is re-enabled. The
bus-level surface (rather than skill-side gating) lets *external* tooling —
admin UIs, A/B experiments, conflict resolution — suppress an intent
without modifying skill code. Both topics share the same payload as
`ovos.intent.deregister` (§8.2), and `lang` semantics:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

If `lang` is omitted, every language for that `(skill_id, intent_name)`
is affected. Like deregistration, enable/disable target the triple and
apply to **all methods** of the intent — there is no per-method
enable/disable. A producer that wants to retain only one method
deregisters the triple (§8.2, removes both methods) and re-registers
just the desired one.

Enabling an already-enabled intent, or disabling an already-disabled
intent, is a no-op. Re-registration (§8.1) preserves enabled/disabled
state unless the producer deregisters first.

---

## 9. The handler reference is not on the bus

Per INTENT-3 §6.1, the **handler reference** — the code object that
runs when the intent matches — never crosses the bus; it is held
locally by the skill process. This specification puts only the
intent *definition* (§§5–7) on the wire; the dispatch Message that
invokes the handler is defined in OVOS-PIPELINE-1. Together they
let a skill in a different process from the orchestrator host its
handlers across the bus without serializing them — the contract
that makes local and remote skills indistinguishable from outside.

---

## 10. Introspection — the orchestrator-owned manifest

Registration broadcasts of §5–§8 are load-time announcements; a
consumer that subscribed after the skill loaded has missed them
(the bus is async with no catch-up channel). The
**orchestrator-owned manifest** is this specification's answer —
the orchestrator indexes every registration it observes and serves
queries against it. Skills have no introspection obligation; they
emit and move on.

**Pull-query is the source of truth.** A consumer that needs
accurate state **MUST** issue `ovos.intent.list` /
`ovos.intent.describe` and **MUST NOT** rely on having heard the
original broadcast. For *compiled-plugin state* — which intents a
particular matcher actually has loaded — query OVOS-PIPELINE-1 §10
instead; the surfaces are distinct (declared vs compiled).

Under a split orchestrator (OVOS-PIPELINE-1 §2), each process
answers from its own slice; consumers aggregate.

Two read-only topics:

### 10.1 `ovos.intent.list`

Lists registered intents. Request payload:

```json
{ "skill_id": "music.skill", "lang": "en-US", "session_id": "satellite-abc" }
```

All fields are **optional filters**: omitting `skill_id` returns every
skill's intents; omitting `lang` returns every language; omitting
`session_id` returns intents from all sessions (global view). When
`session_id` is provided the response returns the **effective pool**
for that session: `"default"` intents plus session-specific intents
(§11.2), not the raw index for that session alone. An intent
registered under both methods (§3.2) appears as two entries
distinguished by `method`.

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
      "enabled": true,
      "session_id": "default"
    }
  ]
}
```

Each entry carries `skill_id`, `intent_name`, `lang`, a `method` of
`"keyword"` or `"template"` (INTENT-3 §2), an `enabled` boolean
(§8.5), and the `session_id` under which the intent was registered.
Reserved-name registrations are malformed (§3.2) and do not appear
in the manifest.

### 10.2 `ovos.intent.describe`

Returns the full definition of one intent. Request payload:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US", "method": "template" }
```

`method` is an **optional filter**: `"keyword"` or `"template"`. When
omitted, the response returns every registered method for the triple.

Response (`ovos.intent.describe.response`):

- On success, `{ "ok": true, "definitions": [ { "method": "...", "definition": {...} }, ... ] }`
  where each `definition` is the §5 or §6 payload as it was broadcast.
  The array carries one entry when `method` was specified or only one
  method was registered, two entries when both methods exist and no
  filter was given. When two entries are returned, the orchestrator
  **MUST** emit them in the order `keyword`, `template` so consumers
  can rely on positional access.
- On unknown intent, `{ "ok": false, "error": "..." }`.

The orchestrator **MAY** restrict access to introspection topics;
authorization is out of scope.

---

## 11. Session-scoped registration

### 11.1 Every registration is session-keyed

The orchestrator keys every registration by the `session_id` it
reads from `context.session.session_id` — the
`context` field of the bus Message envelope, never from
`Message.data`. This is a strict requirement: `session_id` in
`data` would allow a producer to register intents under an arbitrary
session it does not own. Reading from `context` means the
`session_id` is set by the session the producer is running under,
not by anything the producer chooses to assert in its payload.

No change to the registration message shape is required — every bus
Message already carries a session in `context` (OVOS-MSG-1). The
full registration key becomes the quintuple
`(session_id, skill_id, intent_name, lang, method)`; the prior
quadruple `(skill_id, intent_name, lang, method)` is the special
case where `session_id == "default"`.

Skills running on the local device register under `"default"` because
the local device uses the default session (OVOS-SESSION-2 §5). Skills
running on a remote satellite register under whatever `session_id`
the satellite's session carries. No new message, no new field, no
coordination protocol.

### 11.2 Inheritance — `"default"` is the global scope

The **effective intent pool** for a session X is:

```
pool(X) = { intents registered under "default" }
        ∪ { intents registered under session_id == X }
        − { entries blacklisted by session X's blacklist fields }
```

Every session implicitly inherits the full `"default"` set.
Session-scoped registrations extend the pool — they never narrow it.
Narrowing is the exclusive job of the `blacklisted_skills`,
`blacklisted_intents`, and `blacklisted_pipelines` session fields
(OVOS-PIPELINE-1 §5, OVOS-SESSION-1 §3).

If the same `(skill_id, intent_name, lang, method)` exists in both
`"default"` and session X, both index entries are retained and both
appear in the matching pool. The existing first-match-wins iteration
(OVOS-PIPELINE-1 §6) determines which is used; the blacklist is the
explicit suppression mechanism if the satellite wants to shadow a
default intent.

### 11.3 Deregistration and session teardown

`ovos.intent.deregister` and `ovos.entity.deregister` remove the
entry whose full key matches, including `session_id`. An omitted
`session_id` in the deregistration payload removes the entry from
`"default"` only — it does not remove session-scoped registrations
with the same `(skill_id, intent_name, lang)`.

`ovos.skill.deregister` with an optional `session_id` field (§8.4)
removes all registrations for that skill scoped to that session. A
bridge SHOULD emit `ovos.skill.deregister` with the satellite's
`session_id` for each satellite skill when the satellite disconnects,
to clean up the satellite's session-scoped registrations from the
orchestrator's index.

### 11.4 Pipeline plugin visibility

A pipeline plugin that wishes to support session-scoped matching
SHOULD receive the effective pool for the current session's
`session_id` when performing a match, i.e. the union described in
§11.2. Plugins that do not implement session-scoped matching
continue to operate against the `"default"` pool only and remain
conformant; they simply cannot match session-specific intents.

How the orchestrator communicates the effective pool to a plugin is
an implementation concern outside this specification. The normative
requirement is that the pool delivered to a matching plugin for
session X MUST include all entries satisfying §11.2.

### 11.5 Dispatch routing for session-scoped skills

When the orchestrator dispatches a session-scoped intent — one
registered under a non-default `session_id` — the dispatch Message
is a `.reply()` of the inbound utterance, which sets
`context.destination` to the originating participant's `source`. A
bridge conformant with OVOS-BRIDGE-1 §3.2 will route that dispatch
back to the satellite that owns the session. No special routing
protocol is needed; the existing destination-based routing
(OVOS-MSG-1 §3, OVOS-BRIDGE-1 §3.2) handles it transparently.

---

## 12. Conformance

### A **skill** (producer of registration messages) **MUST**:

- emit each registration through the topic that matches its
  definition method (§5 for keyword, §6 for template); a single
  intent **MAY** be registered under both methods if the skill has
  training data of both kinds (§3.2);
- include the identity fields of §3.2 in every registration's `data`;
- set `Message.context["skill_id"]` to its own `skill_id` on every
  Message it emits, per §3.1;
- conform every registration's payload to §5 (keyword), §6 (template),
  or §7 (entity), respectively;
- emit `ovos.intent.deregister` / `ovos.entity.deregister` /
  `ovos.skill.deregister` to retract its registrations, paired with
  the local release of the handler (§9, INTENT-3 §6.1);
- conform its underlying templates, vocabularies, and entities to
  OVOS-INTENT-1 and OVOS-INTENT-2.

A skill **SHOULD** query the manifest (§10) to confirm a
registration landed; there is no acknowledgement.

### A **pipeline plugin** (consumer) **MAY**:

- subscribe to any subset of the registration topics and consume
  what fits its matching strategy — a plugin that consumes none
  and matches by internal rules (e.g. an LLM persona) is also
  conformant.

A plugin **MUST NOT** index a malformed registration (§§5.3, 6.2,
6.3, 7.2 — including registrations whose `intent_name` is reserved,
§3.2) and **MUST** log every such rejection at WARN with `skill_id`,
`intent_name`/`entity_name`, `lang`, the rejecting topic, and a
one-line reason — fire-and-forget means this log is the producer's
only debugging signal. An individual malformed template, sample, or
entity entry within an otherwise valid registration is skipped and
logged, never grounds for rejecting the registration (§§5.3, 6.3,
7.2). Matching behaviour beyond that is OVOS-PIPELINE-1's concern.

### The **orchestrator** **MUST**:

- subscribe to every registration topic (§§5–8) and maintain the
  **manifest** — a passive index built from observed broadcasts;
- key every manifest entry by the quintuple
  `(session_id, skill_id, intent_name, lang, method)`, reading
  `session_id` from `context.session.session_id` of the registration
  message (§11.1);
- serve `ovos.intent.list` and `ovos.intent.describe` queries
  against the manifest, returning the shape of §10.1 / §10.2;
  when the query includes a `session_id`, return the effective pool
  for that session per §11.2;
- treat a re-registration with the same quintuple as replacement of
  the prior manifest entry (§8.1); other `session_id`s, languages,
  and methods for the same intent are unaffected;
- honour `ovos.intent.enable` / `ovos.intent.disable` in the
  manifest (§8.5) — the `enabled` field of §10.1 reflects the
  latest state;
- on receiving `ovos.skill.deregister` with a `session_id` field,
  remove all manifest entries for that `(session_id, skill_id)` pair
  (§8.4, §11.3);
- **NOT** validate, reject, route, or gate any registration message.
  The orchestrator is a passive listener for the manifest, not a
  routing party.

The orchestrator's other responsibilities — matching, dispatch,
handler lifecycle, utterance lifecycle — live in OVOS-PIPELINE-1.

---

## See also

- *Bus Message Specification* (OVOS-MSG-1) — the envelope every
  message here travels in, the shared identifier-component rule
  (§2.1.1) bounding `skill_id` / `intent_name`, the `destination`
  and `session` keys used throughout, and the `forward` / `reply`
  / `response` derivations.
- *Session Specification* (OVOS-SESSION-1) — the wire shape of
  `session` carried on every registration broadcast.
- *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
  — the orchestrator's contract: pipeline-plugin model, utterance
  lifecycle, match-result notification, dispatch, handler-lifecycle
  trio, terminal events. This spec sits next to PIPELINE-1; together
  they cover the full skill ↔ orchestrator ↔ plugin path.
- *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  concept, identity, definition methods, and match result that this
  specification carries on the bus.
- *Locale Resource Formats Specification* (OVOS-INTENT-2) — the
  authoring file formats a skill loader expands into inline samples
  before emitting a registration.
- *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the
  grammar of the `samples` strings carried in every registration
  payload.
