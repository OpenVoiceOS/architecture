# Intent Context Specification

**Spec ID:** OVOS-CONTEXT-1 · **Version:** 2 · **Status:** Draft

This document defines **intent context**: a session-scoped, decaying
key/value store that skills use to bias or **gate** future intent
matching across conversational turns. It defines positive and
negative gating declarations on intents and the mutation pathways
by which components write context entries. It is engine-agnostic:
every intent engine that consumes intent definitions honours the
same gating contracts, and engines that wish to do so MAY
additionally use context entries as matching hints.

It builds on five companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope, the
  `session` carrier in which context lives (§4), and the
  `forward` derivation used by the `ovos.session.sync` mutation
  pathway (§5.3);
- the *Session Specification* (OVOS-SESSION-1) —
  the field-registry mechanism under which this spec claims
  `session.intent_context` (§2);
- the *Session Lifecycle and State Ownership Specification*
  (OVOS-SESSION-2) — the
  `ovos.session.sync` topic and its merge semantics (§5.3);
- the *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  definition this spec extends with a `requires_context` declaration;
- the *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
  — the orchestrator that performs the decay tick and enforces the
  gating contract before each match round.

The key words **MUST**, **MUST NOT**, **SHOULD**, **MAY**, and
**RECOMMENDED** are used as in RFC 2119.

---

## 1. What intent context is

**Intent context** is a collection of named entries attached to a
conversational session. Each entry is a small fact the assistant
remembers across turns — "the user is asking about Bob", "we are
inside a confirmation dialog", "the current room is the kitchen" —
that other turns may consult.

Two things are true of every context entry:

- it **decays** — by elapsed time, by remaining turns, or both, until
  the orchestrator removes it;
- it is **engine-agnostic** — its meaning to an intent engine is
  fixed by this specification, not by any one engine's
  implementation.

Intent context is the mechanism by which a skill's matching surface
can depend on **what just happened**, without the skill having to
inspect transcripts, query other skills, or hard-code multi-turn
state machines into every intent.

### 1.1 Four uses of the word "context" — normative disambiguation

The word *context* appears in **four** distinct places across the
specification set. Conflating them produces real bugs. A consumer
**MUST** distinguish them by the table below and **MUST NOT** treat
any pair as interchangeable:

| Name | Defined in | What it is | JSON path |
|------|------------|------------|-----------|
| `Message.context` | OVOS-MSG-1 §2.3 | The envelope's metadata object on every Message — routing keys, the `session` carrier, tracing identifiers. | `context` |
| `session.intent_context` | this spec §2 | A field **inside** the session carrier; the JSON object that holds intent-context entries. | `context.session.intent_context` |
| **Intent context** (the term) | this spec §1.2 | The decaying key/value state itself — i.e. the entries inside `session.intent_context`. | (entries of `context.session.intent_context`) |
| `Match.slots` | OVOS-PIPELINE-1 §4.3 | The slot-name → value map produced **at match time** for a single intent dispatch — entirely unrelated to `session.intent_context` despite both being key/value maps. | (`data.slots` on the dispatch Message) |

The two `context`s are not nested under each other except
incidentally (intent context happens to ride inside the
`Message.context` envelope because the `session` carrier does).
A consumer reading `Message.context["foo"]` is not reading intent
context; a consumer reading `Match.slots["foo"]` is not reading
intent context either. This spec uses **intent context** when the
distinction matters; otherwise it cites the JSON path explicitly.

### 1.2 Intent context and continuous dialog

A continuous dialog is a sequence of utterances in one session that
depend on each other — a follow-up question, a confirmation, a slot
the user is filling step by step. Intent context is this spec's
**declarative** primitive for such flows: a skill records that the
conversation is in some state, and other intents declare — at
definition time — that they only match while that state holds.

The dominant shape is **intra-skill multi-turn flow**. A skill
handles a top-level intent and, while replying, sets a
*flag-context* (an entry whose `value` is `null`). Follow-up
intents in the same skill declare `requires_context` on that flag
— they only match while the flag is live, which is to say "while
the user is in the middle of replying to me". The classic
illustration is a confirmation branch: the top-level intent asks
*"do you want milk with that?"* and sets a `confirming_milk` flag;
a `yes` intent and a `no` intent — both scoped to this skill —
declare `requires_context: ["confirming_milk"]` and therefore only
match in the narrow window between the question and the user's
answer. Outside that window the same `yes`/`no` words have nothing
to attach to, and the skill is silent.

