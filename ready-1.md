# Boot Readiness and the Utterance Lifecycle

**Spec ID:** OVOS-READY-1 · **Version:** 1 · **Status:** Draft

This document defines when the orchestrator is ready, and the one
requirement it places on the utterance lifecycle (OVOS-PIPELINE-1
§6/§9.1): an utterance that arrives before the orchestrator can answer
**MUST NOT** enter the pipeline loop at all.

It builds on OVOS-PIPELINE-1 (the utterance lifecycle and pipeline
plugin abstraction this spec gates).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED** and **MAY** are used as in RFC 2119.

## 1. Scope

This specification defines:

- when the orchestrator is **ready** (§2): every configured pipeline
  is able to match intents, and the skills have been registered;
- the **lifecycle gate** (§3): what the orchestrator does with an
  utterance until it is ready.

## 2. Ready

A pipeline is either **loading** or **loaded**. It is loaded when it
is able to match intents against what is registered with it: its
models, indexes and any other state it needs to answer a `match`
(OVOS-PIPELINE-1 §4) call are in place. It does not stop being loaded
because more intents are registered later; it keeps serving what it
had and gains the new intents when it has absorbed them.

The orchestrator is **ready** once every configured pipeline is
loaded and the skills it routes to have been registered (§2.2).
Readiness is reached once and is not lost: a pipeline gaining intents
or a skill being installed later does not make the orchestrator
unready.

A pipeline that fails to load does not hold readiness back for ever.
It cannot match, so the orchestrator treats it as not configured and
reports the failure the way it reports any other error; nothing more
needs to be specified here, and no further signal is defined.

An implementation **MAY** consider itself not ready until other
subsystems it depends on to serve a request correctly (audio output,
a listener, a GUI surface) are ready too.

### 2.1 Skills

An utterance that belongs to a skill that has not registered yet is
matched to whatever else is registered, or to nothing. The orchestrator
therefore **MUST NOT** be ready before the component that loads skills
has finished its first load. How the orchestrator learns this is
implementation-defined, but it **MUST** be able to learn it when it
starts after the first load already finished, rather than wait for a
notification sent before it was listening.

Only the first load counts. A skill installed, upgraded or reloaded
afterwards **MUST NOT** make the orchestrator unready.

A deployment with no component that loads skills has nothing to wait
for here.

## 3. Gating the utterance lifecycle

Until the orchestrator is ready (§2), it **MUST NOT** run the pipeline
loop (OVOS-PIPELINE-1 §6) for an incoming `ovos.utterance.handle`
(§9.1). It **MUST** instead:

1. respond with either a short "still starting" dialog on
   `ovos.utterance.speak` (§9.6) or a non-verbal error sound; the
   choice is implementation-defined. A spoken dialog **MUST** be
   rendered in the language OVOS-SESSION-1 §3.2.3 gives for
   responses: `output_lang` when the session sets it, otherwise the
   utterance language resolved at intake (§3.2.7). An implementation
   with no localization for that language **MUST** play the error
   sound instead, never a dialog in some other language;
2. carry the error identifier `ovos.ready.not_ready` in
   `context.error` of whichever response step 1 sent, so a listener
   can tell this reply apart from an ordinary one and knows why it was
   declined;
3. emit `ovos.utterance.handled` as it would for a normal utterance,
   so a caller does not wait on a terminal event that never arrives;
4. produce none of the pipeline-loop side effects (§8's
   handler-lifecycle trio, `ovos.intent.matched` / `.unmatched`): none
   of them ran.

A caller gets a prompt response either way, and it says why, rather
than guessing from silence or risking a wrong match.

## 4. Conformance

A conforming orchestrator:

- is not ready while a configured pipeline is loading or before the
  first skill load is complete (§2, §2.1);
- once ready, stays ready (§2);
- never runs the pipeline loop while not ready, substituting the
  decline of §3;
- resumes ordinary matching as soon as it is ready.

A conforming pipeline does not answer a `match` call with partial
state while it is loading.
