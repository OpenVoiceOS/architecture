# Changelog

Each entry records a change to a specification in this repository. Each
specification carries a `Version` field equal to its V0/V1/V2 compatibility
class (VERSIONING.md): `1` for a formalization compatible with the pre-spec
status quo, `2` once it is not backwards compatible. Entries are grouped under
the spec's current class. Every pull request that alters normative content —
a MUST, SHOULD or MAY, or a wire surface — adds an entry here; editorial
changes do not. No spec carries a revision counter: these entries are the
revision record.

The Policy section below records changes to the rules in `VERSIONING.md`.
Those rules are not a specification and carry no compatibility class, but
they hold shipped code to a duty, so a reader finds them here with the
specifications.

## Policy — VERSIONING.md

- Removed mechanisms (new) — states how long an implementation may keep
  emitting a mechanism that a specification removes. A removal takes away a
  class of behaviour and names no successor, which is different from a
  replacement, and no rule covered it before: the "kept one stable cycle"
  wording applies to replaced topics only. An implementation **MAY** emit a
  removed mechanism for one stable release cycle after it adopts the
  specification that removes the mechanism. It **MUST** log a deprecation
  that names the release which removes the mechanism, and it **MUST NOT**
  begin to read a mechanism it did not already read.
- Removed mechanisms — a record in `appendix/divergences.md` may state a
  narrower window, or keep one part of the mechanism. A record that states
  no window takes the one cycle.

## OVOS-SCHEDULER-1 — Scheduled Events

### 1

- §6.3 now separates an owner that is not running from an owner that is
  not installed, and §9.A.10 records the rule. The first case is the one
  §6.3 always protected: a stopped owner starts again and must find its
  schedules. The second case never reconciles, because nothing starts.
  The scheduler still restores and fires both (§9.A.4) and now MUST NOT
  remove either on its own initiative; it MAY write one WARN line for
  each such schedule in each run. Only a component that holds the
  install inventory, such as a skill loader, may cancel the schedules of
  an uninstalled owner, through the `*` grant of §6.2 that already
  exists. No topic and no field changes, so no implementation changes
  its wire behaviour. Two pieces of new work. The WARN line is new in
  the scheduler in ovos-bus-client. The cancel is new work in the skill
  loader in ovos-core, which holds the inventory as `find_skill_plugins()`
  but sends no cancel today; a `skill.test` schedule whose skill was
  removed fired every two seconds on every run of one deployment from
  2026-09-02.

## OVOS-TOOLS-1 — Agent Tools Bus Contract

### 1

- New. Specifies the agent tools bus surface: the toolbox topics
  `ovos.persona.tools.discover` and `ovos.persona.tools.call` with
  `data.toolbox_id` addressing (§4), the tool service topics
  `ovos.tools.list`, `ovos.tools.get`, `ovos.tools.invoke` and
  `ovos.tools.reload` (§5), and the tool definition shape (§3).
  ovos-plugin-manager's `ToolBox` speaks the discover topic and a
  per-toolbox call topic; ovos-PHAL-plugin-tools speaks the four tool
  service topics. Two things are new work. The static call topic with
  the toolbox in the payload, per OVOS-MSG-1 §2.1.1, lands in
  `ToolBox.bind`, which keeps the per-toolbox topic one stable cycle
  (§6). The §5 rule that a tool service answers an error when two
  toolboxes own one tool name lands in ovos-PHAL-plugin-tools, whose
  registry maps a name to the last toolbox loaded and silently picks it.

## OVOS-INSTALL-1 — Plugin Installation Bus Contract

### 1

- New. Specifies the plugin-installation bus surface: the
  `ovos.pip.install` and `ovos.pip.uninstall` topics, the optional
  `data.service_name` that addresses one service, and the service-name
  form (§2). ovos-audio, ovos-dinkum-listener, ovos-gui and PHAL already
  speak the two topics. None speaks `data.service_name`: every shipped
  installer acts on every request it receives, so the §2.2 addressing
  check is new work in `ovos_utils.skill_installer.ServiceInstaller` and
  in ovos-core's `SkillsStore`.
- §2.1 — the target travels in the payload, not the topic, per OVOS-MSG-1
  §2.1.1.
- §2.2 — an absent `service_name` reaches every installer; a present one
  is acted on by that service alone, and every other installer ignores
  the request in silence rather than declining.
- §4.1 — the reply topic does not vary with `service_name`.
- §4.2 — the `error` vocabulary, and which values describe the request or
  one configuration rather than one environment.
- §5 — the enablement gate, and the duty to answer rather than refuse
  silently.
- §6 — an unaddressed request is answered once per installer, an
  addressed one exactly once, or not at all when that service is absent.
- §8 — the pre-spec `ovos.pip.install.<service_name>` topics are
  catalogued, kept for one stable cycle, and removed at the next major.

## OVOS-AUDIO-1 — Audio Output Service

### 2

- §4.0 (new) — one audio-format rule for all audio payloads: the
  container is the single source of truth (self-describing WAV / MP3 /
  OGG / FLAC); `mime` and `sample_rate` are OPTIONAL advisory fields on
  `ovos.audio.queue`, `ovos.audio.play_sound`, and `ovos.audio.speech`,
  and both become REQUIRED only for headerless raw streams (raw PCM).
- §5.3 — `ovos.audio.is_speaking` is answered via the `response`
  derivation (OVOS-MSG-1 §5.3), on `ovos.audio.is_speaking.response`.
- References OVOS-TRANSFORM-1 by its title, *Transformer Plugins
  Specification*.
- The audio output service: the rendering pipeline (dialog-transformer
  chain, TTS synthesis, TTS-transformer chain, playback queue), the
  sequential playback queue shared by speech (`ovos.utterance.speak`) and
  sound effects (`ovos.audio.queue` / `ovos.audio.play_sound`), the
  remote-client rendering mode (`ovos.utterance.speak.b64` →
  `ovos.audio.speech`), output lifecycle signals
  (`ovos.audio.output.started` / `.ended`), the speaking-status query
  (`ovos.audio.is_speaking`), stop integration (`ovos.audio.stop`,
  `ovos.stop`), and the `listen`-triggered `ovos.mic.listen` follow-up.
## OVOS-PERSONA-1 — Persona Pipeline Plugin

### 2

- §7.1 — embedded persona commands (summon, release, one-off query,
  list, check) are ordinary intents: expressed as locale intent/entity
  resources and matched with the standard intent machinery
  (OVOS-INTENT-2), no bespoke matching layer.