The same mechanism scales to **cross-skill flow** via the shared
scope (§3) and §7's context-supplied capture rule: a skill
publishes a fact (an entry whose `value` is a string, such as the
person the conversation is currently about), and an intent in a
different skill picks it up as a slot capture without the user
having to repeat it. §3.2 works this through end to end.

Intent context is one of several mechanisms an assistant may use to
sustain continuous dialog, and the **only one this spec defines**.
Imperative response-collection and recency-based routing are defined
by the companion *Active Handlers and Interactive Response
Specification* (OVOS-CONVERSE-1) — `session.response_mode`
(OVOS-CONVERSE-1 §2.2) for the imperative response window, and
`session.converse_handlers` (OVOS-CONVERSE-1 §2.1) for the
eligibility list the converse plugin role iterates. The evaluation
order follows from PIPELINE-1's first-match-wins iteration and
pipeline positioning: response-mode pre-empts; converse poll runs
before intent stages; `requires_context` / `excludes_context` apply
only to intent-stage matches. Any other continuous-dialog mechanism
an implementation provides is **out of scope** here.

### 1.3 Scope

This specification defines the context entry shape (§2), the two
scopes a context entry may have (§3), the decay model (§4), the
mutation pathways (§5), the positive and negative gating
declarations on intents (`requires_context`, §6;
`excludes_context`, §6.1), the interaction with the match result
(§7), conformance (§8), and the non-goals around trust and replay
(§9).

It does **not** define how a particular engine *uses* a context
entry's string value as a matching hint — whether as an additional
candidate keyword, an entity hint, a re-ranking signal, or not at
all. The §6 / §6.1 gating contracts and the §7 context-supplied
capture rule are normative; broader use of values as matching hints
is engine-specific.

---

## 2. The context entry

`session.intent_context` is a JSON object — a flat map from **key** (a
string) to **entry** (an object). An absent `session.intent_context` is
equivalent to `{}`.

A context key is one of two shapes:

- A **bare key** — `person`, `in_confirmation`, `active_room`.
  A bare key denotes a **shared** entry, visible to every skill
  (§3).
- A **prefixed key** — `<skill_id>:<key>`, e.g.
  `people.skill:last_query` or `common-qa:last_query`. A prefixed
  key denotes a **private** entry, owned by the `<skill_id>` named
  in the prefix and visible only via the §3 private-scope lookup
  the owner performs. `<skill_id>` is polymorphically a `skill_id`
  or a `pipeline_id`, matching the `<skill_id>:<intent_name>`
  dispatch topic shape of OVOS-PIPELINE-1 §7 — any component that
  can own an intent can own private context.

The `:` is the **single load-bearing separator** between the
owner and the caller-chosen sub-key. A prefixed key contains
exactly one `:`; the `<skill_id>` portion is bound by OVOS-MSG-1
§2.1.1 — it must not contain `:` — and the caller-chosen `<key>`
portion is bound by the same rule, so the split is unambiguous.

Bare keys must not contain `:` (OVOS-MSG-1 §2.1.1); the recommended
form is ASCII letters / digits / `_` / `-` only.

This specification places no length cap on keys; deployers
**SHOULD** choose short, stable names. For **shared** (bare) keys,
which require ecosystem-wide agreement to be useful across skills,
the **RECOMMENDED** form is lowercase with underscores (`person`,
`active_room`, `in_confirmation`). Private keys are scoped to
their owner and may use any valid form.

Because `session.intent_context` is carried inside `session`
(OVOS-MSG-1 §4), orchestrators and skills **SHOULD** keep the
entry set small. An orchestrator **SHOULD** enforce a maximum
entry count (default 1024) and, when exceeded, evict the live
entry closest to natural expiry — smallest `turns_remaining` if
set, then earliest `expires_at`, then arbitrary among entries
with neither.

