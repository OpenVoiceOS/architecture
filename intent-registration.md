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
  served by the orchestrator's passive registration index (§10).

It does **not** define:

- the intent concept itself — see OVOS-INTENT-3;
- how plugins *implement* registration storage, matching, or
  consumption — black box (OVOS-PIPELINE-1);
- the **handler reference** itself (§9): the binding between a
  registered intent and the code that runs it is local to the skill
  and never crosses the bus;
- **session lifecycle** — `session` is carried opaquely per
  OVOS-MSG-1;
- **language fallback** — when no registration matches the utterance
  language exactly. Fallback policy is out of scope, expected in a
  future specification together with text normalization
  (OVOS-INTENT-1 §5.3 and the planned text-normalization spec);
- the **utterance lifecycle**, **match-result notification**,
  **dispatch**, **handler-lifecycle trio**, or the
  **intent-layer failure signal** (`complete_intent_failure`) — all
  formalized by OVOS-PIPELINE-1, the spec that owns the
  orchestrator's contract;
- the mapping from any predecessor topic names to the topics
  defined here — implementation history, documented in
  [APPENDIX.md](APPENDIX.md).

---

## 2. Architectural model — registrations are broadcast

Registration messages defined here are **broadcast** on the bus,
like every other Message. There is no central party that owns,
validates, or routes them. Whether any loaded pipeline plugin
(OVOS-PIPELINE-1) chooses to consume a given registration is a
plugin concern, out of scope for this specification.

Because registrations are broadcast and consumption is plugin-
discretionary, a producer **MUST NOT** rely on any specific
plugin acknowledging a registration. A skill that emits
`ovos.intent.register.template` (§6) and no loaded plugin
consumes it: the registration is silently dropped. The skill's
intent will not match utterances. There is no error event; this
is the deployment's responsibility to debug (typically by
checking which plugins are loaded against the kinds of
registrations the skill emits).

The **orchestrator** (OVOS-INTENT-3 §6.1) maintains the manifest
(§10) — a passive index built from observed registration
broadcasts — and serves introspection queries from it. The
manifest is observability-only: it does **not** gate matching,
does **not** influence what any plugin consumes, and does **not**
prevent a skill from re-registering or de-registering at any
time.

Per OVOS-PIPELINE-1 §2, the orchestrator MAY be split across
cooperating processes; each maintains its own slice of the
manifest built from broadcasts it observed and responds
independently to §10 queries.

Three consequences of this model:

- **Plugins are observably pluggable.** Adding, removing, or replacing
  a plugin is a deployment concern; the bus traffic for registrations
  and the orchestrator's index are unaffected.
- **There is no method-rejection handshake.** Registrations that no
  plugin claims have no error code, no negative response, no signal at
  all. Plugins that *do* claim a registration **MAY** respond with the
  error codes of §3.4 if the payload is malformed or otherwise
  unservable, but only the consuming plugin can do so.
- **The plugin pipeline is what enforces "at most one match per
  utterance."** This specification does not enforce it; OVOS-PIPELINE-1
  §5 does (first-match-wins iteration).

---

## 3. Identity, responses, and error codes

### 3.1 Skills self-identify on every emission

A skill **MUST** set `Message.context["skill_id"]` on **every
Message it emits**, to its own `skill_id` (OVOS-INTENT-3 §3). This
applies to registration messages (§§5–8), to handler emissions
under dispatch (responses, speech, side-effect emissions on other
topics), and to any other bus traffic the skill originates — there
is no exemption for any topic.