- §8.5 — the out-of-band query always answers: on an unsupported
  `persona_id` the plugin replies on `ovos.persona.answer` with the
  `error` field set and `response` omitted, never silently dropping
  the request; `response` and `error` are mutually exclusive.
- §6 — dismissal mechanics per session class: default sessions need an
  explicit clear (empty-string `persona_id`, a release intent, or a
  committed `Match.updated_session` without the field); named-session
  clients dismiss by removing the field from the state they carry.
- §3 — `persona_id` character-set constraint relaxed to a RECOMMENDED
  ASCII letters/digits/`_`/`-` convention.
- Initial draft. Defines persona as a scoped match+handler layer with
  its own pipeline position, `persona_id` session field, summon and
  dismiss bus messages, match contract (MAY return `None` for pass-
  through), handler contract, no-persona mode, multiple-persona
  coexistence rules, `fallback_pipeline_id` for escalation, dialog-
  transformer compatibility, capability-discovery via
  `ovos.persona.capabilities`, and conformance roles (Persona Plugin,
  Orchestrator, Skill).
## OVOS-CONTEXT-1 — Intent Context

### 2

- §5.3 *Default decay* now lets a client library apply a
  deployer-configured default at write-time, as the orchestrator already
  MAY, and states three things the one-sentence rule left open. Neither
  default re-writes a decay field the writer supplied. A write-time
  default is read from the configuration of the process that writes it,
  which is the orchestrator's only where the two run together, so a
  component on a satellite defaults from its own deployment; and such an
  entry no longer arrives without a decay field, so it is out of reach of
  the orchestrator's own default. A wall-clock default bounds
  accumulation alone, a turn-based one does not, because
  `turns_remaining` is decayed by the orchestrator after each match round
  (§4). This describes what ships: ovos-workshop stamps `context.timeout`
  on its private write path (the cross-skill path is in ovos-workshop#629,
  not yet merged), and the adapt-context view in ovos-bus-client stamps
  the same key, while ovos-core applies no default decay. ovos-core does
  implement §4: at dev `f3d08e9b` the orchestrator decrements
  `turns_remaining` after each match round
  (`ovos_core/intent_services/service.py:843`) and prunes dead entries
  before the next (`:1207`). No topic and no payload shape changes. The
  new work is a docstring in each writer naming whose configuration
  supplies the window.

- Initial draft. Defines `session.intent_context` as a flat map of
  key → entry carried inside the SESSION-1 session carrier. Covers
  context entries (key, value, TTL, owner `skill_id`), private vs.
  shared scopes (`:` discriminator in the key), decay (TTL counts down
  per utterance; entries without TTL are session-persistent), three
  mutation pathways (skill bus events; engine auto-population on match;
  orchestrator sweep on session close), the `requires_context` and
  `excludes_context` intent declaration fields, interaction with the
  match result (context values MAY be injected into `Match.slots`),
  and conformance roles (Orchestrator, Pipeline Plugin, Skill).
  Non-goals: trust enforcement and replay prevention are explicitly
  out of scope.
- §3 — a stored key is read as its shape says, however the writer
  derived it: a key without `:` is a shared entry even when the writer
  meant it to be private, because §2 gives the entry no `scope` field
  and no `origin` field. A component that cannot compute
  `<own_id>:<key>` for an entry it means to keep private **MUST NOT**
  write the entry at all, rather than write it bare. Concatenation
  without the separator is named as no substitute: it is neither
  reversible nor collision-free, so two owner and key pairs can produce
  one bare key, and a bare key the ecosystem agreed on can be produced
  by a pair that was never meant to reach it. A released client library
  folded legacy adapt context entities into `intent_context` under such
  a key, which §3.1 lets satisfy a shared gate and §7 offers as a slot
  candidate.

## OVOS-TRANSFORM-1 — Transformer Plugins

### 2

- §4 — chain order restated cleanly: ascending priority, `1` runs
  first, default `50`; a priority assignment authored for the
  inverse (descending) convention MUST be renumbered, since the two
  orderings are exact inverses.
- §5.3 — the effective chain per injection point is composed once
  per utterance lifecycle from the session as committed at
  utterance start; mid-lifecycle mutations of the preference or
  denylist fields take effect on the next utterance.
- §3.4 — the `Match` slot map is named `slots` (aligned with
  OVOS-PIPELINE-1 §4.1); all `captures` wording renamed.
- Terminal-event topics — `ovos.intent.unmatched` replaces the
  retired failure topic on the no-transcription and cancellation
  paths (OVOS-PIPELINE-1 §9.3).
- OVOS-CONTEXT-1 pathway citations repointed (§5.1 engine-side,
  §5.2 in-place transformer mutation, §5.3 in-place handler
  mutation); the bus-event mutation topics dropped; attribution
  precedence and cancellation-stamp rationale stated locally.
- SESSION-1 registry claims corrected to §2.2; slot-typing
  deferrals restated as timeless out-of-scope statements.

### 1

- Initial draft. Defines six transformer chains at six injection
  points in the OVOS-PIPELINE-1 §6 utterance lifecycle, in lifecycle
  order: audio (raw audio before STT, §3.1), utterance (post-STT text
  normalization before intent matching, §3.2), metadata (session
  enrichment after the utterance text, before the match round, §3.3),
  intent (match-result adjustment after the match round, before
  dispatch, §3.4), dialog (response-text transformation after a skill
  emits `speak()`, before TTS, §3.5), and tts (synthesized-audio
  transformation after TTS, before playback, §3.6). An orchestrator
  MAY implement any subset of the six points; an unimplemented chain
  is a no-op. Chains are ordered; the output of one transformer is the
  input to the next. Per-session ordering and denylists via the
  `<type>_transformers` / `blacklisted_<type>_transformers` session
  fields (§5). Defines session mutation discipline: transformers MAY
  mutate session fields they own (SESSION-1 §2.1) but MUST NOT mutate
  fields owned by other specs; and utterance cancellation (§8) as the
  only sanctioned early short-circuit of the lifecycle, preserving the
  `ovos.utterance.handled` invariant. Conformance roles: Audio,
  Utterance, Metadata, Intent, Dialog, and TTS Transformer, plus
  Orchestrator.

## OVOS-INTENT-1 — Sentence Template Grammar

### 2

**Breaking change.** Version 2 adds the `<name>` inline vocabulary reference
token. A template using `<name>` is not valid version-1 syntax — a version-1
tool does not recognize the token and cannot expand the template.