An entry has the following fields:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `value` | string \| null | yes | The associated value, or `null` to mark the key as a flag (presence only). A non-null value is consumed by §7's context-supplied capture rule; engines MAY additionally use it as a matching hint (§6). |
| `expires_at` | number \| null | no | Absolute expiry time in Unix seconds. If absent or `null`, the entry has no wall-clock expiry. |
| `turns_remaining` | integer \| null | no | Number of subsequent utterance dispatches the entry will survive. If absent or `null`, the entry has no turn-based expiry. |

Scope and ownership are **encoded in the key itself** — a prefixed
key is private to the named owner, a bare key is shared. There is
no separate `scope` field and no separate `origin` field on the
entry; the key carries both.

An entry is **live** iff both of:

- `turns_remaining` is unset, `null`, or strictly greater than `0`;
- `expires_at` is unset, `null`, or strictly greater than the current
  Unix time.

A dead entry **MUST** be removed by the orchestrator before the
next match round (§4) and **MUST NOT** be considered by any
engine.

An entry with neither `expires_at` nor `turns_remaining` set
persists until it is explicitly removed (§5) or the session ends.
Implementations **SHOULD** treat such entries with care: long-lived
context can mask classification errors.

### 2.1 Example

```json
{
  "person": {
    "value": "Bob",
    "turns_remaining": 3
  },
  "people.skill:last_query": {
    "value": "who is Bob",
    "expires_at": 1717000000.0
  },
  "tea.skill:in_confirmation": {
    "value": null,
    "turns_remaining": 1
  }
}
```

Three live entries. `person` is a shared string value (bare key,
§3) visible to every skill. `people.skill:last_query` is private
to `people.skill` (prefixed key). `tea.skill:in_confirmation` is
a private flag owned by `tea.skill` — confirmation state belongs
in private scope so it cannot accidentally be satisfied by a
different skill's shared entry of the same name.

---

## 3. Scopes — private and shared

Scope is **encoded in the key shape**:

| Scope | Visible to | Key shape in `session.intent_context` |
|-------|-----------|--------------------------------|
| `private` | Only intents owned by the named owner (skill or pipeline plugin). | `<skill_id>:<key>` — exactly one `:`, owner before, sub-key after. |
| `shared` | Every owner's intents. | `<key>` — bare, no `:`. |

The `:` is the load-bearing scope marker, mirroring the
`<skill_id>:<intent_name>` dispatch topic shape of
OVOS-PIPELINE-1 §7. A component writing context computes the
stored key directly (§5): private entries use `<own_id>:<key>`,
shared entries use the bare `<key>`. The stored key is the single
source of truth for scope and ownership.

An owner that wants to remember something only for its own
follow-up intents (an in-dialog flag, a last-query value) stores
the entry under its own private prefix. An owner that wants to
publish a fact other components may key off (an entity the
conversation is currently about, a room the user has selected)
stores under a bare shared key.

### 3.1 Gating resolution by scope

The scope of a `requires_context` / `excludes_context` entry
(§6, §6.1) selects which stored key the engine consults:

- **Private** (`scope: "private"`, the default for bare-string
  declarations): the engine **MUST** look up
  `<owning_skill_id>:<key>`, where `<owning_skill_id>` is the
  `skill_id` for skill-owned intents (OVOS-INTENT-3 §3), or
  `pipeline_id` for plugin-owned intents (OVOS-PIPELINE-1 §7). Shared entries
  with the same `<key>` do **not** satisfy a private-scope gate.
- **Shared** (`scope: "shared"`): the engine **MUST** look up the
  bare `<key>`. Private entries with the same `<key>` in any
  owner's namespace do **not** satisfy a shared-scope gate.

A private entry of skill *A* (stored at `A:k`) never satisfies a
gate declared by an intent of skill *B* (which looks at `B:k` for
private or `k` for shared). A skill that wants to depend on
either scope declares two entries, one of each.

### 3.2 Worked examples

#### Intra-skill flag-context (private scope)

A single skill `tea.skill` runs a confirmation branch:

1. The user says **"make me some tea"**. `tea.skill`'s top-level
   intent matches, and while handling the utterance the skill
   asks the question and sets a private flag entry. It writes
   the key `tea.skill:confirming_milk` directly into its local
   copy of `session.intent_context`:

   ```json
   { "value": null, "turns_remaining": 1 }
   ```

   and emits `ovos.session.sync` (§5.3) with the updated session
   snapshot. The orchestrator merges the snapshot; the entry is
   now live at `tea.skill:confirming_milk`.

