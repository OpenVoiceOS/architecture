# Boot Readiness and the Utterance Lifecycle

**Spec ID:** OVOS-READY-1 · **Version:** 1 · **Status:** Draft

This document defines a **readiness state** the orchestrator tracks
during boot, and the one requirement it places on the utterance
lifecycle (OVOS-PIPELINE-1 §6/§9.1): an utterance that arrives before
the orchestrator is ready **MUST NOT** enter the pipeline loop at all.

It builds on OVOS-PIPELINE-1 (the utterance lifecycle and pipeline
plugin abstraction this spec gates) and OVOS-MSG-1 (the envelope and
session carrier its new topic travels in).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED** and **MAY** are used as in RFC 2119.

## 1. Motivation

Right after a restart, while plugins are still loading languages,
training models and warming caches, an utterance can legally reach a
pipeline whose `match` (OVOS-PIPELINE-1 §4) call exceeds its bound
(§4.4) purely from contention, not from the utterance being
unmatchable. The call is declined as designed, and the loop falls
through to the next pipeline — which, for a mid-conversation
utterance like a bare "yes", can match an unrelated literal entity
instead of continuing the conversation it belonged to. Observed in
production: OpenVoiceOS/ovos-core#1027.

Raising the bound only delays the same race; it does not remove the
window one utterance can land in. The orchestrator already knows
whether it is still loading. This specification makes that knowledge
observable and gates the lifecycle on it, instead of asking every
pipeline's bound to absorb boot contention it was never sized for.

## 2. The `ovos.ready` namespace

- `ovos.ready.state` (bus message, no payload required) — the
  orchestrator's current readiness, emitted on every transition.
  `context.ready: bool` carries the state so a late subscriber does
  not need a second round-trip to learn it.
- `ovos.ready.query` / `ovos.ready.state` reply — a late subscriber
  asks and gets exactly one reply, the same way `mycroft.ready` (now
  superseded by this state for the orchestrator's own gating; a
  producer **MAY** keep emitting it per VERSIONING.md's one-cycle
  removal window) could not be queried after the fact.

An implementation **MUST** emit `ovos.ready.state` with
`context.ready: false` at orchestrator start, and with
`context.ready: true` exactly once, when it considers itself ready
to match utterances. It **MAY** become unready again later (a
plugin reload, a dependency outage) and **MUST** emit the state
change if it does.

"Ready" is implementation-defined per deployment: at minimum it
**MUST NOT** report ready before every configured pipeline plugin has
completed its own first-load registration. A plugin whose boot work
is inherently per-language (training, prototype loading) **MAY**
report its own partial readiness; the orchestrator's aggregate state
is not ready until none of its plugins are still in that partial
state for the deployment's configured language set.

## 3. Gating the utterance lifecycle

While `ovos.ready.state`'s `context.ready` is `false`, the
orchestrator **MUST NOT** run the pipeline loop (OVOS-PIPELINE-1 §6)
for an incoming `ovos.utterance.handle` (§9.1). It **MUST** instead:

1. speak a short, localized "still starting" response on
   `ovos.utterance.speak` (§9.6), in the utterance's language where a
   translation exists, falling back to the deployment default
   language otherwise;
2. emit `ovos.utterance.handled` as it would for a normal utterance,
   so a caller does not wait on a terminal event that never arrives;
3. do none of the pipeline-loop side effects (§8's handler-lifecycle
   trio, `ovos.intent.matched` / `.unmatched`): none of them ran.

This is a **decline-with-dialog**, not a silent drop and not an
error: a caller gets a prompt response either way, and it says why,
rather than guessing from silence or risking a wrong match.

## 4. Conformance

A conforming orchestrator:

- emits `ovos.ready.state` on every readiness transition (§2);
- answers `ovos.ready.query` with the current state;
- never runs the pipeline loop while unready, substituting the
  decline-with-dialog of §3;
- resumes ordinary pipeline matching the moment it reports ready,
  with no residual gating once `context.ready` is `true`.

---

Proposed alongside a reference implementation in ovos-core (a boot-
state check early in `handle_utterance`, gated on this same signal)
and measured on a production 11-language hub: see the linked PR/issue
discussion.