- §3.6 — the adjacent-slot rule now applies to input-direction templates
  only. Its reason is a match-time one: a matcher must recover two values
  from one span of text, and with no literal word between the slots it
  cannot. The caller fills an output-direction template (`.dialog`) before
  the renderer renders it (§5.1), so nothing recovers a value from the text
  and no boundary is ambiguous. A tool MUST NOT reject a `.dialog` template
  because two of its slots are adjacent. The change only widens the set of
  templates a tool accepts, so the version class does not change.
- §3.6 — the slot-only rule applies to input-direction templates only, for
  the same reason and with the same effect. A bare `{day}` in a `.dialog`
  is rendered, not matched: §6 puts a `.dialog` outside training and §5.1
  fills it from the caller, so "no anchoring text to learn from or match
  against" describes a matcher that never reads the file. The line also
  satisfies §5.5, because `{day}` declares the same slot set as `It is
  {day}`. It is the terse spoken answer, and the natural short reply in
  many languages. Measured: ovos-skill-date-time#367 removed 16 such lines
  from six locales, and #373 then removed the line from 105 files, en-US
  included, because a lint gate reads this bullet against every `.dialog`
  in the fleet. A tool MUST NOT reject an output-direction template because
  it carries no literal word. The change only widens the set of templates a
  tool accepts, so the version class does not change.
- §5.6 — registers a fifth typed-slot type, `language`. Its value is
  `{"code": <lowercase BCP-47 tag>, "name": <string or null>}`: the tag of
  the language that the surface names, and the autonym of that language (its
  name in that language itself, for example `Deutsch`, `Português`, `日本語`).
  For a tag with a script or a region subtag, `name` is the autonym of the
  primary language subtag, and the script and the region stay in `code`.
  The user-locale text stays in `surface`. An orchestrator built against the
  four-type table drops the key (OVOS-TRANSFORM-1 §3.7), and a placeholder
  `{language:name}` degrades to `{name}` (§3.4), so the addition does not change the version class.
- §5.6 — registers two more typed-slot types. `location` has the value
  `{"name": <surface text>, "kind": "city" | "country" | "region" | null}`,
  with no coordinates and no zone. `timezone` has the value
  `{"tz": <IANA zone name>}`, read from a zone name or a zone abbreviation. A
  zone name or abbreviation binds `timezone`; a place binds `location`, and a
  consumer that needs the zone of a place resolves it. A surface gives one
  zone: the session's zone when that zone uses the abbreviation, otherwise
  the zone from the language's zone table. Both degrade like
  `language`, so the version class does not change.
- §3.6 — names a form that was already malformed: a `|` outside a `( … )`
  group. The pipe separates the branches of a group (§3.2) and has no meaning
  elsewhere, and §2 forbids the metacharacters as literal text, so a template
  holding a bare pipe was never valid. The entry is here because the bullet
  sits inside a MUST list; the sentence adds no requirement, and a tool that
  already applied §2 and §3.2 accepts and rejects the same templates as
  before. It closes a real reading gap: every measured loader kept
  `plata|argent` in an `.entity` file as one value, while two other consumers
  read it as two.
- §3.6 — names a second form the list omitted: a matched `< … >` whose
  inner text is not a valid reference name, such as `<Greeting>` or
  `<1abc>`. §3.7 gives `name` the slot-name charset, and §3.6 named an
  unmatched bracket and an undefined or cyclic reference, so a balanced
  token with an invalid name fell between the entries. A tool had to choose
  a category, and ovos-m2v-pipeline#198 chose "unresolved", which is the
  wrong one: an invalid name is decided from the template alone, while
  "undefined" depends on the vocabularies the expander holds. The rejection
  itself is old: §3.7 carries the charset, and §6.2 step 2 already tells an
  engine to verify a template against §3. The reporting duty is new. No
  clause before this one said which category a tool reports, so "a tool MUST
  NOT report it as an undefined reference" is a new requirement on a tool's
  output.
- §3.7 (new) — the `<name>` inline vocabulary reference: a token replaced
  during expansion by a named slot-free vocabulary (a `.voc`, OVOS-INTENT-2).
- §3 — `<name>` added to the grammar token table; §1 lists it under the
  expansion facet.
- §2 — `<` and `>` added to the structural metacharacters that cannot occur as
  literal input.
- §4.1 — expansion gains a first step that resolves `<name>` references
  recursively, before the `[x]` / `()` steps.
- §3.6 — new malformed forms: a reference to an undefined vocabulary, and a
  cyclic reference chain; `<` and `>` join the unbalanced-metacharacter rule.
- §7 — the Expander conformance role MUST resolve inline vocabulary references.
- §3.4, §3 — a named slot MAY be written either `{name}` or `{{name}}`; the two
  forms are exactly equivalent, with a conformant tool folding `{{name}}` to
  `{name}`. The slot-name charset and whitespace rules apply to both. The
  double-brace spelling is a slot, not a brace-escaping form; the grammar still
  provides no escape.
- §5.5 — slot consistency applies to `.dialog` only; a `.intent` file MAY
  declare different slot sets across its templates, and the engine extracts
  the slots of the best-matching template. §6.2 — the engine verifies a
  consistent slot set for `.dialog` and accepts differing slot sets for
  `.intent`.

### 1

- Initial draft.

## OVOS-INTENT-2 — Locale Resource Formats

### 2

- §2.3 (new) makes the resource set of a skill the same in every locale: a
  `(role, base name)` present in one language directory **MUST** be present
  in every other, with `.blacklist` and `.prompt` the two exceptions.
  `.blacklist` is excepted because its content is a property of one
  language, and `.prompt` because a prompt is language-model input, not a
  match or speech surface. The rule is about the file set, not about
  content: it names no reference language and places no language directory
  above another. §2.3 also fixes the **available slot
  set** of an intent, the union of the names its templates declare
  (OVOS-INTENT-1 §5.5), as identical in every locale, because the two
  definitions of one intent share a qualified name and a handler
  (OVOS-INTENT-3 §3). A missing `.voc` an `.intent` of the locale
  references inline stays the OVOS-INTENT-1 §3.6 error, not a parity gap. A
  parity tool MUST report each missing pair by `(role, base name)` and each
  slot-set difference by intent name, and MUST keep them apart from
  malformed files.
- §4.3 adds a corpus rule: every untyped `{slot}` an intent declares
  **SHOULD** have an `.entity` of the same base name in every locale. This
  binds the corpus and not the matcher — OVOS-INTENT-1 §5.4 is unchanged,
  the value set stays an optional refinement, and a tool **MUST NOT** reject
  a skill for a missing `.entity`.