2. The user says **"yes"**. `tea.skill` has two narrow intents,
   `confirm_milk_yes` and `confirm_milk_no`, each declaring
   `requires_context: ["confirming_milk"]`. §3.1 resolves
   `confirming_milk` for these intents to the private key
   `tea.skill:confirming_milk`, which is live, so the gate is
   satisfied. The matching intent runs; the skill clears the flag
   (or lets it decay) and the conversation moves on.

3. The same word "yes" spoken outside this window matches no intent
   — the gate is not satisfied — so the skill is silent. This is
   the value of the gate: it makes narrow follow-up intents
   *invisible except when relevant*, with no skill-side state
   machine.

#### Cross-skill value-context (shared scope)

A multi-skill conversation:

1. The user says **"who is Bob"**. `people.skill` matches and,
   while handling the utterance, sets a shared entry. It writes
   the bare key `person` into its local copy of
   `session.intent_context`:

   ```json
   { "value": "Bob", "turns_remaining": 3 }
   ```

   and emits `ovos.session.sync` (§5.3) with the updated snapshot.

2. The user says **"how tall is he"**. `bio.skill` has registered a
   template intent `height_query` whose template names a `{person}`
   slot and which declares:

   ```yaml
   requires_context:
     - { key: "person", scope: "shared" }
   ```

   The `scope: "shared"` is required here: the bare-string short
   form defaults to private scope (§6), which would look for
   `bio.skill:person` — a key `people.skill` never set. With
   `scope: "shared"`, §3.1 selects the bare `person` entry, the
   gate is satisfied. The utterance itself does not fill `{person}`
   (the user said "he", not "Bob"), so §7 fills the slot from
   the entry's value: the engine reports a match with
   `slots: { person: "Bob" }`.

3. If only `people.skill` had set `person` privately (stored at
   `people.skill:person`), `bio.skill`'s intent would **not**
   match regardless of the scope declared: §3.1's private-scope
   lookup keys off the *declaring intent's* `skill_id` (queries
   `bio.skill:person`), and a private entry of `people.skill` is
   invisible to a `bio.skill` intent. This is precisely the
   difference between the scopes — shared context is the only
   cross-skill channel.

---

## 4. Decay

Decay runs **once per utterance dispatch**, in two halves bracketing
the match round of OVOS-PIPELINE-1 §6 (the orchestrator's
per-utterance flow over `session.pipeline`).

**Before the match round, the orchestrator MUST:**

- For each entry in `session.intent_context`, remove the entry if it is no
  longer live per §2 (wall-clock expired, or `turns_remaining` is
  set and not greater than `0`).

This is the gating snapshot every matcher sees during this match
round.

**After the match round (whether or not any intent matched), the
orchestrator MUST:**

- For each remaining entry whose `turns_remaining` is set and not
  `null`, decrement it by `1`.

`turns_remaining` therefore counts the **number of subsequent
match rounds the entry will survive**. An entry set with `turns_remaining: 1`
is live for exactly the next match round and is removed before the
one after that. An entry set with `turns_remaining: 0` is dead on
arrival and removed at the next pre-match prune. `turns_remaining: 1`
is the canonical value for "live for the immediate follow-up
utterance" patterns.

This ordering — prune-then-match-then-decrement — makes the gating
contract (§6) deterministic: every matcher in a single utterance
sees the same context snapshot, and entry lifetimes match the
intuitive reading of `turns_remaining`.

Decay applies to entries of both scopes identically.

The post-match decrement runs **whether or not any intent
matched** — an unmatched utterance (`ovos.intent.unmatched`,
PIPELINE-1 §9.3) still decrements every live entry's
`turns_remaining`. This is intentional: the counter tracks
conversational turns, not recognised intents. A confirmation
window set with `turns_remaining: 1` expires after the user's
next utterance regardless of whether that utterance was
understood.

### 4.1 Mid-dispatch mutations