`Message.context["skill_id"]` is the **authoritative attribution
key** for skill-originated bus traffic. It lets observers
(loggers, audit, analytics, telemetry, debug tooling, the
orchestrator's manifest of §10) attribute any Message to its
originating skill without parsing topic names or `data` payloads —
which would otherwise be infeasible for skill emissions on
non-`<skill_id>:<intent_name>`-shaped topics (e.g. `speak`,
`enclosure.eyes.color`, custom skill-defined topics).

#### `context["skill_id"]` vs `data["skill_id"]`

The two are **different fields with different semantics**, and
**MUST NOT** be conflated:

- **`Message.context["skill_id"]`** — the **emitter**. Identifies
  the skill that produced this Message. Set by the emitting skill
  on every Message it emits, per the rule above. Read by observers
  for attribution.
- **`Message.data["skill_id"]`** — the **subject** of the topic.
  Whenever a topic's payload schema (defined by some other spec)
  carries `skill_id` as a payload value, that value identifies the
  skill the topic is *about* — a search filter, a target of an
  operation, an intent registration's owning skill — not the
  emitter. Examples: `ovos.skill.deregister` carries
  `data.skill_id` for the skill *being deregistered* (which may be
  emitted by the skill itself or by another component on its
  behalf); `ovos.intent.list` carries an optional
  `data.skill_id` filter.

A consumer reading `data.skill_id` is reading a payload value
whose meaning is fixed by the topic's owning spec. A consumer
reading `context.skill_id` is reading the emitter. The two
**MAY** differ on a single Message (a tool component
deregistering on behalf of a skill, a debug harness re-emitting a
skill's prior registration); a consumer that needs the emitter
**MUST** read `context.skill_id`, not `data.skill_id`.

#### Orchestrator-side enforcement

The orchestrator (or any component that loads skills) **MUST**
enforce the `context["skill_id"] == emitting skill's skill_id`
invariant **whenever it is in a position to do so** — typically
by intercepting / decorating the skill's emit pathway at load
time, so that even a non-compliant handler cannot emit a Message
that lacks or misstates `context["skill_id"]`. This places the
discipline on the skill-loading infrastructure rather than on
every skill author, and survives buggy or malicious handler code.

When enforcement is not possible (a skill emitting on a transport
the loader cannot intercept), the rule still binds the skill, and
non-conformant traffic remains diagnosable via the consumer-side
absence-detection rule below.

#### Other component types

The orchestrator and other infrastructure components are not
bound by this rule — they do not have a `skill_id`. Other
component types identify themselves through component-specific
reserved context keys their owning specifications define
(`pipeline_id`, `transformer_id`, future entry-service identity,
etc.). The rule above is **specifically a skill discipline**.

`source` (OVOS-MSG-1 §3.2) is **not** an identity surface for any
component. It is opaque metadata typically populated by the
originator of an entry-point Message and propagated by the
derivations of OVOS-MSG-1 §5; this specification does not
prescribe its value space.

#### Consumer-side

A consumer **MUST NOT** infer the originating skill from `data`
fields whose presence is not normatively required, or from the
topic name. The presence of `context["skill_id"]` is the only
authoritative attribution surface for skill-originated Messages.

A Message that arrives with no `context["skill_id"]` is either not
skill-originated, or is from a non-conformant skill that escaped
loader-side enforcement; the orchestrator **SHOULD** log this for
diagnostics but **MUST NOT** reject the Message at this layer
(rejection on a per-topic basis is the responsibility of that
topic's owning spec — e.g. CONTEXT-1 §5.2 rejects context-mutation
events that lack the field).

### 3.2 Identity carried by every registration message

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

### 3.3 Responses are optional

Registration topics defined here **MAY** have a `.response` reply
(OVOS-MSG-1 §5.3). When emitted, a response is correlated to its
request by topic and `session` (OVOS-MSG-1 §5.4) — no per-message
identifier is used.

Responses, when present, take this shape:

Success:

```json
{ "ok": true }
```

Rejection:

```json
{ "ok": false, "error_code": "malformed_payload", "error": "free-form human-readable detail" }
```

Whether to emit a response is a plugin-level decision. A plugin that
chose to consume a particular registration **MAY** emit `ok: true` on
success or `ok: false` with a normative `error_code` (§3.4) on
rejection. A plugin that did **not** consume a registration emits
nothing.

Producers (skills) **MUST NOT** require any response and **MUST
NOT** block waiting for one. A registration that produces no
`.response` may have been silently dropped (no plugin consumed it),
may have been accepted by a plugin that doesn't bother to confirm,
or may have been accepted by multiple plugins that each respond
differently. The bus is async; the producer is responsible for its
own observability and bookkeeping.

The introspection topics of §10 (`ovos.intent.list` /
`ovos.intent.describe`) are the supported way to verify a
registration landed — they query the orchestrator's manifest, which
the orchestrator maintains regardless of whether any plugin
emitted a `.response`. A producer wanting acknowledgement queries
the manifest after registering; it does not wait on `.response`.

### 3.4 `error_code` values

The `error_code` enum is normative; new codes **MUST** be added by a
future version of this specification, not invented per-deployment.
These codes are **plugin-emitted** on `.response` rejections (§3.3);
no party in this specification synthesizes them on a producer's
behalf.

| `error_code` | Meaning |
|--------------|---------|
| `malformed_payload` | The Message's `data` violates the shape required by the topic (§§5–8). The plugin received the message and judged the shape invalid. |
| `unknown_intent` | A deregister, enable, or disable request references an `(skill_id, intent_name, lang)` triple that the responding plugin has no record of. |
| `unknown_entity` | Analogous to `unknown_intent`, for entities. |
| `unknown_skill` | A `skill_id`-only request references a skill the responding plugin has no record of. |

A plugin that emits an `unknown_*` rejection is reporting *its own*
state; another plugin that did consume the original registration
might have a different view. Producers should not interpret an
`unknown_*` rejection as "no plugin owns this" — only as "this
particular plugin doesn't."

**Idempotent deregistration.** A plugin that receives a deregister,
enable, or disable request for an entity it has no record of
**SHOULD** respond with the corresponding `unknown_*` error code,
but **MAY** alternatively treat the operation as a no-op success
(`{ "ok": true }`) when the request would be idempotent — most
commonly during a skill's shutdown sequence, where deregistering an
already-cleared intent is the intended terminal state. Producers
that want idempotent removal **MAY** ignore `unknown_intent` /
`unknown_entity` / `unknown_skill` codes specifically and treat
them as the operation having already completed.

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

The orchestrator-level match notification, dispatch, and
handler-lifecycle topics that some readers may expect to see here are
defined in OVOS-PIPELINE-1 (§§9–10 there), not in this spec.

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

`name` is the vocabulary name (INTENT-3 §4.1) — this is the key under
which the vocabulary's captured phrase appears in the match result
(OVOS-PIPELINE-1 captures map; INTENT-3 §4.3). Exactly one of
`samples` or `file` **MUST** be present.

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

A orchestrator **MUST** reject any registration violating these rules with
`error_code: "malformed_payload"` (§3.3).

### 5.4 No `.blacklist`

A `.blacklist` is **not** used with keyword intents. The `excluded` role is
the keyword-intent suppression mechanism (INTENT-3 §4.2, §5.4).

### 5.5 File form is single-orchestrator only

A `file:` reference is a filesystem path that **MUST** be resolvable by the
orchestrator. In a deployment where producer and orchestrator share a filesystem (typical
single-machine installs), file form is fully equivalent to inline. In
distributed deployments — separate containers, separate hosts, or any case
where the producer's filesystem is not visible to the orchestrator — producers
**MUST** use the inline form (`samples`). A orchestrator that cannot resolve a
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

Or, using the file form (INTENT-1 §6.1; single-orchestrator only, §5.5):

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
| `file` | string | exactly one of `samples`/`file` | Absolute path to a `.intent` resource (INTENT-2 §4.1). Single-orchestrator only (§5.5). |
| `blacklist` | array of strings | no | Slot-free phrases (INTENT-2 §4.3) whose occurrence suppresses the match (INTENT-3 §5.4). |
| `blacklist_file` | string | no | Absolute path to a `.blacklist` file. At most one of `blacklist`/`blacklist_file` may be present. Single-orchestrator only (§5.5). |

### 6.2 Slot-consistency

Every template in `samples` (or in the file) **MUST** declare the same set of
named slots — the slot-consistency rule of INTENT-1 §5.5. A orchestrator **MUST**
reject a registration that violates it with `error_code: "malformed_payload"`.

### 6.3 Malformed payloads

A orchestrator **MUST** reject (with `error_code: "malformed_payload"`) a template
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

Or (single-orchestrator only, §5.5):

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
| `file` | string | exactly one of `samples`/`file` | Absolute path to a `.entity` resource (INTENT-2 §4.4). Single-orchestrator only (§5.5). |

### 7.2 Malformed payloads

A orchestrator **MUST** reject an entity registration with
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

This is the message a orchestrator emits, or that a skill sends to the orchestrator, when a
skill is unloaded (INTENT-3 §6.1).

Deregistering an intent, entity, or skill that is not currently registered
**MUST** be rejected with `error_code: "unknown_intent"`,
`"unknown_entity"`, or `"unknown_skill"` respectively — silent no-ops mask
bugs. A producer that wants idempotent removal can ignore those specific
error codes.

### 8.5 `ovos.intent.enable` and `ovos.intent.disable`

A registered intent is, by default, **enabled** — eligible for matching. A
skill **MAY** temporarily **disable** an intent without removing it; the
orchestrator retains the definition and handler binding but excludes the intent
from match candidacy until it is re-enabled. Both topics share the same
payload as `ovos.intent.deregister` (§8.2), and `lang` semantics:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

If `lang` is omitted, every language for that `(skill_id, intent_name)` is
affected.

Enabling an already-enabled intent, or disabling an already-disabled
intent, is **not** an error — the orchestrator **MAY** respond `{ "ok": true }`.
Re-registration (§8.1) preserves enabled/disabled state unless the producer
deregisters first.

---

## 9. The handler reference is not on the bus

Per INTENT-3 §6.1, the **handler reference** — the actual code object that
runs when the intent matches — is **not** part of any registration message
and **never** crosses the bus. It is held locally by the skill process.

This specification puts only the intent **definition** (§§5–7) on
the bus. The dispatch message that triggers the handler, the
match-result notification, and the handler-lifecycle messages are
all defined in OVOS-PIPELINE-1.

Together the registration payloads and the PIPELINE-1 dispatch
contract let a skill in a different process from the orchestrator
host its handlers across the bus, without ever serializing the
handler itself. This is the key contract that lets local and remote
skills look the same from outside.

---

## 10. Introspection — the orchestrator-owned manifest

Registration topics of §5–§8 are load-time **announcements**:
when a skill loads, it emits one `ovos.intent.register.*` per
intent it owns. A consumer that subscribed before the skill
loaded receives those announcements in real time. A consumer that
started later — a monitoring tool started mid-session, a pipeline
plugin loaded after the skill, a new orchestrator process joining
a split deployment (OVOS-PIPELINE-1 §2) — has missed them; the
bus is asynchronous with no catch-up channel for missed
broadcasts.

The introspection surface this specification defines is the
**orchestrator-owned manifest** of intents observed on the bus.
**Skills have no introspection obligation** under this
specification — they emit their registrations once at load and
move on. The orchestrator that observes those registrations
maintains the read-side manifest and answers queries against it.

For per-pipeline-plugin detail — *which* intents are currently
loaded inside a particular pipeline plugin's matcher, as opposed
to *what skills have declared on the bus* — see OVOS-PIPELINE-1
§10 (per-`pipeline_id` introspection). The two surfaces are
distinct: the orchestrator's manifest is observability of
declared intents; PIPELINE-1 §10 is observability of compiled
plugin state. A consumer wanting both queries both.

When the orchestrator is split across cooperating processes
(OVOS-PIPELINE-1 §2), each process answers from its own
observed-broadcast slice. The composition of all per-process
responses is the orchestrator's full manifest.

**Pull-query is the source of truth.** A consumer that needs
accurate state **MUST** issue `ovos.intent.list` /
`ovos.intent.describe` and **MUST NOT** assume that any prior
`ovos.intent.register.*` broadcast reached it. Registration
broadcasts are convenience for already-subscribed observers; they
are not delivery-guaranteed and a consumer that started after a
skill loaded missed them.

Two read-only topics:

### 10.1 `ovos.intent.list`

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

### 10.2 `ovos.intent.describe`

Returns the full definition of one intent. Request payload:

```json
{ "skill_id": "music.skill", "intent_name": "play_music", "lang": "en-US" }
```

Response (`ovos.intent.describe.response`):

- On success, `{ "ok": true, "method": "keyword"|"template", "definition": {...} }`
  where `definition` is the §5 or §6 payload (file forms expanded to inline
  where possible).
- On unknown intent, `{ "ok": false, "error_code": "unknown_intent", "error": "no such (skill_id, intent_name, lang) registered" }`.

The orchestrator **MAY** restrict who is permitted to call
introspection topics; authorization is not defined by this
specification.

---

## 11. Conformance

### A **skill** (producer of registration messages) **MUST**:

- emit each registration through the topic that matches its
  definition method (§5 for keyword, §6 for template), never both for
  a single intent (INTENT-3 §2);
- include the identity fields of §3.2 in every registration's `data`;
- set `Message.context["skill_id"]` to its own `skill_id` on every
  Message it emits, per §3.1;
- conform every registration's payload to §5 (keyword), §6 (template),
  or §7 (entity), respectively, including §5.5's single-host rule for
  `file:` references;
- emit `ovos.intent.deregister` / `ovos.entity.deregister` /
  `ovos.skill.deregister` to retract its registrations, paired with
  the local release of the handler (§9, INTENT-3 §6.1);
- conform its underlying templates, vocabularies, and entities to
  OVOS-INTENT-1 and OVOS-INTENT-2.

A skill **SHOULD NOT** assume any specific plugin will consume its
registration. Producers responsible for delivery confirmation
**SHOULD** query (§10) rather than wait for `.response` events.

A skill carries **no introspection obligation** under this
specification — it has no §10 query-response responsibility. The
orchestrator owns the manifest (see below).

### A **pipeline plugin** (consumer of registration messages) **MAY**:

- subscribe to any subset of the registration topics defined here and
  consume what fits its matching strategy. A plugin that consumes
  `ovos.intent.register.template` only is conformant; a plugin that
  consumes none of them and matches utterances by its own internal
  rules (e.g. an LLM persona) is also conformant.
- emit `.response` messages per §3.3 to confirm or reject what it
  consumed. Such responses, when emitted, **MUST** use the error
  codes of §3.4.

A plugin **MUST NOT** synthesize a `.response` for a registration it
did not actually consume — `.response` is the consumer's
acknowledgement, not a routing decision.

The plugin's matching behaviour, lifecycle, and bus emissions
beyond `.response` are out of scope for this specification — see
OVOS-PIPELINE-1.

### The **orchestrator** **MUST**:

- subscribe to every registration topic (§§5–8) and maintain the
  **manifest** — a passive index built from observed broadcasts;
- serve `ovos.intent.list` and `ovos.intent.describe` queries
  against the manifest, returning the shape of §10.1 / §10.2;
- treat a re-registration with the same key as replacement of the
  prior manifest entry (§8.1);
- honour `ovos.intent.enable` / `ovos.intent.disable` in the
  manifest (§8.5) — the `enabled` field of §10.1 reflects the
  latest state;
- **NOT** validate, reject, route, gate, or synthesize
  `.response` messages for any registration message. The
  orchestrator is a passive listener for the manifest, not a
  routing party.

When the orchestrator is split across cooperating processes
(OVOS-PIPELINE-1 §2), each process maintains its own slice of the
manifest built from broadcasts it observed and responds
independently to §10 queries; consumers aggregate the slices.

For per-pipeline-plugin detail (which intents a particular
plugin's matcher has compiled), consumers query
OVOS-PIPELINE-1 §10 directly against the responsible
`pipeline_id`. The orchestrator's manifest under this
specification is the declared-intents view; PIPELINE-1 §10 is the
compiled-state view.

The orchestrator's other responsibilities — matching, dispatch,
handler lifecycle observation, utterance lifecycle — are defined
by OVOS-PIPELINE-1.

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
  resource files referenced by the `file` form of each registration.
- *Sentence Template Grammar Specification* (OVOS-INTENT-1) — the
  grammar of the inline `samples` form, and the file-or-inline
  training-data contract.