- No loader, topic or field changes. New work: the parity census in
  ovos-m2v-pipeline (skills-qa T-2835) reads these clauses, and an extra
  file or a slot-set mismatch becomes a defect it reports. Before them, no
  clause said which files a locale mirrors, so parity was policy.

**Breaking change.** Version 2 reinterprets brace handling in `.prompt` and
`.dialog` files. A `.prompt` authored against version 1 changes meaning under
version 2: its `{{ … }}` sequences become substitution points, and its
`<!-- … -->` sequences are no longer stripped.

- §4.4 (`.prompt`) — the substitution variable is now the **double-brace**
  `{{name}}` form **only**. A single `{name}`, a lone `{` or `}`, and literal
  JSON/markup pass through unchanged, which is why prompts (full of literal
  single braces) require double braces. The previous author-comment exception
  is **removed**: a `.prompt` has no comment handling and `<!-- … -->` is
  literal text. `{{name}}` substitution is now the *only* special handling a
  `.prompt` receives, dropping the prior fenced-code-block rule (a single-brace
  `{name}` is already literal everywhere).
- §4.2 (`.dialog`) — now recognizes **both** named-slot forms, `{name}` and the
  equivalent `{{name}}`, per OVOS-INTENT-1 §3.4, treating them identically. The
  prior "single-brace only; no `{{ }}` form" restriction is dropped.
- §4.1 (`.intent`) — clarified to match OVOS-INTENT-1 §5.5: lines MAY declare
  different sets of named slots, the intent's slot set is their union, and the
  engine extracts only the slots of the matched template. The prior "every line
  MUST declare the same set of slots; use a separate file" rule is dropped for
  `.intent` (it still holds for `.dialog`, §4.2).
- Consistency review: §2.2 language-fallback suggestion made
  implementation-neutral — a loader MAY fall back to the nearest
  available language as measured by any language-tag distance metric
  that treats close regional variants as usable matches; the reference
  to one particular library and its numeric threshold is removed.

### 1

- The locale folder layout and the plain-text resource file formats
  (`.intent`, `.dialog`, `.entity`, `.voc`, `.blacklist`).
- §1, §4.3 — the `.voc` role is a named set of localized phrasings,
  consumed as a keyword vocabulary and/or referenced inline via `<name>`
  (OVOS-INTENT-1 §3.7), and may itself contain such references.
- §4.4 — the `.prompt` resource role: a whole-file verbatim string
  delivered to a language model. Not a template grammar file: no
  expansion, no line filtering, every character literal. Author-only
  HTML-style comments (`<!-- … -->`) are stripped before delivery; a
  malformed comment (unmatched `<!--`) MUST be reported. Optional
  `{name}` substitution fills only names the caller provides; unfilled
  slots remain literal text, and slots inside fenced code blocks are
  never substituted. Follows the §2.1 locale-override precedence.

## OVOS-INTENT-3 — Intent Definition

### 1

- The intent definition model: keyword and template intents, identity,
  captured slots, suppression, and the handler contract.
- §5.1 — a template intent's `.intent` templates MAY declare different slot
  sets; the engine extracts the slots of the best-matching template
  (OVOS-INTENT-1 §5.5).
- §5.3 — `required_slots`: an optional list of slot names the engine MUST
  extract for a match to be valid; a required slot must be declared by at
  least one template, otherwise the definition is malformed. §5.4 (Example)
  and §5.5 (the `.blacklist` suppression) renumber accordingly.
- §7.1 — a handler MAY rely on `required_slots` being populated; all other
  slots remain optional and must be defended against.

## OVOS-MSG-1 — Bus Message

### 2

- §2.1.1 — the topic convention made the single authoritative rule
  every topic-defining specification inherits: a `:` in a topic marks
  a **dispatch-shaped** topic assembled from identifiers (canonical
  shape `<skill_id>:<intent_name>`), definable only by a formal
  specification; all other topics use the dotted
  `<x>.<y>.<verb>` form and MUST NOT contain `:`. Separator-hygiene
  rules for identifiers used as topic components restated per shape.
- §2 — unknown top-level keys: producers MUST NOT emit them;
  consumers SHOULD treat such a Message as malformed but MAY ignore
  the unknown keys, keeping strictness on the producer side. §7
  consumer conformance aligned.
- §5.2 — `reply` over an array `destination`: selecting the first
  element as the new `source` is RECOMMENDED for deterministic
  convergence; the choice remains implementation-defined and
  consumers still MUST NOT rely on it.
- §3.1 walkthrough marked informative.

### 1

- Initial draft. Formalizes existing OVOS bus behaviour as a single
  specification covering: the on-the-wire JSON envelope (`type` /
  `data` / `context`); the routing keys `source` and `destination`
  that mark the OVOS / handler-code boundary (the attachment point
  layer-2 systems like HiveMind build on top of); the `session`
  carrier with two normative internal fields — `session_id` (where
  `"default"` is reserved for "the Message originates from the device
  itself", already used by `ovos-audio` to decide whether to play TTS
  locally; an absent `session` is treated as equivalent to
  `session_id: "default"`, and `forward`/`reply`/`response` MAY
  materialize the default during derivation) and `lang` (the user's
  preferred language, distinct from per-payload `data.lang`
  describing the message's data language, usually but not necessarily
  matching); the `forward` / `reply` / `response` Message
  derivations; the topic+session correlation model for `.response`
  matching; and UTF-8 JSON serialization rules. No new fields are
  introduced; every key and derivation defined already exists in
  current OVOS code paths (`ovos-bus-client.Message` for the
  envelope, `Message.reply` for source/destination swap,
  `context["session"]` for the session carrier, `ovos-audio` for the
  `session_id == "default"` policy hook). Encryption, transport,
  authentication, authorization, retry, delivery and ordering
  guarantees, session lifecycle, and the internal shape of `session`
  beyond `session_id` and `lang` are explicitly out of scope.

## OVOS-SESSION-2 — Session Lifecycle and State Ownership

### 1

- §3.2, §3.3 — `ovos.utterance.handled` is emitted once per
  lifecycle, not once per utterance: a round that opens a nested
  lifecycle (OVOS-PIPELINE-1 §6.5) produces an inner and an outer
  marker for the same `session_id`, and nothing ranks one above the
  other. A client that adopts at the marker **MUST** apply that
  adoption to every marker for its session and **MUST NOT** discard a
  second one as a duplicate. The rule is order-independent: a client
  **MUST NOT** take the arrival order of two markers as their emission
  order (§3.2), and whichever it holds when the round is over is a
  conformant snapshot. No producer obligation changes, and no wire
  change: the marker was already emitted per lifecycle.
