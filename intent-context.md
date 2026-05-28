# Intent Context Specification

**Spec ID:** OVOS-CONTEXT-1 · **Version:** 1 · **Status:** Draft

This document defines **intent context**: a session-scoped, decaying
key/value store that skills use to bias or **gate** future intent
matching across conversational turns. It defines positive and
negative gating declarations on intents, a three-event bus API for
skills to mutate context. It is
engine-agnostic: every intent engine that consumes intent
definitions honours the same gating contracts, and engines that
wish to do so MAY additionally use context entries as matching
hints.

It builds on four companion specifications:

- the *Bus Message Specification* (OVOS-MSG-1) — the envelope, the
  `session` carrier in which context lives (§4), and the bus events
  skills emit to mutate it;
- the *Session Carrier Wire Shape Specification* (OVOS-SESSION-1) —
  the field-registry mechanism under which this spec claims
  `session.intent_context` (§2);
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
| `Match.captures` | OVOS-PIPELINE-1 §4.3 | The slot-name → value map produced **at match time** for a single intent dispatch — entirely unrelated to `session.intent_context` despite both being key/value maps. | (`data.captures` on the dispatch Message) |

The two `context`s are not nested under each other except
incidentally (intent context happens to ride inside the
`Message.context` envelope because the `session` carrier does).
A consumer reading `Message.context["foo"]` is not reading intent
context; a consumer reading `Match.captures["foo"]` is not reading
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
Specification* (OVOS-CONVERSE-1) — `session.response_mode` for the
imperative response window, and `session.converse_handlers` for the
eligibility list the converse plugin role iterates. The evaluation
order follows from PIPELINE-1's first-match-wins iteration and
pipeline positioning: response-mode pre-empts; converse poll runs
before intent stages; `requires_context` / `excludes_context` apply
only to intent-stage matches. Any other continuous-dialog mechanism
an implementation provides is **out of scope** here.

### 1.3 Scope

This specification defines the context entry shape (§2), the two
scopes a context entry may have (§3), the decay model (§4), the
three bus events that mutate context (§5), the
positive and negative gating declarations on intents
(`requires_context`, §6; `excludes_context`, §6.1), the
interaction with the match result (§7), conformance (§8), and the
non-goals around trust and replay (§9).

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

- A **bare key** — `Person`, `in_confirmation`, `active_room`.
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
**SHOULD** choose short, stable names.

