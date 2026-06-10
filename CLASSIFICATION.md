# Spec V1 / V2 Classification

This table applies the compatibility semantics from `VERSIONING.md`:
**V1** = formalization compatible with the V0 status quo (a V0 component keeps
working, even if degraded); **V2** = not backwards compatible, requires
coordinated migration; **V1 + gated V2** = ungated behavior is V1, explicitly
gated sections (e.g. behind `legacy_namespace`) are V2.

Classification is evidence-based: for each spec the table records whether its
normative message types, namespaces, and wire shapes exist unmodified in current
`ovos-core` / `ovos-workshop` / `ovos-dinkum-listener` / plugin code, or
whether the spec introduces renames, removals, or protocol changes that require
a coordinated migration.

> **Note on header stamps:** this document carries the classification but does
> **not** edit any spec header. Header stamps follow per-spec once each
> classification is reviewed — editing spec headers here would conflict with the
> 13 in-flight PRs. See issue #60.

---

## On-dev-branch specs (shipped on `dev` today)

| Spec | Classification | One-line justification | Confidence |
|------|---------------|----------------------|------------|
| **OVOS-MSG-1** | **V1** | Formalizes the existing `Message(type, data, context)` wire shape; no message type renames, no field removals. `forward` / `reply` / `response` derivations already exist in `ovos-bus-client`. | certain |
| **OVOS-SESSION-1** | **V1** | Formalizes the existing `session` carrier shape already in `ovos-bus-client`; `session_id`, `site_id`, `pipeline` fields already present. Adds field-registry mechanism as a rule, not a rename. | certain |
| **OVOS-SESSION-2** | **V1** | Client-authority rule and `ovos.session.sync` / `ovos.session.update_default` already exist in `ovos-bus-client`. Spec formalizes the ownership model without changing any topics or payloads. | certain |
| **OVOS-PIPELINE-1** | **V2** | Entry topic is `ovos.utterance.handle`; V0 entry is `recognizer_loop:utterance` (still in `ovos-core` service.py and `ovos-dinkum-listener`). All downstream topics (`ovos.utterance.handled`, `ovos.utterance.cancelled`, `ovos.intent.matched`, `ovos.intent.unmatched`) already exist. The single rename of the entry topic is the breaking change. | certain |
| **OVOS-INTENT-1** | **V1** | Grammar spec for `.intent` / `.dialog` / `.entity` / `.voc` / `.blacklist` files. All constructs (`(a|b)`, `[x]`, `<name>`, `{name}`) already exist in padatious/padacioso/adapt loaders. No wire changes. | certain |
| **OVOS-INTENT-2** | **V1** | File-format and folder-layout spec. Existing five roles are documented as-is. PR #4 adds `.prompt` as a sixth role (additive — skills not using `.prompt` are unaffected). | certain |
| **OVOS-INTENT-3** | **V1** | Conceptual definition of intent, skill-id model, and training-data contract. No new bus messages; no renames. All described behavior pre-exists in `ovos-workshop`. | certain |
| **OVOS-INTENT-4** | **V2** | Introduces unified `ovos.intent.register.keyword` / `ovos.intent.register.template` / `ovos.entity.register` / `ovos.skill.deregister` topics. V0 uses engine-specific topics (`padatious:register_intent`, `adapt:register_intent`, etc.). PR #45 §11 (session-scoped registration) adds no new messages and is V1-compatible within the broader V2 spec. | certain |
| **OVOS-AUDIO-IN-1** | **V2** | Requires the audio-input service to emit `ovos.utterance.handle`; V0 (`ovos-dinkum-listener`) emits `recognizer_loop:utterance`. Breaking only because of the PIPELINE-1 entry-topic rename (same root cause). | certain |
| **OVOS-STOP-1** | **V2** | Introduces `ovos.stop.ping` / `ovos.stop.pong` / `ovos.stop` broadcast. V0 uses `skill.stop.pong`, `<skill_id>.stop.ping`, `mycroft.stop`, `mycroft.stop.handled`. Both the namespace and the shared-vs-addressed pong pattern change. | certain |
| **OVOS-BRIDGE-1** (on-dev) | **needs-review** | On-dev version formalizes emergent patterns from SESSION-2 / MSG-1 / PIPELINE-1 compositions. Classification depends on whether the PR #43 branch version, which depends on TRANSFORM-1 and CONTEXT-1, introduces additional breaking changes. Treat as V2 if any normative dependency is V2. | needs-review |

---

## In-flight PR specs