Mutations via `ovos.session.sync` (§5.3) emitted while a dispatch
is in flight take effect **after** the current dispatch's
post-match decrement and **before** the next dispatch's pre-match
prune. They are visible to the matchers of the *next* utterance,
never to any matcher in the current one, and they are not
themselves decremented by the post-match decrement of the dispatch
in which they were emitted (so an entry written with
`turns_remaining: 1` lands alive for exactly the next match round,
as documented in §4).

Engine-side direct session mutations per §5.1 land in the
post-match-pre-dispatch window of OVOS-PIPELINE-1 §6.1 and are
likewise not subject to the current dispatch's post-match
decrement; the next dispatch's pre-match prune and the
match-round-after's post-match decrement see them as freshly-set
entries.

This ordering keeps the per-utterance context snapshot stable for
all matchers in a single match round, removes any ordering
dependency between handler execution and matcher evaluation
within one dispatch, and makes `turns_remaining` arithmetic match
its intuitive reading regardless of where the entry was set.

### 4.2 Session lifecycle is the client's responsibility

Session lifecycle — preservation, resumption, hand-off — is
defined by OVOS-SESSION-2 §3. This spec does not prescribe it.
The orchestrator and engines see whatever `session.intent_context`
arrives with each utterance; the gating contract (§6) and §7's fill
rule apply uniformly to that snapshot. The route the session took
to get here is not material to matching.

---

## 5. Mutation pathways

`session.intent_context` **MUST** only be mutated at the three
boundaries below. The orchestrator MUST NOT apply mutations that
arrive outside these pathways.

In all three cases the emitter computes the stored key directly per
§3: private entries use `<own_id>:<key>` (where `<own_id>` is the
emitter's `skill_id`, `pipeline_id`, or transformer identifier per
OVOS-TRANSFORM-1); shared entries use the bare `<key>`. Both segments must not contain `:`
(OVOS-MSG-1 §2.1.1). An entry written on a key that already exists
replaces it wholesale.

### 5.1 Pipeline plugin — `Match.updated_session`

A pipeline plugin that needs to add or remove entries constructs
the full updated `session.intent_context` map and returns it as
part of `Match.updated_session` (OVOS-PIPELINE-1 §4.2). The
orchestrator applies the snapshot in the post-match-pre-dispatch
window (OVOS-PIPELINE-1 §6.1) — before the dispatch Message is
emitted.

A canonical use is an engine promoting slot captures to private
context at match time, so they are available as gates for
follow-up intents on the very next utterance. A private entry
**MUST** be stored under `<Match.skill_id>:<key>`; shared entries
(bare keys) **SHOULD** be left to the handler to set, since
promoting a capture to shared scope is a deliberate cross-skill
decision.

### 5.2 Transformer — in-place during hook

A transformer (OVOS-TRANSFORM-1 §3) writes or deletes entries
directly in `session.intent_context` on the session object it
holds during its hook. The mutation **MUST** land before the
transformer returns; it then rides forward on every downstream
Message of the same lifecycle.

For private-key attribution: use the matched skill's `skill_id`
when a skill is in scope (intent, dialog, or TTS transformer
stage); use the transformer's own identifier (per OVOS-TRANSFORM-1)
otherwise.

### 5.3 Skill or handler — `ovos.session.sync`

A skill handler that needs to add, update, or remove entries:

1. Takes its local copy of `session` (received on the dispatch
   Message via OVOS-MSG-1 §4).
2. Writes or deletes entries directly in `session.intent_context`,
   computing the stored key per §3.
3. Emits `ovos.session.sync` (OVOS-SESSION-2 §2.7) derived via
   MSG-1 `forward` from the dispatch Message, with
   `Message.data.session` carrying the updated snapshot.

The orchestrator applies `intent_context` from the sync payload
**entry-by-entry**, not as a wholesale replacement:

- A key present in the payload with an entry object **sets or
  replaces** that key in the working map.
- A key present in the payload with a **`null` entry** (the key
  exists but maps to JSON `null`, not an entry object) **removes**
  that key from the working map.
- Keys absent from the payload are left unchanged.

This entry-level merge means concurrent handlers writing
**disjoint keys** do not overwrite each other. Skills using
private-scope keys (`<own_skill_id>:<key>`) are naturally
disjoint by owner; shared-scope keys written by concurrent
handlers **SHOULD** be coordinated by the skills involved to
avoid last-write-wins conflicts.