Because `session.intent_context` is carried inside `session`
(OVOS-MSG-1 §4), orchestrators and skills **SHOULD** keep the
entry set small. An orchestrator **SHOULD** enforce a maximum
entry count (default 1024) and evict the oldest live entry when
exceeded.

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
  "Person": {
    "value": "Bob",
    "turns_remaining": 3
  },
  "people.skill:last_query": {
    "value": "who is Bob",
    "expires_at": 1717000000.0
  },
  "in_confirmation": {
    "value": null,
    "turns_remaining": 1
  }
}
```

Three live entries. `Person` is a shared string value (bare key,
§3) visible to every skill. `people.skill:last_query` is private
to `people.skill` (prefixed key). `in_confirmation` is a shared
flag (bare key, no value).

---

## 3. Scopes — private and shared

Scope is **encoded in the key shape**:

| Scope | Visible to | Key shape in `session.intent_context` |
|-------|-----------|--------------------------------|
| `private` | Only intents owned by the named owner (skill or pipeline plugin). | `<skill_id>:<key>` — exactly one `:`, owner before, sub-key after. |
| `shared` | Every owner's intents. | `<key>` — bare, no `:`. |

The `:` is the load-bearing scope marker, mirroring the
`<skill_id>:<intent_name>` dispatch topic shape of
OVOS-PIPELINE-1 §7. A component writing context expresses its
intent through the bus-event `scope` field (§5.1); the
orchestrator turns that into the appropriate stored-key shape
using the emitter's `skill_id` (OVOS-INTENT-4 §3.1) or
`pipeline_id` (OVOS-PIPELINE-1 §3.1) as the prefix for
private entries. The stored key is
the single source of truth for scope and ownership.

An owner that wants to remember something only for its own
follow-up intents (an in-dialog flag, a last-query value) sets
the entry with `scope: "private"`. An owner that wants to
publish a fact other components may key off (an entity the
conversation is currently about, a room the user has selected)
sets `scope: "shared"`.

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

This single-scope lookup is what makes the default-private rule
in §6 the **safe default**: a bare `requires_context: [Person]`
declared by `bio.skill` looks only at `bio.skill:Person` and is
not accidentally satisfied by some other skill's shared `Person`.

### 3.2 Worked examples

#### Intra-skill flag-context (private scope)

A single skill `tea.skill` runs a confirmation branch:

1. The user says **"make me some tea"**. `tea.skill`'s top-level
   intent matches, and while handling the utterance the skill
   asks the question and sets a private flag entry:

   ```
   ovos.context.set { key: "confirming_milk", value: null,
                        scope: "private", turns: 1 }
   ```

   The orchestrator reads `Message.context["skill_id"] = "tea.skill"`
   (OVOS-INTENT-4 §3.1) and stores the entry at the private key
   `tea.skill:confirming_milk`.

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
   while handling the utterance, sets a shared entry:

   ```
   ovos.context.set { key: "Person", value: "Bob",
                        scope: "shared", turns: 3 }
   ```

2. The user says **"how tall is he"**. `bio.skill` has registered a
   template intent `height_query` whose template names a `{Person}`
   slot and which declares `requires_context: ["Person"]`. The
   shared `Person` entry is live, so §3.1 selects it and the gate
   is satisfied. The utterance itself does not fill `{Person}`
   (the user said "he", not "Bob"), so §7 fills the capture from
   the entry's value: the engine reports a match with
   `captures: { Person: "Bob" }`.

3. If only `people.skill` had set `Person` privately (stored at
   `people.skill:Person`), `bio.skill`'s intent would **not**
   match: §3.1's private-scope lookup keys off the *declaring
   intent's* `skill_id` (so it queries `bio.skill:Person`), and a
   private entry of `people.skill` is invisible to a `bio.skill`
   intent. This is precisely the difference between the scopes.

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
match rounds the entry will survive**. An entry set with `turns: 1`
is live for exactly the next match round and is removed before the
one after that. An entry set with `turns: 0` is dead on arrival
and removed at the next pre-match prune. `turns: 1` is the
canonical value for "live for the immediate follow-up utterance"
patterns.

This ordering — prune-then-match-then-decrement — makes the gating
contract (§6) deterministic: every matcher in a single utterance
sees the same context snapshot, and entry lifetimes match the
intuitive reading of `turns_remaining`.

Decay applies to entries of both scopes identically.

### 4.1 Mid-dispatch mutations

Mutations issued via the §5 bus events
(`ovos.context.set` / `.unset` / `.clear`) while a dispatch is in
flight take effect **after** the current dispatch's post-match
decrement and **before** the next dispatch's pre-match prune. They
are visible to the matchers of the *next* utterance, never to any
matcher in the current one, and they are not themselves decremented
by the post-match decrement of the dispatch in which they were
emitted (so a `set { turns: 1 }` emitted from a handler lands with
`turns_remaining = 1` and is alive for exactly the next match
round, as documented in §4).

Engine-side direct session mutations per §5.3 land synchronously
inside the dispatch sequence (between match accept and dispatch
emit) and are likewise not subject to the current dispatch's
post-match decrement; the next dispatch's pre-match prune and the
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

## 5. Bus events — the skill API

The context API is three bus events. Each is produced via OVOS-MSG-1 `forward` from a Message
the emitter is currently handling, so the originating session is
preserved.

| Topic | Direction | Effect |
|-------|-----------|--------|
| `ovos.context.set` | skill → orchestrator | Set or replace a context entry. |
| `ovos.context.unset` | skill → orchestrator | Remove a context entry. |
| `ovos.context.clear` | skill → orchestrator | Remove all entries in the session. |


### 5.1 Payload shapes

`ovos.context.set`:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `key` | string | yes | The caller-chosen sub-key, **before** the orchestrator prepends the `<skill_id>:` prefix for private entries. MUST NOT contain `:` (MSG-1 §2.1.1 — colon is the owner/key separator). |
| `value` | string \| null | no | The entry value; defaults to `null` (flag). |
| `scope` | `"private"` \| `"shared"` | no | Defaults to `"private"`. Tells the orchestrator whether to store under `<emitter_skill_id>:<key>` (private) or under bare `<key>` (shared), per §3. |
| `turns` | integer | no | Sets `turns_remaining`. |
| `ttl_seconds` | number | no | Sets `expires_at` = now + `ttl_seconds`. |

A `set` with neither `turns` nor `ttl_seconds` creates an entry
that decays only on explicit removal or session end (§2). A `set`
on a key that already exists **replaces** the entry wholesale —
fields not supplied revert to their defaults; values do not merge.

*Refreshing decay timers.* A `set` without `turns` or
`ttl_seconds` therefore removes any decay the entry previously
had — re-setting `Person: Bob` without supplying a timer makes the
entry persist indefinitely. To bump the value while preserving the
original timer, the skill **SHOULD** re-supply the timer it wants.

*Default decay.* An orchestrator **MAY** apply a
deployer-configurable default decay (turn-based, wall-clock, or
both) to entries created without an explicit `turns` or
`ttl_seconds`, to bound state accumulation. If applied, the
default values are implementation-defined; deployers tuning them
**SHOULD** consider both interactive latency (turn-based decay is
deterministic across pauses) and idle expiry (wall-clock decay
survives the device sitting idle). If no default is applied,
indefinite-lifetime entries are the caller's responsibility to
clean up.

`ovos.context.unset`:

| Field | Type | Required | Meaning |
|-------|------|----------|---------|
| `key` | string | yes | The caller-chosen sub-key. The orchestrator resolves the stored key per `scope`, same rule as `set`. |
| `scope` | `"private"` \| `"shared"` | no | Defaults to `"private"`. |

`ovos.context.clear` takes no payload. It removes **all** entries
in the session, both private and shared, regardless of which
skill originally set them. This is destructive and **SHOULD** be
reserved for session boundaries — end-of-conversation cleanup,
hard resets, test fixtures. Skills that want narrower removals
use `ovos.context.unset` per key.



### 5.2 Mutation ownership

**Bus-event pathway — orchestrator computes the stored key.** When
a component emits `ovos.context.set` / `.unset` with
`scope: "private"`, the orchestrator **MUST** compute the stored
key as `<skill_id>:<key>`, where `<skill_id>` is the emitter's
identifier read from whichever **component-identity context key**
the emitter's spec requires it to stamp:

- `Message.context["skill_id"]` for skills (OVOS-INTENT-4 §3.1);
- `Message.context["pipeline_id"]` for pipeline plugins
  (OVOS-PIPELINE-1 §3.1);
- any of the six `<type>_transformer_ids` list-valued keys for
  transformers (OVOS-TRANSFORM-1 §1.3) — `audio_transformer_ids`,
  `utterance_transformer_ids`, `metadata_transformer_ids`,
  `intent_transformer_ids`, `dialog_transformer_ids`,
  `tts_transformer_ids`. For these, the emitter's identifier is
  the **last element** of the list (the most recent transformer
  to touch the Message of that type).

The component-identity context keys claimed across the spec set
may **coexist** on a single Message. The orchestrator applies
**most-specific by lifecycle position wins** per
OVOS-TRANSFORM-1 §2 (latest lifecycle stage first). For
list-valued transformer keys, the owner identifier is the
**last element** of the list; for single-string keys (`skill_id`,
`pipeline_id`), the value itself is the owner.

The first key present from this precedence identifies the owner;
lower-ranked keys are not used for stored-key computation.

The prefix source is **never** any field of the `data` payload — a
component therefore cannot impersonate another by populating a
`data` field. When `scope: "shared"`, the orchestrator stores
under the bare `key` without a prefix.

If **none** of the component-identity context keys above is
present, the orchestrator **MUST** treat the event as malformed
and **MUST NOT** apply the mutation; it **SHOULD** log the
violation. Each emitter type's own spec requires the corresponding
field on every Message it originates or modifies in place, so a
wholly unattributed event has escaped its emitter's conformance
discipline.

**`unset` of private entries.** A private `unset` resolves the
target stored key as `<emitter_skill_id>:<key>` by the same rule.
A skill can therefore only `unset` its own private entries by
construction; no separate authorization rule is needed.

**`unset` of shared entries.** Shared context is a noticeboard.
The mechanism permits any skill to issue an `ovos.context.unset`
against a bare shared key, including entries it did not originally
set — this is what makes the noticeboard collaboratively
maintainable. **Skills SHOULD NOT** unset shared entries they did
not set unless removing them is part of the skill's user-visible
purpose (an explicit "forget that" / "never mind" intent, an
end-of-conversation cleanup skill). The originator of a shared
entry is **not recorded** on the entry itself; a consumer wanting
to audit "who set this" subscribes to `ovos.context.set` events
and keys off `Message.context["skill_id"]`.

**`clear`.** `ovos.context.clear` is destructive — it removes all
entries in the session, both private and shared. Same SHOULD
applies: skills issue `clear` only as part of their own
user-visible purpose, never as defensive cleanup of state another
skill may legitimately depend on.

### 5.3 Direct session mutation

The bus events of §5 are the **skill API** for mutating context.
A pipeline plugin (OVOS-PIPELINE-1 §4) or a transformer
(OVOS-TRANSFORM-1 §3) that holds an in-flight session has a more
direct pathway available: it **MAY** mutate `session.intent_context`
directly on the session object it currently has in hand. The
mutated session then rides forward on the next Message that
propagates it (OVOS-MSG-1 §5).

**Timing.** A direct mutation **MUST** happen **before** the next
Message bearing the affected session is emitted. The propagation
rule of OVOS-MSG-1 §4.3 carries whatever snapshot of `session` the
emitting Message holds at emit time; a mutation performed after
that emission does not retroactively reach the consumers of the
already-emitted Message. A plugin that mutates context between
match acceptance and dispatch (OVOS-PIPELINE-1 §7) lands the
mutation on the dispatch Message; a transformer that mutates
context inside its `transform` call lands the mutation on every
downstream Message of the same lifecycle.

**Reach.** A direct mutation lands on the session object the
mutating component holds — typically its own process's working
copy. Other components observe the mutation only through Messages
that propagate the updated session, not by reading any shared
state. Plugins that need other components to react to context
mutations they cannot reach through propagation (sibling
processes, listeners outside the current lifecycle, persistent
session stores) **MUST** use the §5 bus events; direct mutation is
not a substitute for the bus event when the audience extends
beyond the current Message's propagation chain.

A canonical use is an intent engine promoting captured values to
private-scope context entries so they are available as gates for
follow-up intents — without round-tripping through the §5 bus
events.

When a **pipeline plugin** performs such a mutation:

- It **MUST** occur outside the engine's `match` operation
  (OVOS-PIPELINE-1 §4.2 requires `match` itself to be side-effect-
  free). The mutation happens **after** the orchestrator has
  accepted the match and **before** the dispatch is emitted (per
  the timing rule above).
- A private entry **MUST** be stored under
  `<Match.skill_id>:<key>` — using the matched intent's owning
  identifier (OVOS-PIPELINE-1 §4.1) as the prefix. The engine
  **MUST NOT** invent another prefix; the matched intent's owner
  is the only legitimate one.
- Shared entries (bare keys) **SHOULD** be left to the matched
  skill's handler to set via §5 events; promoting a captured
  value to shared scope is a deliberate cross-skill decision.

When a **transformer** (OVOS-TRANSFORM-1 §3) performs a direct
context mutation, the same timing rule applies: the mutation
**MUST** land on `session.intent_context` before the transformer returns,
so it rides forward on every Message the orchestrator derives from
the current lifecycle.

For the private-key prefix: use the skill's `skill_id` when a
skill is in scope (intent/dialog/TTS transformers); use your own
`transformer_id` otherwise. Shared scope (bare keys) is always
available.

Decay (§2 `turns_remaining` / `expires_at`) is the mutator's
choice.

Direct session mutation is **OPTIONAL**. A component that performs
none is fully conformant. A skill that wants context behaviour
independent of engine or transformer participation emits the §5
events itself from its handler.


## 6. The `requires_context` intent declaration

An intent definition (OVOS-INTENT-3 §4 for keyword intents, §5 for
template intents) **MAY** declare a `requires_context` list.

Each entry is either a bare **key** string or an object pairing a
key with an explicit **scope** discriminator:

```yaml
# short form: bare keys, default scope = private
requires_context:
  - Person
  - in_confirmation