| Spec (PR) | Classification | One-line justification | Confidence |
|-----------|---------------|----------------------|------------|
| **OVOS-BRIDGE-1** (PR #43 branch) | **V2** | PR branch adds normative composition of PIPELINE-1 (V2), CONTEXT-1 (V2), and TRANSFORM-1 as emergent patterns at the bridge boundary. A V0 bridge does not understand the `ovos.utterance.handle` entry topic or the new `ovos.fallback.*` / `ovos.stop.*` namespaces that flow across it. | needs-review |
| **OVOS-COMMON-QUERY-1** (PR #40) | **V2** | Full bus surface change: V0 uses `common_query.question` / `question:query.response`; spec introduces `ovos.common_query.ping` / `ovos.common_query.pong` / `<skill_id>:common_query`. All V0 common-query skills would need updating. | certain |
| **OVOS-FALLBACK-1** (PR #39) | **V2** | Renames from `ovos.skills.fallback.register` / `ovos.skills.fallback.ping` (shared pong) to `ovos.fallback.register` + per-skill `<skill_id>.fallback.ping` / `<skill_id>.fallback.pong`. Every V0 fallback skill must update its registration and ping/pong topics. | certain |
| **OVOS-AUDIO-1** (PR #38) | **V2** | V0 audio service subscribes to `speak` / `mycroft.audio.queue` / `mycroft.audio.play_sound`; spec introduces `ovos.utterance.speak` (the PIPELINE-1 §9.6 topic) as the primary ingress plus `ovos.audio.queue` / `ovos.audio.play_sound` (renamed from `mycroft.*`). The `speak` → `ovos.utterance.speak` rename is the breaking change. | certain |
| **OVOS-PERSONA-1** (PR #37) | **V2** | V0 persona plugin uses `persona:query`, `persona:summon`, `persona:list`, `persona:release`; spec introduces `ovos.persona.query`, `ovos.persona.list`, `ovos.persona.register` / `ovos.persona.deregister` and uses PIPELINE-1 §7 dispatch (`<pipeline_id>:<intent_name>`) instead of the old addressed topics. | certain |
| **OVOS-CONVERSE-1** (PR #25) | **V2** | V0 converse service uses a **shared** reply topic `skill.converse.pong`; spec uses per-skill **addressed** `<skill_id>.converse.pong`. The change in pong routing from broadcast to addressed is the breaking change. The response-mode replacement of `skill.converse.get_response.enable` / `.disable` with `session.response_mode` is also incompatible. | certain |
| **OVOS-TRANSFORM-1** (PR #20) | **V1** | All six transformer chains (`audio`, `utterance`, `metadata`, `intent`, `dialog`, `tts`) already exist in `ovos-dinkum-listener` / `ovos-core` / `ovos-audio`. The spec formalizes the injection points and chain contract without changing any bus topics or plugin interfaces. | certain |
| **OVOS-CONTEXT-1** (PR #18) | **V2** | V0 uses `add_context` / `remove_context` / `clear_context` bus topics that `ovos-core` subscribes to. Spec replaces these with `session.intent_context` carried in the session object and mutated exclusively via `Match.updated_session`, in-transformer mutation, or `ovos.session.sync`. The three V0 mutation topics are not referenced by the spec. | certain |
| **OVOS-INTENT-2 v3** (PR #4) | **V1** | Adds `.prompt` role (additive). No existing roles changed. Backward-compatible with any V0 skill loader that ignores unknown extensions. | certain |
| **OVOS-INTENT-4 §11** (PR #45) | **V1 section within V2 spec** | Session-scoped registration reuses the existing message shapes; session context already rides on every bus message. The addition is purely additive. The parent spec (OVOS-INTENT-4) is V2 due to the registration topic renames. | certain |
| **OVOS-USER-ID-1** (PR #54) | **V1** | Entirely new spec; adds session fields for identity/auth with no V0 equivalent. Additive: V0 components ignore unknown session fields per SESSION-1 §2.4. No bus topic renames or removals. | certain |

---

## Needs-review list and open questions

1. **OVOS-BRIDGE-1 (on-dev vs PR #43 branch):** The on-dev version is a thinner
   spec (emergent-patterns framing); the PR #43 branch is a significant rewrite
   that adds normative composition of TRANSFORM-1, CONTEXT-1, and PIPELINE-1 at
   the bridge boundary. Question: is the on-dev BRIDGE-1 to be superseded by
   PR #43 before classification stamps are applied? If yes, classify only the
   PR #43 version (V2). If both versions need a stamp, on-dev needs a separate
   determination.

2. **OVOS-PIPELINE-1 legacy entry topic:** The spec does not include a
   `legacy_namespace`-gated dual-emission section for `recognizer_loop:utterance`.
   If such a section is added (parallel emission with a configuration gate), this
   becomes **V1 + gated V2** rather than pure V2. Confirm intent before stamping.

3. **OVOS-AUDIO-1 legacy `speak` topic:** Same question as PIPELINE-1. If
   `ovos-audio` is updated to subscribe to both `speak` (legacy) and
   `ovos.utterance.speak` (spec) gated by `legacy_namespace`, this becomes
   **V1 + gated V2**. Needs decision.

4. **OVOS-CONVERSE-1 response-mode migration:** The `skill.converse.get_response.enable`
   / `.disable` pair in V0 controls get-response mode via explicit bus events.
   The spec replaces this with `session.response_mode`. A migration path
   (dual-mode shim) could make this **V1 + gated V2**; without it, it is pure V2.
   Confirm whether a shim is planned.

5. **OVOS-FALLBACK-1 `ovos.skills.fallback.*` vs `ovos.fallback.*` gap:** The
   namespace change is confirmed breaking, but the dual-emit pattern from
   `workshop#415` / `dinkum#219` mentioned in VERSIONING.md could apply here to
   produce **V1 + gated V2**. Check whether those PRs cover fallback registration
   topics, or only the skill-side ping/pong.

6. **OVOS-STOP-1 `mycroft.stop` compat:** V0 code emits both `<skill_id>.stop`
   and `mycroft.stop` on global stop. If the spec's `ovos.stop` broadcast is
   emitted in parallel with `mycroft.stop` under a configuration gate, this
   becomes **V1 + gated V2**. Needs decision on whether legacy broadcast compat
   is planned.