*There is no read-back API.* A component that wants a specific
decay window **MUST** supply it explicitly at write-time; the only
source of the current timer is the `session` the component
received on its last dispatch.

*Default decay.* An orchestrator **MAY** apply a
deployer-configurable default decay (turn-based, wall-clock, or
both) to entries written without an explicit `turns_remaining` or
`expires_at`, to bound state accumulation. If applied, the default
values are implementation-defined; deployers **SHOULD** consider
both interactive latency (turn-based decay is deterministic across
pauses) and idle expiry (wall-clock decay bounds a device sitting
idle).

**Scope discipline.** A component SHOULD NOT write into another
component's private namespace (keys prefixed with a foreign
`<id>`). A component MAY delete shared entries it did not set only
when doing so is part of its user-visible purpose (an explicit
"forget that" command, end-of-conversation cleanup). Neither
prohibition requires enforcement by the orchestrator; violations
produce incorrect behaviour for the violating component's own
intents.


## 6. The `requires_context` intent declaration

An intent definition (OVOS-INTENT-3 §4 for keyword intents, §5 for
template intents) **MAY** declare a `requires_context` list.

Each entry is either a bare **key** string or an object pairing a
key with an explicit **scope** discriminator:

```yaml
# short form: bare keys, default scope = private
requires_context:
  - person
  - in_confirmation

# long form: explicit scope per entry
requires_context:
  - { key: "person",          scope: "private" }
  - { key: "active_room",     scope: "shared"  }
  - { key: "in_confirmation", scope: "private" }
```

The scope discriminator selects which §3.1 resolution branch the
engine consults:

- `scope: "private"` (the default for bare-string entries) — the
  engine **MUST** look only at the private key
  `<owning_skill_id>:key`. Shared entries with the same key do not
  satisfy the gate.
- `scope: "shared"` — the engine **MUST** look only at the shared
  key `key`. Private entries with the same name in any owner's
  namespace do not satisfy the gate.

A bare-string entry is interpreted as `{ key: <string>, scope:
"private" }`. The default-to-`private` rule is the **safe default**:
an author writing `requires_context: [person]` cannot accidentally
have an unrelated owner's shared `person` entry satisfy a private
gate it never declared.

The gating contract is normative for every intent engine:

> If an intent declares `requires_context: [g1, …, gN]`, an engine
> **MUST NOT** report that intent as matched unless, for **every**
> `gI`, a live context entry exists in the session at the key
> `<owning_skill_id>:gI.key` when `gI.scope == "private"`, or at
> the key `gI.key` when `gI.scope == "shared"`. The check **MUST**
> be made against the post-decay snapshot of §4.

Engines **MAY** additionally consume entries whose `value` is a
non-null string as candidate keywords (keyword engines per
OVOS-INTENT-3 §4) or as candidate entity values (template engines
per OVOS-INTENT-3 §5). This use is **OPTIONAL** and engine-specific
— a conformant engine that ignores values entirely still satisfies
this specification.

An intent that declares no `requires_context`, or declares an empty
list, has no positive context precondition.

The `requires_context` and `excludes_context` (§6.1) fields travel
with the rest of the intent definition. In-process engines read
them from the registration record they receive locally. They are
**optional** declarations of the OVOS-INTENT-3 intent definition,
not enumerated by the OVOS-INTENT-4 registration payloads (§5.2 /
§6.1 of that spec): a skill that uses them attaches them to the
`ovos.intent.register.template` / `.keyword` payload as additional
fields, which OVOS-INTENT-4 §6.3 / §5.3 carry without rejecting
(unknown fields are tolerated, not malformed). An engine that does
not implement OVOS-CONTEXT-1 ignores them and matches as if absent.

### 6.1 The `excludes_context` intent declaration

An intent definition **MAY** declare an `excludes_context` list,
using the same short-or-long entry form as `requires_context`
(§6) with the same default scope `private`:

```yaml
excludes_context:
  - said_hello                                  # private (default)
  - { key: "active_room", scope: "shared" }
  - { key: "in_confirmation", scope: "private" }
```

The negative gating contract is normative for every intent engine:

> If an intent declares `excludes_context: [g1, …, gN]`, an engine
> **MUST NOT** report that intent as matched if, for **any** `gI`,
> a live context entry exists at the key
> `<owning_skill_id>:gI.key` (when `gI.scope == "private"`) or at
> the key `gI.key` (when `gI.scope == "shared"`). The check
> **MUST** be made against the post-decay snapshot of §4.

`requires_context` and `excludes_context` are complementary and
**MAY** both be declared on the same intent. When both are
declared, both contracts apply: a match requires that every
required key be live **and** that every excluded key be absent. A
single key **MUST NOT** appear in both lists for the same intent —
such an intent could never match.

An intent that declares no `excludes_context`, or declares an
empty list, has no negative context precondition.

The negative gate addresses patterns the positive gate cannot
express cleanly — most prominently *fire-once* intents (greet only
once per session: pair `excludes_context: ["said_hello"]` with a
handler that sets `said_hello` on its first run), and *modal
suppression* (suppress a default intent while a more specific
context is active).

---

## 7. Interaction with the match result

OVOS-INTENT-3 §7 defines the match result as a qualified intent
name plus a **slot map**. This specification places exactly one
normative requirement on that map — the *context-supplied slot*
rule below. All other surfacing of context entries is engine-
specific.

**Context-supplied slots (normative).** When a **slot of the
intent's definition** has the name `k` (a template slot per
OVOS-INTENT-3 §5, or a vocabulary name per OVOS-INTENT-3 §4), and a
context entry for `k` has a live non-null `value`, the engine
**MUST** offer that value to its matcher as a **candidate** for
slot `k` **before** matching, and report the resolved value in
`Match.slots[k]` (keyed by `k`, unprefixed). The entry for `k` is
selected by the intent's owner: a private entry `<skill_id>:k`
takes precedence over a shared entry `k`, mirroring §3.1.

This rule is **independent of `requires_context`.** A string-valued
context entry is ambient conversational data: any declared slot or
vocabulary whose name matches a live entry is a candidate for it,
whether or not the intent declares `requires_context`. `requires_context`
and `excludes_context` (§6) are the **flag-gating** declarations —
they decide whether an intent may match; they do not drive slot
fill. A `null`-valued entry (a flag) is never a slot candidate.

The candidate is offered before the match, not applied after it,
because the two engine families both need it there:

- a **keyword** engine gates the match on the presence of the `k`
  keyword; the injected value stands in for that keyword so the
  intent can match even when the utterance does not contain it. A
  value supplied only after a successful match could never have
  enabled the match.
- a **template** engine resolves `{k}` while matching; a value
  supplied after `{k}` has already bound to an utterance token
  cannot correct that binding.

A value the utterance itself produces for slot `k` (a slot the
user filled, a vocabulary phrase that occurred) **MUST** replace
the context candidate: the utterance-produced value wins, and
context fills only what the utterance leaves unresolved.

This is the portable, engine-agnostic mechanism by which a fact
recorded by an earlier turn (`person: "Bob"`) reaches a later turn's
handler as a slot value without the later utterance having to
repeat it. The later intent needs only to name a `{person}` slot;
the live `person` entry fills it when the utterance leaves it
unresolved, and an utterance-produced `person` overrides it.

---

## 8. Conformance

**An orchestrator** (OVOS-PIPELINE-1) **MUST**:

- store `session.intent_context` as the entry map of §2 — entries
  carry only `value`, `expires_at`, `turns_remaining`; scope and
  ownership are encoded in the key shape (§3);
- treat `session.intent_context` in `Message.context.session` on
  ordinary (non-`ovos.session.sync`) Messages as **read-only** —
  the session carrier propagates the current snapshot; only the
  three pathways of §5 write it;
- on receipt of `ovos.session.sync`, apply the `intent_context`
  payload entry-by-entry per §5.3: present entry objects set or
  replace the keyed entry; `null` entries delete the key; absent
  keys are unchanged;
- prune dead entries before the first matcher runs and decrement
  `turns_remaining` after the match round (§4);