# long form: explicit scope per entry
requires_context:
  - { key: "Person",          scope: "private" }
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
an author writing `requires_context: [Person]` cannot accidentally
have an unrelated owner's shared `Person` entry satisfy a private
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
them from the registration record they receive locally; bus-side
engines receive them via the intent registration wire format
defined in OVOS-INTENT-4.

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
name plus a **capture map**. This specification places exactly one
normative requirement on that map — the *context-supplied capture*
rule below. All other surfacing of context entries is engine-
specific.

**Context-supplied captures (normative).** When an intent's
`requires_context` list contains a key `k` that **also names a
slot of the intent's definition** (a template slot per
OVOS-INTENT-3 §5, or a vocabulary name per OVOS-INTENT-3 §4), the
engine **MUST**, before reporting the match:

1. determine the §3.1-selected entry for `k`;
2. if its `value` is non-null **and** the utterance did not itself
   produce a capture for `k`, fill the capture map's `k` from that
   value (keyed by `k`, unprefixed, regardless of whether §3.1
   selected a private or shared entry).

If the utterance itself produced a capture for `k` (a slot the user
filled, a vocabulary phrase that occurred), that utterance-produced
value **MUST** win — context is a fallback signal, not an override.

This is the portable, engine-agnostic mechanism by which a fact
recorded by an earlier turn (`Person: Bob`) reaches a later turn's
handler as a capture without the later utterance having to repeat
it. An intent that wants this behaviour declares the shared key in
`requires_context` **and** names a slot or vocabulary with the same
name in its definition. Intents that declare `requires_context`
keys with no matching slot or vocabulary name are gated only — the
rule above does not apply to them.