- The state-ownership model (stateless bus, stateless orchestrator for
  named sessions, orchestrator-owned default session), the mutation
  boundaries, convergence without push topics (§2.7), client-side
  merge rules, resumption semantics, and conformance.
- §2.4 — a handler that emits no Message cannot propagate in-place
  session mutations: the orchestrator-emitted handler-lifecycle trio
  does not reflect handler-side changes, so a handler whose mutations
  must appear in terminal events emits at least one Message
  (typically `ovos.utterance.speak`).
## OVOS-SESSION-1 — Session Carrier Wire Shape

### 1

- §3.4, §6 — the per-component override class, as the wire-weight
  rule (§3.4 items 2 and 3) and the producer **SHOULD NOT** (§6)
  enumerate it, now names the six `blacklisted_*_transformers` lists
  the §3 field table already claims for OVOS-TRANSFORM-1 §5.2. The
  class had fifteen members and the enumerations listed nine. Through
  OVOS-PIPELINE-1 §5.5, which defers to this class, the orchestrator
  MUST now re-impose the six lists onto an `updated_session` a plugin
  returns. ovos-core at dev `f3d08e9b` (3.7.0a1) does not yet:
  `_DEPLOYMENT_OWNED_SESSION_FIELDS` holds the nine fields and not
  the six lists. No wire change: an omitted field already resolves to
  the deployment default.
- The `context.session` carrier wire shape: the `session_id` and `lang`
  core fields, the language field family (§3.2), the §2.1 field-registry
  mechanism by which other specifications claim OPTIONAL session fields,
  and the propagation and wire-weight rules.
- §2.1 — registered fields and their owning specifications:
  `converse_handlers` (OVOS-CONVERSE-1 §2.1), `fallback_handlers`
  (OVOS-FALLBACK-1 §4), and `persona_id` (OVOS-PERSONA-1 §3).
- §3.3 — `site_id` is defined by OVOS-BRIDGE-1 §3.3; this section states
  the consumer constraints that apply within the orchestrator pipeline.
- See also — each field's owning specification, including
  `session.active_handlers` (OVOS-PIPELINE-1 §7.1) and
  `session.converse_handlers` (OVOS-CONVERSE-1 §2.1).
- §4.1 — the materialization rule binds a component that derives a
  Message for a session it did not originate. It does not reach the
  session origin populating the session it originates: there is no
  source Message, so there is nothing the origin did not receive, and
  the fields the origin sets are the values it requests. A shipped
  client library read §4.1 as binding the origin and therefore never
  put a satellite's configured `location` on the wire, so the master
  answered every satellite session from its own configuration.
- §3.5, §2.1 — `location` is named as an **override field**, the
  resolution class §2.2 item 3 requires and §3.5 never stated. A
  session origin that has a configured position **SHOULD** set
  `location` explicitly, because omission is a read-side default that
  resolves at the consumer, which across a layer-2 boundary is a
  different box from the origin. The §3.4 wire-weight rule is
  unchanged and continues to apply where the origin can establish that
  the consumer computes the same default.

## OVOS-INTENT-4 — Intent and Entity Registration Bus Contract

### 2

- §8.4 — the cross-skill deregistration (source ≠ target) is carried
  only by a transport that delivers `ovos.skill.deregister` on its own;
  a producer **MUST NOT** emit it through a client that mirrors the
  topic onto `detach_skill`, whose predecessor handlers act on the
  source. Self-deregistration is unaffected. A malformed emission
  (payload without `skill_id`) has no target, so spec handlers remove
  nothing. The mirrored predecessor acts on it as a
  self-deregistration. The mirror
  runs both ways, so emitting `detach_skill` directly does not avoid
  it. Divergence row for `detach_skill` records both shapes.
- §3.2: a message of §5 to §8 whose payload omits a required identity
  field is malformed. A consumer **MUST NOT** index or act on it,
  **MUST NOT** derive the value from context, topic or another field,
  and **MUST** log the rejection at WARN naming the missing field.
- §7.2: an entity registration that omits `lang` is malformed. A
  consuming plugin rejects it and logs at WARN with the name of the
  missing field. It does not substitute its own configured language
  or the session language. §8.3 is unchanged. An omitted `lang` on
  `ovos.entity.deregister` still removes every language.
- §8.6 (new) — `ovos.skill.loaded`, the session-keyed load announcement
  with a registered `capabilities` vocabulary (`fallback`, `common_query`,
  `converse`); withdrawn by `ovos.skill.deregister`. §10.3 (new) —
  `ovos.skills.list` / `.list.response`, loaded skills with capabilities
  and intent counts, optional `session_id` filter. §12 updated.
- §8.5 — enable/disable scope stated: a disable under `"default"` acts on
  the inherited `"default"` registration and is device-wide; a disable
  under a specific `session_id` affects that session's view only.
- Bus contract for declaring intents and entities, the wire companion to
  OVOS-INTENT-3. Registration topics (`ovos.intent.register.keyword` /
  `.template`, `ovos.entity.register`), deregistration / enable / disable,
  and orchestrator-owned manifest introspection (`ovos.intent.list` /
  `.describe`). Keyword registration carries inline `required` / `optional`
  / `one_of` / `excluded` vocabulary descriptors. A single intent MAY be
  registered under both keyword and template methods. Registrations are
  fire-and-forget broadcasts: no `.response` acknowledgement; manifest
  presence is the only success signal. Consuming plugins log
  malformed-payload rejections at WARN with full identifiers and the
  rejecting topic. File paths never cross the bus — locale files are
  expanded inline before emission.
- §6 — `required_slots`: an optional array on the
  `ovos.intent.register.template` payload listing slot names the engine
  MUST extract for a match to be valid (OVOS-INTENT-3 §5.3). §6.2 —
  templates MAY declare different slot sets (OVOS-INTENT-1 §5.5). §6.3 —
  a `required_slots` entry naming a slot no template declares is malformed.
- §11 — session-scoped registration. The registration key is the quintuple
  `(session_id, skill_id, intent_name, lang, method)`, read from
  `context.session.session_id`; `session_id == "default"` is the global
  scope every session inherits (§11.2). Deregistration MAY narrow to one
  `session_id` (§8.4); session teardown removes its session-scoped
  registrations (§11.3). A plugin that does not implement session scoping
  treats every registration as global (§11.4). §11.5 — the orchestrator
  routes a session-scoped intent only to utterances whose session matches
  the registration's `session_id`.