- apply `ovos.session.sync` mutations received mid-dispatch after the current post-match decrement and before the next pre-match prune (§4.1);
- pass the post-decay `session` (with `intent_context` reflecting
  the §4 prune-and-decrement state) to each plugin's
  `match(utterances, lang, session)` call (OVOS-PIPELINE-1 §4);

**An intent engine that consumes OVOS-INTENT-3 registrations**
**MUST**:

- honour the positive gating contract of §6 — never report a
  match whose intent declares an unsatisfied `requires_context`,
  resolved per §3.1;
- honour the negative gating contract of §6.1 — never report a
  match whose intent declares an `excludes_context` key that is
  live in the session, resolved per §3.1;
- apply the §7 context-supplied slot rule for every slot or
  vocabulary of the intent's definition whose name has a live
  non-null context entry — offering that value to the matcher as a
  candidate for the slot **before** matching (independent of any
  `requires_context` declaration), and letting an utterance-produced
  value for the same slot replace it;
- read context from the post-decay snapshot the orchestrator
  presents on each `match` call (OVOS-PIPELINE-1 §4).

Such an engine **MAY**:

- consume non-null context values as matching hints for keys that
  do not name a declared slot or vocabulary (§6);
- surface used context entries in `Match.slots` (§7) in cases
  not covered by the §7 normative rule.

An intent engine that consumes no OVOS-INTENT-3 registrations
(a language-model-backed persona, a chatbot, and so on per
OVOS-PIPELINE-1 §1) has no registered intents to gate and is
unaffected by this specification.

**A skill** that uses intent context **MUST**:

- mutate `session.intent_context` only via `ovos.session.sync`
  (§5.3) — the only normative mutation pathway available to
  handlers; direct in-process mutation without syncing has no
  effect on the orchestrator's working session;
- choose the key scope explicitly (§3): `private` prefix for
  skill-internal state, bare key for facts other skills may key
  off;
- not assume any particular engine consumes its values as
  matching hints beyond the §7 fill rule — engines may treat
  context as gates only.

---

## 9. Non-goals — trust and replay

Intent context is **trust-tied to the session that carries it**. It
is not independently authenticated. Two consequences a deployer
**MUST** be aware of:

- **Key-prefix attribution is per-mutation, not retroactive.**
  The prefix on a private key encodes which component wrote the
  entry **in this orchestrator instance, at this moment** — it
  was written by that component when it mutated the session
  directly (§5.1 / §5.2) or synced via `ovos.session.sync`
  (§5.3). It does **not** authenticate entries already present
  in a `session` blob that arrives over the bus — those entries
  are trusted to the extent the session itself is.

- **Sessions are replayable carriers.** Any participant that can
  present a `session` (a remote chat client, a remote satellite,
  a test harness, a layer-2 system per OVOS-MSG-1 §3.4) can
  present its `session.intent_context` along with it — including entries
  fabricated outside this orchestrator, or carried forward from
  an earlier interaction. A participant who can resume a session
  can resume its context.

This is the same threat surface the session identifier already
has. Authenticating session-bound state — proving that a private
entry stored at `<skill_id>:<key>` was actually set by that
`<skill_id>`, in the session named by its identifier, at the time
it claims — is **out of scope** for this specification and belongs
to a future session-security specification.

Deployments that need stronger guarantees (multi-tenant assistants,
hostile-network bus deployments) **SHOULD NOT** rely on intent
context for security-sensitive gating. The gating contract of §6 is
a **classification primitive**, not an authorization primitive.

---

## See also

- *Intent Definition Specification* (OVOS-INTENT-3) — the intent
  definitions this spec extends with `requires_context` (§6).
- *Utterance Lifecycle and Pipeline Specification* (OVOS-PIPELINE-1)
  — the orchestrator that decays context and enforces the gating
  contract.
- *Bus Message Specification* (OVOS-MSG-1) — the envelope, the
  shared identifier-component rule (§2.1.1) bounding context keys,
  and the `session` carrier (§4) in which intent context lives.
- *Session Specification* (OVOS-SESSION-1) — the wire shape of
  `session`, the registry mechanism under which this specification
  claims the `intent_context` field, and propagation semantics.
- *Session Lifecycle and State Ownership Specification*
  (OVOS-SESSION-2) — session
  lifecycle responsibilities; cited by §4.2 for client-side
  session management.
