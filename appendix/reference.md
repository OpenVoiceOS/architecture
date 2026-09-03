---
[← APPENDIX.md](../APPENDIX.md) · Non-normative

## 6. Implementer reference

Material an implementer reaches for repeatedly: cross-spec
tables that don't fit cleanly in any single normative spec.

### 6.1 Topic-name conventions across the family

The naming conventions of OVOS-MSG-1 v2 §2.1.1 — dot-separated
hierarchy, stable root, verb-tense pattern for the trailing
segment, request/terminal pairs sharing a root verb,
`.response` suffix — apply across the family. One rule sits
above the rest: identifiers appear in a topic **only** in the
`<skill_id>:<intent_name>` dispatch shape, and every dotted
topic is a static string. A consumer classifies any topic by a
single test — a `:` anywhere means dispatch, no `:` means an
ordinary dotted topic — and the complete topic surface of a
deployment is enumerable from the specifications alone.
The four-way collision of the word "intent" in introspection
topics deserves an explicit callout:

- `ovos.intent.list` (INTENT-4 §10) — list of registered
  *intents* (skills declare them; `data` entries name
  `intent_name`).
- `ovos.pipeline.<pipeline_id>.intents.list` (PIPELINE-1
  §10) — list of *intents currently compiled by one plugin's
  matcher* (`data` entries name `intent_name`).
- `ovos.transformer.intent.list` (TRANSFORM-1 §6) — list of
  *intent-transformer plugins* loaded at the intent-transformer
  injection point (`data` entries name `transformer_id`).
  Despite the topic shape, this is **not** an intent-listing
  surface; it follows the per-chain pattern
  `ovos.transformer.<type>.list` where `<type>` happens to
  be `intent` for this chain (alongside `audio`, `utterance`,
  `metadata`, `dialog`, `tts`).

The collision is at the human-reading level only; payload
shapes are distinct and a consumer subscribing to one cannot
accidentally parse responses from another.

### 6.2 Session-field cheat-sheet

Every spec in the family that claims a `session` field does
so via the OVOS-SESSION-1 §2.2 registry mechanism. The full
set spans four specs; this table consolidates them. All
fields follow the canonical SHOULD-omit /
`[]`-equivalent-to-omission wire-weight rule of
OVOS-SESSION-1 §3.4.