- §8.1 — intent replacement is keyed by the quintuple, entity replacement
  by `(session_id, skill_id, entity_name, lang)`. §12 — the orchestrator
  keys the manifest by the quintuple and serves session-aware
  `ovos.intent.list` queries.
- Consistency and design review: §5.2/§6.1 — an absent list-valued key
  equals an empty list and MUST NOT be treated as malformed (the
  all-four-keys shape-stability requirement is dropped). §5.3/§6.3 —
  unknown payload fields are ignored and preserved, never malformed;
  this is what carries companion-spec fields (e.g. OVOS-CONTEXT-1
  gating declarations). §2 — manifest non-gating scoped to registration
  processing; read-only consultation by other specifications is
  permitted. §8.4/§11.1/§11.3 — session scoping for deregistration
  reads exclusively from `context.session.session_id`; the
  payload-level `session_id` field is removed. §8.5 —
  enable/disable of an unregistered intent is a no-op; disabled-state
  survival across a plugin reload is out of scope (recover via the
  manifest). §10 — cold-start recovery: a skill SHOULD re-emit its
  registration set on observing the deployment's readiness
  announcement; re-emission is idempotent per §8.1. §10.2 — describe
  responses are keyed on the `method` field, with `keyword`-then-
  `template` ordering RECOMMENDED. §12 — orchestrator passivity
  qualified by the §3.2 reserved-`intent_name` exclusion; §3.2 identity
  citation corrected to INTENT-3 §3; language-fallback deferral
  restated as an out-of-scope statement.
- §§5.3, 6.3, 7.2 — partial-malformation tolerance: an individual
  template, vocabulary sample, or entity entry that is not parsable as
  OVOS-INTENT-1 grammar (or expands to zero non-empty samples) is
  skipped with a per-item WARN carrying the §5.3 fields; the consuming
  plugin indexes the remaining valid items and rejects the registration
  only when no valid item remains. Whole-registration rejection is
  reserved for reserved `intent_name`, missing top-level keys, and
  missing/empty `samples`.
- §3.2 — registration acts on the payload `skill_id`. The payload names
  the target of every message of §§5–8; `context.skill_id` names the
  source and is provenance only, which a consumer logs at DEBUG when the
  two differ but never substitutes for the payload and never rejects on.
  A message of §§5–8 is complete without `context.skill_id` and its
  absence is not malformed, so a source that is not a skill — an
  administrative script, a provisioning tool — registers or retracts on a
  skill's behalf. Enable and disable stop being an exemption and become
  the same shape as the rest. The deployment policy that MAY block
  cross-skill messages now covers all of §§5–8, naming
  `ovos.skill.deregister` as the remote-uninstall risk it guards. §12
  producer, plugin and orchestrator obligations follow, and the appendix
  divergence entry is restated to match.

## OVOS-AUDIO-IN-1 — Audio Input Service

### 2

- §6.4 — `ovos.listener.wake`, the controller's request to leave sleep
  mode; device-scoped like `ovos.listener.sleep`, a no-op while awake,
  followed by `ovos.listener.awoken` on the transition. Successor of the
  pre-spec `recognizer_loop:wake_up`.
- §6.5 (new) — `ovos.listener.wakeword`: the wake-word detection
  signal (`wake_word`, optional `lang`), preceding
  `ovos.listener.record.started`; the observable event behind a
  wake-word-derived `session.request_lang`. Push-to-talk /
  `ovos.mic.listen` capture emits no wake-word signal.
- §6.3 — sleep is device-scoped: a sleeping service captures nothing
  for any session; sleep entry is unacknowledged by design, the only
  sleep-related emission being `ovos.listener.awoken`.
- §5.1 — language resolution is a MUST-precedence rule
  (`detected_lang` → `request_lang` → `session.lang` → deployment
  default), so every producer of a language hint can predict the
  transcription language.
- §6 (new) — listening lifecycle signals. The audio input service
  emits `ovos.listener.record.started` / `ovos.listener.record.ended` around
  voice-command capture, accepts `ovos.listener.sleep` to enter sleep mode
  and suspend capture, and emits `ovos.listener.awoken` on the sleep→awake
  transition. These replace the legacy `recognizer_loop:record_begin`
  / `recognizer_loop:record_end` / `recognizer_loop:sleep` /
  `mycroft.awoken` topics. All carry no payload; the session is
  identified by `context.session.session_id`.
- §6.5 — bus surface table for the listener role, including the
  consumer-side `ovos.mic.listen` row (defined in OVOS-AUDIO-1 §4.4).
- See-also — cross-references OVOS-AUDIO-1 §4.4 as the defining spec
  for `ovos.mic.listen`.

## OVOS-OCP-1 — OVOS Common Playback: the Virtual Media Player

### 2

- §3.1 — numeric codes for the `PlayerState` axis; the code, not the
  symbolic name, travels on the wire in state reports.
- §4.2.1 (new) — the `ovos.common_play.play` payload: `media` (media
  entry), `playlist`, `disambiguation`, `repeat`.
- §4.2.2 (new) — seek is absolute: `position` in milliseconds within
  now-playing; relative skips are resolved by the requester from the
  state reports, so concurrent seekers cannot compound offsets.
- §4.2 — `ovos.common_play.search.start` / `.end` bracket the
  discovery step as first-class Messages.
- §4.4 — the shared state-report payload (`state`: numeric axis code)
  and the track-state axis (disambiguation / queued / playing,
  qualified by backend kind).
- §4.5 (new) — the media entry object exchanged by playback requests
  and state consumers; unknown fields are ignored by consumers.
  `length` counts in milliseconds with `-1` = unknown/live — one time
  convention across the media surface (OVOS-GUI-1 §3.4).

### 1

- Initial draft. Formalizes the Virtual Media Player: one logical,
  session-scoped media player that every media command targets, behind
  which any number of playback backends, remote devices, or external OS
  players may serve a track. Defines the player and media state model;
  the bus surface that requests playback and transport control and
  reports state; the distinction between playback requests (start
  something) and control requests (act on what is already playing); and
  the MPRIS bridge by which the virtual player is exported to the host OS
  and by which externally-initiated, standards-compliant players become
  controllable by voice. Media discovery and ranking, URI-to-bytes
  playback, stream-extraction formats, now-playing rendering, and the
  NLU that classifies an utterance as media are out of scope — provider,
  backend, GUI, and pipeline concerns respectively.
## OVOS-GUI-1 — GUI Display Subsystem

### 1