---

## 8. Conformance

**An orchestrator** (OVOS-PIPELINE-1) **MUST**:

- store `session.intent_context` as the entry map of §2 — entries
  carry only `value`, `expires_at`, `turns_remaining`; scope and
  ownership are encoded in the key shape (§3);
- compute the stored key for every `ovos.context.set` /
  `.unset`: `<skill_id>:<key>` when `scope: "private"`, bare
  `<key>` when `scope: "shared"`. The prefix source is the
  chain's **most-specific component-identity context key**, by
  lifecycle-position precedence per OVOS-TRANSFORM-1 §2 (latest
  stage wins); the keys may coexist along a derivation chain
  (§5.2);
- reject any `ovos.context.set` / `.unset` carrying **none** of
  the component-identity context keys of §5.2 as malformed (§5.2);
- accept direct session mutations per §5.3, where private entries
  are stored under `<Match.skill_id>:<key>` (for pipeline-plugin
  mutations) or `<acting_skill_id>:<key>` (for transformer
  mutations);
- enforce the §5.2 SHOULD-NOTs around shared `unset` and `clear`
  by logging (and optionally deny-and-warn);
- prune dead entries before the first matcher runs and decrement
  `turns_remaining` after the match round (§4);
- defer mid-dispatch mutations to the next decay tick (§4.1);
- expose the post-decay `session.intent_context` to every matcher in
  `session.pipeline` (OVOS-PIPELINE-1 §5);