| Field | Owner | Role | Empty-array semantics |
|-------|-------|------|------------------------|
| `session_id` | SESSION-1 §3.1 | identity / channel | n/a (string; `"default"` reserved) |
| `lang` | SESSION-1 §3.2.1 | preference (user) | n/a (string) |
| `secondary_langs` | SESSION-1 §3.2.2 | preference (user) | ≡ absent |
| `output_lang` | SESSION-1 §3.2.3 | preference (renderer) | n/a (string) |
| `stt_lang` | SESSION-1 §3.2.4 | signal (per-utterance) | n/a (string) |
| `request_lang` | SESSION-1 §3.2.5 | signal (emitter hint) | n/a (string) |
| `detected_lang` | SESSION-1 §3.2.6 | signal (lang-detect) | n/a (string) |
| `site_id` | SESSION-1 §3.3 | opaque group identifier | n/a (string) |
| `location` | SESSION-1 §3.5 | preference (client-owned) | object; absent ≡ empty |
| `pipeline` | PIPELINE-1 §5.1 | preference (ordering) | ≡ absent |
| `blacklisted_pipelines` | PIPELINE-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_skills` | PIPELINE-1 §5.3 | policy (denylist) | ≡ absent |
| `blacklisted_intents` | PIPELINE-1 §5.4 | policy (denylist) | ≡ absent |
| `audio_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `utterance_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `metadata_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `intent_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `dialog_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `tts_transformers` | TRANSFORM-1 §5.1 | preference (chain) | ≡ absent |
| `blacklisted_audio_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_utterance_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_metadata_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_intent_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_dialog_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `blacklisted_tts_transformers` | TRANSFORM-1 §5.2 | policy (denylist) | ≡ absent |
| `intent_context` | CONTEXT-1 §2 | per-session state | object; absent ≡ empty |

**Role glossary:**

- *Preference* — populated by the session origin to request
  specific behaviour. Orchestrator narrows the request by
  availability and policy.
- *Policy* — populated by deployment / layer-2 substrate to
  enforce constraints. Overrides preference at the
  composition stage (PIPELINE-1 §5.5, TRANSFORM-1 §5.3).
- *Signal* — recorded by a producer or earlier lifecycle
  stage to communicate information about this specific
  utterance.
- *Identity / channel* — names the session itself; not a
  preference or policy knob.

### 6.3 Component-identity stamp-rule cheat-sheet

Each component type self-identifies via a reserved context
key. The keys coexist freely on a single Message when the
derivation chain crosses component boundaries; attribution
consumers apply the eight-level lifecycle-position precedence
of CONTEXT-1 §5.2 to pick a single owner when needed.

| Context key | Owner | Stamps on (origination + modify-in-place) | `.reply` / `.response` | `.forward` |
|-------------|-------|------|----------|--------|
| `skill_id` | INTENT-4 §3.1 | yes | yes (authorial — overwrite) | no (preserve inherited) |
| `pipeline_id` | PIPELINE-1 §3.1 | yes | yes (authorial — overwrite) | no (preserve inherited) |
| six `<type>_transformer_ids` (list-valued) | TRANSFORM-1 §1.3 | yes (append) | yes (append) | no (list rides through) |

The `<type>_transformer_ids` list-valued form preserves the
full per-type chain provenance on the wire (every transformer
of that type that touched the Message, in order of touch).
Single-string `skill_id` / `pipeline_id` reflect that those
component types *originate* Messages rather than chain over
them.

### 6.4 Introspection patterns

Several specs in this set define pull-query / scatter-response
introspection surfaces. The shapes are intentionally similar
but serve different scopes:

| Spec | Topic | Scope | Authoritative responder |
|------|-------|-------|-------------------------|
| INTENT-4 §10 | `ovos.intent.list` / `.describe` | Declared intents observed on the bus | Orchestrator (the manifest) |
| PIPELINE-1 §10 | `ovos.pipeline.<pipeline_id>.intents.list` | Intents currently compiled inside a specific plugin's matcher | The pipeline plugin |
| TRANSFORM-1 §6 | `ovos.transformer.<type>.list` | Loaded transformers per injection point | The orchestrator process implementing that chain |

Three properties hold across these surfaces:

1. **Pull-query is the source of truth.** Producers MAY
   broadcast load-time announcements; consumers MUST NOT
   rely on having received them. The bus is asynchronous
   and gives no delivery guarantee; a consumer that started
   late missed the broadcast.
2. **No completeness signal.** A consumer that wants
   completeness keeps its own roster of expected responders
   and times out non-responders.
3. **Per-process slices under split orchestrators.** When
   the orchestrator is split (PIPELINE-1 §2), each process
   responds from its own slice; consumers aggregate.

All of these surfaces share the `ovos.<domain>.` prefix; verb
segments vary by domain (some nest, some don't). The
uniformity is in the namespace, not in a fixed depth.

### 6.5 Message derivation cheat-sheet — `forward` vs `reply`

Use `forward` when a Message travels in the **same direction**
as the source (handler → client). Use `reply` when a Message
travels **back toward the sender** of the source (responder →
requester). Using the wrong derivation produces messages that
work on a single-node local bus but silently mis-route through
layer-2 transports (see appendix/patterns.md §3.1.2).

| Emission | Derivation | Rule |
|---|---|---|
| Handler `speak`, GUI events, session mutations during dispatch (PIPELINE-1 §7) | `forward` | Same direction as the inbound dispatch |
| Handler-lifecycle trio `.start` / `.complete` / `.error` (PIPELINE-1 §8) | `forward` | Same direction as the inbound dispatch |
| `ovos.stop.pong` (STOP-1 §4.2) | `reply` | Skill answers back to the stop plugin that sent the ping |
| `ovos.converse.pong` (CONVERSE-1 §4.2) | `reply` | Candidate answers back to the converse plugin that polled |
| `ovos.fallback.pong` (FALLBACK-1 §6.1) | `reply` | Skill claims or declines back to the fallback plugin |
| `ovos.common_query.pong` / `ovos.common_query.response` (COMMON-QUERY-1 §6.2, §7.1) | `reply` | Skill answers back to the common-query plugin |
| Pipeline introspection response (PIPELINE-1 §10.2) | `reply` | Plugin answers back to the observer that requested |

Deriving a pong with `reply` also carries `context.utterance_id`
(PIPELINE-1 §9.1.1) back to the poller untouched, which is what makes the
answer correlatable to its round. A pong assembled as a fresh
Message loses it and is discarded.