- Initial draft. Formalizes the GUI display subsystem that decouples
  voice applications from rendering technology: applications declare
  *what* to display by naming a template from a closed, curated
  vocabulary of `SYSTEM_*` templates and supplying that template's
  session data; interchangeable render backends (adapters) decide
  *how*. Defines the closed template vocabulary (§3) — state/feedback,
  content primitives, media, domain cards, and interactive companions
  — each with its normative session-data keys; the wire protocol (§4)
  — `gui.value.set` / `gui.page.show`, the reserved `__from` / `__idle`
  data keys, and the per-session namespace lifecycle; addressing (§5)
  — routing by `session_id` alone, with shared/multi-room screens
  expressed by clients sharing a `session_id` and no separate
  site/location dimension; the adapter contract (§6) — entry-point
  discovery, unconditional multi-modal fan-out to every installed
  adapter, exception-safe non-blocking handlers/hooks keyed by
  `session_id`, and capability-degradation guidance; the
  interaction path (§7) for media transport and `confirm` / `select`
  responses carrying the originating `session_id`; and conformance
  (§8) for producers, the GUI service, and adapters. Raw-passthrough
  templates (`SYSTEM_html` / `SYSTEM_url`) are documented as
  discouraged escape hatches. Builds on OVOS-MSG-1 (envelope),
  OVOS-SESSION-1 (the `session_id` routing key and reserved
  `"default"`), and OVOS-SESSION-2 (lifecycle / client authority).

## OVOS-STOP-1 — Stop Pipeline Plugin

### 2

- The stop pipeline plugin: matches utterances expressing the intent to
  interrupt the assistant and either cascades a stop across the
  recency-ordered active handlers or broadcasts a global stop. The
  ping/pong discovery exchange (`ovos.stop.ping` / `.pong`, §4) that
  selects the most recently activated handler able to stop, per-handler
  dispatch on the reserved intent_name `stop` (`<skill_id>:stop`,
  OVOS-PIPELINE-1 §7.3), the global-stop path (`global_stop` →
  `ovos.stop` broadcast), and the session-scoping obligations over
  `session.active_handlers`.

## OVOS-COMMON-QUERY-1 — Common Query Pipeline Plugin

### 2

- Initial draft. Specifies the common query pipeline plugin: a
  scatter-gather contest that answers factual questions by
  broadcasting the utterance, collecting competing answers from
  skills, ranking them, and speaking the best. Reserves the
  `common_query` intent_name (PIPELINE-1 §7.3). The full contest runs
  in the plugin's blocking `match` (a deliberate, documented exception
  to PIPELINE-1 §4.4 latency discipline, since the answer is the claim
  decision): a fast `ovos.common_query.ping`/`pong` poll filters
  skills down to plausible answerers using only cheap local checks,
  then `<skill_id>:common_query` requests full answers (where network
  and DB I/O are expected) collected on `<skill_id>.common_query.response`.
  Filtering and selection (minimum confidence, denylist, fast-win,
  optional reranker) run against the live session; if no answer
  survives, `match` returns `None` so the pipeline reaches fallback.
  A surviving answer is carried in `Match.slots.answer` and spoken by
  the plugin's trivial handler — skills never speak. Defines an
  optional question gate (SHOULD, for latency) and an early-start
  optimisation subscribing to `ovos.utterance.handle` to overlap the
  contest with upstream pipeline stages, caching only raw responses
  keyed by `(session_id, utterance)`. All poll/answer messages carry
  the `utterance` as correlation key and derive via MSG-1 `reply`,
  with the session in `context.session`. Tunable defaults and
  confidence-range guidance are collected in appendices.
- Consistency audit: the full-answer request renamed
  `<skill_id>:common_query` → `<skill_id>.common_query.request` — it is
  a plugin-emitted request, not an orchestrator dispatch, and the colon
  form is reserved for the PIPELINE-1 §7 dispatch shape (OVOS-MSG-1
  §2.1.1); the message family is now uniformly dotted
  (`ovos.common_query.ping`/`.pong`,
  `<skill_id>.common_query.request`/`.response`) with the single
  remaining colon topic the genuine dispatch `<pipeline_id>:common_query`;
  §8 fast-win demoted to a deployment-opt-in rule, off by default —
  self-reported `conf` shares no calibrated scale across skills, so
  first-past-threshold selection is a nondeterministic latency race;
  confidence thresholds, window sizes, and `latency_ms` interpretation
  remain RECOMMENDED tunables (Appendix A); §5 early-start phrased
  normatively (responses MAY already be collected); cross-spec note that
  the poll boolean's field name (`can_answer`) is protocol-local;
  OVOS-SESSION-1 cited by its canonical title.
## OVOS-FALLBACK-1 — Fallback Pipeline Plugin

### 2

- §3.4 — the registry is not served on the bus; loaded fallback skills
  are listed by OVOS-INTENT-4 §10.3.
- The fallback pipeline plugin: the final stage(s) that handle utterances
  no earlier stage claimed by querying registered fallback skills in
  priority order and dispatching to the first willing one. Skill
  registration (`ovos.fallback.register` / `.deregister`) with a priority
  hint, session-scoped per OVOS-INTENT-4 §11. The `session.fallback_handlers`
  preference list (§4), pool construction (§5), the sequential unicast
  ping/pong match contract (`<skill_id>.fallback.ping` / `.pong`, §6),
  dispatch on the reserved intent_name `fallback` (§7, OVOS-PIPELINE-1 §7.3),
  and pipeline positioning with multi-stage priority ranges (§8).
- Consistency audit: §6.1 the per-skill wait MUST be bounded by a ceiling
  (unbounded waits stall the utterance), with the ceiling itself a
  RECOMMENDED default of 0.5 s matching the analogous OVOS-CONVERSE-1 /
  OVOS-STOP-1 polls; an absent or malformed pong (missing or non-boolean
  `can_handle`, mismatched `skill_id`) MUST be treated as
  `can_handle: false`, uniform with the companion polls' silence rules;
  §6.1 broadcast-poll optimisation defined as observably equivalent to
  the sequential cycle (selection stays keyed on pool order, never
  response-arrival order); §3.3 priority tiers remain a recommended
  convention mapped by band membership, not numeric rescale; §3.3
  catch-all ordering stated as SHOULD; §6.3 `utterance` defined as the
  first element of the candidate list (OVOS-PIPELINE-1 §4.1); cross-spec
  note that the poll boolean's field name (`can_handle`) is
  protocol-local; citations corrected (`blacklisted_skills` →
  OVOS-PIPELINE-1 §5.3, session registry → OVOS-SESSION-1 §2.2,
  `ovos.intent.unmatched` → PIPELINE-1 §9.3) and OVOS-SESSION-1 cited by
  its canonical title.