**An intent engine that consumes OVOS-INTENT-3 registrations**
**MUST**:

- honour the positive gating contract of §6 — never report a
  match whose intent declares an unsatisfied `requires_context`,
  resolved per §3.1;
- honour the negative gating contract of §6.1 — never report a
  match whose intent declares an `excludes_context` key that is
  live in the session, resolved per §3.1;
- apply the §7 context-supplied capture rule when a
  `requires_context` key also names a slot or vocabulary of the
  intent's definition;
- read context from the post-decay snapshot the orchestrator
  presents on each `match` call (OVOS-PIPELINE-1 §4).

Such an engine **MAY**:

- additionally consume non-null context values as matching hints
  beyond the §7 fill rule (§6);
- surface used context entries in `Match.captures` (§7) in cases
  not covered by the §7 normative rule.

An intent engine that consumes no OVOS-INTENT-3 registrations
(a language-model-backed persona, a chatbot, and so on per
OVOS-PIPELINE-1 §1) has no registered intents to gate and is
unaffected by this specification.

**A skill** that uses intent context **MUST**:

- treat `session.intent_context` as read-only outside the §5 bus events;
  the events of §5 are the skill's only mutation API
  (engines have their own pathway per §5.3);
- choose the scope explicitly (§3): `private` for skill-internal
  state, `shared` for facts other skills may key off;
- not assume any particular engine consumes its values as
  matching hints beyond the §7 fill rule — engines may treat
  context as gates only.

---

## 9. Non-goals — trust and replay

Intent context is **trust-tied to the session that carries it**. It
is not independently authenticated. Two consequences a deployer
**MUST** be aware of:

- **Key-prefix attribution is per-mutation, not retroactive.**
  §5.2 requires the orchestrator to compute the stored key from
  `Message.context["skill_id"]` on every `ovos.context.set` /
  `.unset` it processes. The prefix on a private key proves which
  skill set the entry **in this orchestrator instance, at this
  moment**. It does **not** authenticate entries already present
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
- *Session Lifecycle Specification* (OVOS-SESSION-2) — session
  lifecycle responsibilities; cited by §4.2 for client-side
  session management.
