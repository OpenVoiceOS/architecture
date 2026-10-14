# Boot Readiness and the Utterance Lifecycle

**Spec ID:** OVOS-READY-1 · **Version:** 1 · **Status:** Draft

This document defines when the orchestrator is ready, and the one
requirement it places on the utterance lifecycle (OVOS-PIPELINE-1
§6/§9.1): an utterance that arrives before the orchestrator can answer
**MUST NOT** enter the pipeline loop at all.

It builds on OVOS-PIPELINE-1 (the utterance lifecycle and pipeline
plugin abstraction this spec gates) and OVOS-MSG-1 (the envelope and
session carrier its new topic travels in).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED** and **MAY** are used as in RFC 2119.

## 1. Scope

This specification defines:

- what **ready** means for the orchestrator (§2): its process status
  reaches `READY` once every configured pipeline plugin is loaded;
- how a pipeline plugin that loads in the background says so (§2.1);
- the skills condition on answering (§2.2);
- the **lifecycle gate** (§3), what the orchestrator does with an
  utterance until it can answer.

## 2. Ready

The orchestrator's process status (`ProcessState.READY`) **MUST NOT** reach `READY` until every
configured pipeline plugin is loaded. A pipeline is either loading or
loaded; there is no third state. The status is the readiness signal:
this specification adds no topic of its own for it.

Work that depends on skills, such as compiling their registrations,
is not part of loading: skills start only after the orchestrator is
`READY`. It is covered by §2.2.

### 2.1 Pipeline loading

A pipeline plugin that finishes loading after its constructor returns
(a model, an index, a compile) announces `ovos.pipeline.loading`
(`data.pipeline_id`) when that work begins and `ovos.pipeline.loaded`
(`data.pipeline_id`) when it ends. The orchestrator **MUST** hold
`READY` back while a pipeline announced before the plugin load step
ended is still loading. A plugin that loads synchronously announces
nothing and counts as loaded. A plugin **MUST** announce `loaded`
when its work ends for any reason, failure included: a failed load is
not a load in progress.

### 2.2 Skills

An utterance that belongs to a skill which has not registered yet, or
whose registration a pipeline has not absorbed yet, is matched to
whatever else is registered. The orchestrator therefore **MUST NOT**
answer utterances until the component that loads skills has finished
its first load (signalled by the skill component, for example
`mycroft.skills.initialized`) and no pipeline is loading.

This holds once, at first start. A skill installed, upgraded or
reloaded later, or a pipeline that retrains for it, **MUST NOT** stop
the orchestrator answering. An orchestrator that starts after the
skill component finished **MUST** ask it (`mycroft.skills.is_ready`)
rather than wait for a signal sent before it was listening. A
deployment with no skill component MAY turn this condition off.

## 3. Gating the utterance lifecycle

Until the orchestrator is `READY` and §2.2 is satisfied, it **MUST NOT** run the pipeline loop (OVOS-PIPELINE-1 §6)
for an incoming `ovos.utterance.handle` (§9.1). It **MUST** instead:

1. respond with either a short "still starting" dialog on
   `ovos.utterance.speak` (§9.6) or a non-verbal error sound — the
   choice is implementation-defined. If a dialog is spoken, it
   **MUST** be rendered in the language OVOS-SESSION-1 §3.2.3 gives
   for responses: `output_lang` when the session sets it, otherwise
   the utterance language resolved at intake (§3.2.7). When there is
   no such localization for `output_lang`, it **MAY** fall back to
   the input-side language as §3.2.3 allows. An implementation that
   has no localization in either **MUST** play the error sound
   instead, never a dialog in some other language;
2. carry a stable error identifier, `ovos.ready.not_ready`, in
   `context.error` on whichever response step 1 sent, so a listener on
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

- does not report `READY` while a pipeline plugin it loads is loading
  (§2, §2.1);
- does not answer before the first skill load is complete and no
  pipeline is loading, and does not stop answering for a later skill
  load or retrain (§2.2);
- never runs the pipeline loop before then, substituting the
  decline-with-dialog of §3;
- resumes ordinary pipeline matching the moment it can answer.

A conforming pipeline plugin that loads in the background announces
`ovos.pipeline.loading` and `ovos.pipeline.loaded` (§2.1), and does
not answer a `match` call with partial state while loading.