- §3.1, §3.2 — fallback registration and deregistration act on the
  payload `skill_id`, matching OVOS-INTENT-4 §3.2. The payload names the
  target, `context.skill_id` names the source and is provenance a plugin
  logs at DEBUG but never substitutes in, rejects on, or requires. The
  identity check that demanded the two be equal is replaced by a
  deployment policy a plugin MAY enforce against cross-skill
  deregistration, the same treatment INTENT-4 §3.2 gives
  `ovos.skill.deregister`. The change is text-only on the wire: the
  producer emits registrations with no context at all and the consumer
  already reads the payload.
## OVOS-CONVERSE-1 — Active Handlers and Interactive Response

### 2

- The imperative continuous-dialog surface. Two session fields:
  `converse_handlers` (§2.1), a recency-ordered list of active handlers,
  and `response_mode` (§2.2), the single entry holding the next utterance
  exclusively. The converse plugin as a pure matcher (§3) that checks
  `response_mode` first, then polls `converse_handlers` via sequential
  unicast ping/pong (`<skill_id>.converse.ping` / `.pong`, §4). Response
  mode (§5): single-shot delivery via the reserved `response` intent_name.
  Activation lifecycle (§6), stop integration (§7), and the bus surface
  (§8). Dispatch on the reserved intent_names `converse` and `response`
  (OVOS-PIPELINE-1 §7.3) follows ordinary §7 routing.
## OVOS-PIPELINE-1 — Utterance Lifecycle and Pipeline

### 2

- §9.7 (new) — `ovos.policy.denied`, a broadcast diagnostic the
  orchestrator MAY emit when a candidate Match is dropped for a
  `blacklisted_*` reason (§5.2–§5.4); payload `field` and `value`, session
  in `context.session`; informative only.
- §6.2 — the orchestrator treats a `Match` as declined when any slot
  listed in the intent's `required_slots` (OVOS-INTENT-3 §5.3) is absent,
  behind the engine's own enforcement during `match`.
- §7.3 — reserve the intent_names `fallback` (OVOS-FALLBACK-1 §6.3) and
  `common_query` (OVOS-COMMON-QUERY-1 §3).
- §9.6 — the OPTIONAL `listen` field on `ovos.utterance.speak`: when
  `true`, the output stage re-opens the user input channel after the
  response is delivered.
- Consistency and design review: §4/§9.1 — when the entry topic carries
  no authoritative `lang`, the orchestrator MUST resolve the utterance
  language once (OVOS-SESSION-1 §3.2 evidence) and pass the resolved tag
  to every plugin's `match` call; plugins MAY refine but MUST NOT
  re-derive independently (`Match.lang` remains the plugin's
  declaration). §4.4 — RECOMMENDED default match-phase timeout of 10 s;
  an applied bound MUST be at least any stage-internal collection
  ceiling. §6.1 — context decay aligned with OVOS-CONTEXT-1 §4: the
  post-match `turns_remaining` decrement runs after the match round
  whether or not any intent matched, with freshly written entries
  exempt; promotion citations corrected to CONTEXT-1 §5.1. §6.5 —
  orchestrator liveness: the bus loop MUST keep servicing subscriptions
  (including poll replies for an in-flight plugin) while a `match` call
  is in flight. §7.1/§7.3 — `active_handlers` stamping suppression MUST
  key on the Match's reserved `intent_name`, never the producing
  `pipeline_id`. SESSION-1 registry citations corrected to §2.2;
  reservation wording made timeless.

### 1

- The utterance lifecycle, the pipeline-plugin `match` contract, the
  session fields owned by this specification (§5), dispatch (§7), the
  handler-lifecycle trio (§8), and the utterance-layer topics — the
  entry point `ovos.utterance.handle` (§9.1) and the response exit point
  `ovos.utterance.speak` (§9.6).
## OVOS-BRIDGE-1 — Bus Bridge and Opaque Relay

### 2

- §3.2 — the `session_id` NAT, where a bridge performs it, is
  **total**: the bridge MUST map every inbound `session_id` to a
  hub-side identifier, including the reserved `"default"` and the
  omitted and empty forms that resolve to it, unless a layer-2 grant
  authorises the participant to act on the default session. Left
  unmapped, a remote `"default"` reaches the orchestrator's own
  default-session store. NAT itself stays a MAY (§3.2, §3.4.1,
  §4.2.3). No wire change: this formalizes the non-admin path
  HiveMind-core ships since `4.10.9a1`.
- §4.1 — a gate that refuses a client-supplied session field MAY emit
  `ovos.policy.denied` (OVOS-PIPELINE-1 §9.7).
- §4.1 — the **gate invariant** made explicit: a bridge that
  injects policy fields (any `blacklisted_*` array, a restricted
  `pipeline`) MUST re-apply them on every inbound Message from the
  governed participant; connect-time-only policy is not conformant
  under client authority (OVOS-SESSION-2 §2.5). The bridge is a
  gate, not a handshake.
- §4.2.4 — a managed-mode bridge (§3.4.2) fronting concurrent
  independent clients MUST assign each a distinct `session_id`; in
  managed mode session attachment is the bridge's obligation.
  Relaying-mode satellites keep the SHOULD.
- §4.2.5 — the TTS-as-a-service topology relocated next to the
  hub-side-audio topology it complements; the hub-side audio bus
  surface cited to OVOS-AUDIO-1.
- Timelessness — the hardened-minimum topic set cites
  OVOS-PIPELINE-1 §9 without enumerating its topics; repository
  gap-catalogue pointers removed from normative text.
- The bus bridge: a participant that terminates an external channel and
  relays Messages between the internal bus and remote participants. §3 —
  the normative core: inbound identity stamping (`source`), outbound
  routing by `destination` / `session_id` / `site_id`, `site_id`
  assignment, and the relaying vs managing session-preservation modes.
  §4 — emergent patterns over MSG-1 + SESSION-1/2 + PIPELINE-1 +
  TRANSFORM-1 + CONTEXT-1 + INTENT-4 at a bus boundary: policy injection,
  multi-deployment topologies, and satellite skill registration. §5 ordering guidance; §6 conformance.
- §3.3 — `site_id` assignment is owned here; OVOS-SESSION-1 §3.3 carries
  the registry pointer and the orchestrator-pipeline consumer constraints.
