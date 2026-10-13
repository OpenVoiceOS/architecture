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
  that reports one is warm and the skills the orchestrator routes to
  are registered, `false` otherwise;
- the `ovos.ready` **namespace** (§2) — the state topic and the query
  a late subscriber uses;
- the **per-plugin warm report** (§2.1) a pipeline plugin emits;
- the **lifecycle gate** (§3) — what the orchestrator does with an
  utterance while not ready.

It does **not** define **what "ready" means for anything other than
the pipeline loop** — a deployment MAY use `ovos.ready` for other
gating, but this specification only constrains utterance handling.

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

The orchestrator's readiness is **at least** the aggregate of two
things: every configured pipeline plugin's own warm state (§2.1), and
the registration of the skills it routes to (§2.2). It is ready once
every plugin that reports a warm state has reported warm and the
skills are registered, not ready otherwise. A plugin that reports no
warm state **MUST** be treated as warm by default, so readiness never
blocks on a signal that does not exist.
Process status (whether the orchestrator's process itself has
started) is a necessary floor — an unstarted orchestrator cannot be
ready — but is not sufficient on its own: a process can be running
and answering health checks while a pipeline plugin it hosts has boot
work left to do.

An implementation **MAY** fold other subsystems it depends on to
serve a request correctly (audio output, a listener, a GUI surface)
into the same aggregate, so that `ovos.ready` does not report ready
while one of those is still unavailable. Pipeline plugins and skill
registration are the only subsystems this specification itself
requires in the aggregate.

### 2.1 Per-plugin warm report

A pipeline plugin reports its own cold-to-warm transition to the
orchestrator once it has finished enough of its own loading to answer
a `match` (OVOS-PIPELINE-1 §4) call correctly, rather than with a
stale or partial result; the specific topic is implementation-defined
between a plugin and its orchestrator. Readiness, once reported, does
not regress for later work the plugin does on its own schedule (a
retrain, a configuration change) while still able to answer correctly
from what it already has: a plugin reports cold once, before its
first warm report, and warm exactly once after that.

### 2.2 Skill registration

Warm pipelines are not enough. A pipeline can answer correctly from
what is registered, and an utterance that belongs to a skill which has
not registered yet is matched to whatever else is registered, or to
nothing. The orchestrator therefore **MUST NOT** report ready until
the skills it will route to have been registered: that is, until the
component that loads skills has finished its first load and signalled
so. The signal is implementation-defined between the orchestrator and
that component.

Only the **first** load counts. Once the orchestrator has seen the
first load complete, a skill that is installed, upgraded or reloaded
later **MUST NOT** make the orchestrator unready: that skill's
registration is the loading component's own business, and refusing
every utterance for one skill would cost far more than the one
utterance it could misroute. An orchestrator that starts after the
loading component has already finished **MUST** be able to learn that
(by asking it), and **MUST NOT** wait for a signal that was sent
before it was listening.

## 3. Gating the utterance lifecycle

While `ovos.ready.state`'s `context.ready` is `false`, the
orchestrator **MUST NOT** run the pipeline loop (OVOS-PIPELINE-1 §6)
for an incoming `ovos.utterance.handle` (§9.1). It **MUST** instead:

1. respond with either a short "still starting" dialog on
   `ovos.utterance.speak` (§9.6) or a non-verbal error sound — the
   choice is implementation-defined. If a dialog is spoken, it
   **MUST** be localized into the session's own language (OVOS-
   SESSION-1); an implementation that has no such localization
   **MUST** play the error sound instead, never a dialog in the
   wrong language;
2. carry a stable error identifier, `ovos.ready.not_ready`, in
   `context.error` on whichever response §1 sent, so a listener on
   either topic can tell this reply apart from an ordinary one and
   knows why it was declined;
3. emit `ovos.utterance.handled` as it would for a normal utterance,
   so a caller does not wait on a terminal event that never arrives;
4. do none of the pipeline-loop side effects (§8's handler-lifecycle
   trio, `ovos.intent.matched` / `.unmatched`): none of them ran.

This is a **decline-with-dialog**, not a silent drop and not an
error: a caller gets a prompt response either way, and it says why,
rather than guessing from silence or risking a wrong match.

## 4. Conformance

A conforming orchestrator:

- emits `ovos.ready.state` on every readiness transition (§2);
- answers `ovos.ready.query` with the current state;
- does not report ready before the first skill load is complete, and
  does not report unready again for a later skill load or reload
  (§2.2);
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
