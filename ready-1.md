# Boot Readiness and the Utterance Lifecycle

**Spec ID:** OVOS-READY-1 · **Version:** 1 · **Status:** Draft

This document defines a **readiness state** the orchestrator tracks,
and the one requirement it places on the utterance lifecycle
(OVOS-PIPELINE-1 §6/§9.1): an utterance that arrives while the
orchestrator is not ready **MUST NOT** enter the pipeline loop at all.

It builds on OVOS-PIPELINE-1 (the utterance lifecycle and pipeline
plugin abstraction this spec gates) and OVOS-MSG-1 (the envelope and
session carrier its new topic travels in).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED** and **MAY** are used as in RFC 2119.

## 1. Scope

This specification defines:

- the **readiness state** (§2) — `true` once every pipeline plugin
  that reports one is warm, `false` otherwise;
- the `ovos.ready` **namespace** (§2) — the state topic and the query
  a late subscriber uses;
- the **per-plugin warm report** (§2.1) a pipeline plugin emits;
- the **lifecycle gate** (§3) — what the orchestrator does with an
  utterance while not ready.

It does **not** define:

- **how a pipeline plugin decides it is warm** — implementation- and
  plugin-specific (a language's model finishing its first compile, a
  cache finishing its first load, or any other plugin-defined
  condition);
- **what "ready" means for anything other than the pipeline loop** —
  a deployment MAY use `ovos.ready` for other gating, but this
  specification only constrains utterance handling.

## 2. The `ovos.ready` namespace

- `ovos.ready.state` (bus message) — the orchestrator's current
  readiness. `context.ready: bool` carries the state. An
  implementation **MUST** emit this message with `context.ready:
  false` at orchestrator start, and **MUST** emit it again, with the
  new value, on every transition between `false` and `true` in
  either direction — not only the first time it becomes ready.
  Readiness is not a one-shot milestone: an orchestrator **MAY**
  become unready again after reporting ready, and **MUST** announce
  that too.
- `ovos.ready.query` — a late subscriber asks and gets exactly one
  `ovos.ready.state` reply, carrying the current state, never a
  cached first answer.

The orchestrator's readiness is **at least** the aggregate of every
configured pipeline plugin's own warm state (§2.1): ready once every
plugin that reports one has reported warm, not ready otherwise. A
plugin that reports no warm state **MUST** be treated as warm by
default, so readiness never blocks on a signal that does not exist.
Process status (whether the orchestrator's process itself has
started) is a necessary floor — an unstarted orchestrator cannot be
ready — but is not sufficient on its own: a process can be running
and answering health checks while a pipeline plugin it hosts has boot
work left to do.

An implementation **MAY** fold other subsystems it depends on to
serve a request correctly (audio output, a listener, a GUI surface)
into the same aggregate, so that `ovos.ready` does not report ready
while one of those is still unavailable. Pipeline plugins are the
only subsystem this specification itself requires in the aggregate.

### 2.1 Per-plugin warm report

A pipeline plugin with boot work that can leave it serving a stale or
partial answer (a language- or model-dependent compile, a cache that
has not finished its first load) reports its own cold-to-warm
transition to the orchestrator; the specific topic is
implementation-defined between a plugin and its orchestrator. A
plugin that has reported warm once **MUST NOT** report cold again
merely because it has later boot work to redo (an incremental
retrain, a configuration change): it already has a genuine prior
state to serve, which is the condition this report exists to
distinguish FROM. A plugin that has never reported warm has no prior
state at all, and **MUST NOT** answer a `match` (OVOS-PIPELINE-1 §4)
call with a stale or partial result in that state — it **MUST**
decline instead (§4.4's bounded-match timeout, or an explicit
no-match).

## 3. Gating the utterance lifecycle

While `ovos.ready.state`'s `context.ready` is `false`, the
orchestrator **MUST NOT** run the pipeline loop (OVOS-PIPELINE-1 §6)
for an incoming `ovos.utterance.handle` (§9.1). It **MUST** instead:

1. speak a short, localized "still starting" response on
   `ovos.utterance.speak` (§9.6), in the utterance's language where a
   translation exists, falling back to the deployment default
   language otherwise; an implementation that has no localized string
   at all for the resolved language **MAY** play a non-verbal error
   sound instead of a dialog in the wrong language -- the choice
   between the two is implementation-defined;
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

A conforming pipeline plugin that has a warm state (§2.1):

- reports cold before it reports warm for the first time;
- reports warm exactly once for its first readiness, and never
  reports cold again afterward;
- does not answer a `match` call with stale or partial state before
  its first warm report.
