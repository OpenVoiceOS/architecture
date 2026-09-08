# Plugin Installation Bus Contract Specification

**Spec ID:** OVOS-INSTALL-1 · **Version:** 1 · **Status:** Draft

This specification defines the bus surface by which an administration
client installs and uninstalls Python packages into a running OVOS
service: two topics, the payload field that addresses one service among
several, and the reply they produce.

A deployment split across containers runs each plugin family in the
service that loads it. A speech-to-text plugin installed into the skills
container is never loaded, so a client needs to name the environment an
install lands in. It names it in the payload, and a request that names
nobody reaches every installer.

Dependencies: OVOS-MSG-1 (envelope and the `reply` derivation).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED**, **MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines: the install and uninstall request topics, the
payload field that addresses one service, the form of a service name, the
request payload, the reply topics and their payloads, the error
vocabulary, the enablement gate, the fan-out a client must expect, and
the pre-spec suffixed topics it replaces.

It does **not** define: how a service loads a plugin once installed,
plugin discovery, the package manager or constraint files an installer
uses, which plugin families belong to which service, package naming, or
any authorization beyond the single enablement flag of §5. Authorization
of who may install is a layer-2 concern (OVOS-MSG-1 §3.4).

---

## 2. Topics and addressing

### 2.1 The two topics

| Topic | Meaning |
|-------|---------|
| `ovos.pip.install` | Install the named packages. |
| `ovos.pip.uninstall` | Uninstall the named packages. |

Both are static dotted strings. A service that installs plugins **MUST**
subscribe to both.

No topic carries the target. OVOS-MSG-1 §2.1.1 puts addressing in the
payload: "Nothing about a Message's target travels in a dotted topic:
addressing beyond the topic is **payload** (`data`)", and it admits only
two enumerable runtime-shaped dotted patterns, a per-component
introspection topic and a per-component scheduled event. A topic naming
the service to install into is neither, so the target is a payload field
(§2.2). The suffixed spelling deployments already use is catalogued in
§8.

### 2.2 `service_name` addresses one installer

`data.service_name` names the single service a request is for.

| `data.service_name` | Effect |
|---------------------|--------|
| absent | Every installer acts and answers. |
| present | Only the installer whose own service name matches acts. |

An installer that receives a request naming a different service **MUST**
ignore it: it installs nothing and **MUST NOT** answer, not even to
decline. A refusal from every other installer on the bus would turn one
addressed request into a burst of replies that a client cannot tell from
the real answer.

An installer **MUST** compare `service_name` exactly. It is an
identifier, not a pattern: no case folding, no prefix matching.

A `service_name` naming a service that is not on the bus is answered by
nobody, exactly as a request to a stopped service is (§6).

### 2.3 Service names

A `service_name` is the service's own process or module name written with
underscores, in the case the service itself uses. It is not lowercased:
`ovos_PHAL` is the name PHAL registers, not `ovos_phal`.

The names in use:

| `service_name` | Service |
|----------------|---------|
| `ovos_audio` | the audio output service |
| `ovos_dinkum_listener` | the audio input service |
| `ovos_gui` | the GUI service |
| `ovos_PHAL` | the PHAL daemon |
| `ovos_PHAL_admin` | the privileged PHAL daemon, a separate process |
| `ovos_core` | the skills service |

A service that is not listed chooses its own name by the same rule. Two
services on one bus **MUST NOT** share a service name, since the name is
the whole of the addressing.

---

## 3. Request

Both topics carry the package list, and optionally the service that the
request is for:

```json
{
  "type": "ovos.pip.install",
  "data": {
    "packages": ["ovos-tts-plugin-piper"],
    "service_name": "ovos_audio"
  }
}
```

`packages` is a list of package requirement strings. An installer
**MUST** treat a request whose `packages` is absent or empty as a failure
and answer with the `no packages to install` error of §4.2 rather than
succeeding silently, and **MUST** apply the §2.2 addressing check first,
so a request for another service produces no reply at all.

`service_name` is optional (§2.2).

---

## 4. Reply

### 4.1 Topics

An installer answers with the `reply` derivation of OVOS-MSG-1 §5.2, on
the **base** topic, never on a suffixed one:

| Request | Reply on success | Reply on failure |
|---------|------------------|------------------|
| `ovos.pip.install` | `ovos.pip.install.complete` | `ovos.pip.install.failed` |
| `ovos.pip.uninstall` | `ovos.pip.uninstall.complete` | `ovos.pip.uninstall.failed` |

The reply topic does not vary with `service_name`. An installer **MUST
NOT** answer on a `service_name`-suffixed reply topic, and a client
**MUST NOT** subscribe to one. A client tells replies apart by the
`service_name` it addressed, not by the topic it hears.

The success reply carries no data. The failure reply carries an `error`
key (§4.2).

### 4.2 Errors

`ovos.pip.install.failed` and `ovos.pip.uninstall.failed` carry an
`error` string from this vocabulary:

| `error` | Meaning | Scope |
|---------|---------|-------|
| `pip disabled in mycroft.conf` | Installation is not enabled (§5). | the answering configuration |
| `no packages to install` | `packages` was absent or empty (§3). | the request |
| `skill url validation failed` | A supplied URL was rejected. | the request |
| `error in pip subprocess` | The package manager failed. | the answering environment |

The scope column is normative for a client waiting on a broadcast. The
first three describe the request or one configuration and are the same
answer whichever installer sends them. `error in pip subprocess`
describes one environment only: another service may still succeed, so a
client **MUST NOT** treat it as the outcome of a broadcast.

---

## 5. Enablement

Installation is gated by the deployment's `skills.installer.allow_pip`
configuration key, and the RECOMMENDED default is disabled. An installer
whose gate is closed **MUST NOT** install or uninstall anything, and
**MUST** answer with the `pip disabled in mycroft.conf` error rather than
staying silent.

A silent refusal is indistinguishable from an absent service, and leaves
a client waiting out its own timeout for an answer that will never come.

---

## 6. Fan-out

A request with no `service_name` is answered once **per installer on the
bus**. A client **MUST NOT** assume it produces a single reply, and
**MUST NOT** treat the first reply as the outcome of the whole request,
since a later installer may still answer.

A request naming a `service_name` is answered exactly once, by that
service, because every other installer ignores it in silence (§2.2).

A `service_name` naming a service that is not on the bus is answered by
nobody. A client **MUST** bound its own wait rather than block forever.

---

## 7. Conformance

### 7.1 A conforming installing service **MUST**

- subscribe to `ovos.pip.install` and `ovos.pip.uninstall` (§2.1);
- act on a request whose `data.service_name` is absent or matches its own
  service name, and ignore any other in silence, answering nothing
  (§2.2);
- answer every request it accepts on the base reply topic, using the
  `reply` derivation (§4.1);
- answer a request with no packages with the `no packages to install`
  error rather than reporting success (§3);
- answer with `pip disabled in mycroft.conf`, and install nothing, when
  the enablement gate is closed (§5);
- use an `error` value from the §4.2 vocabulary.

### 7.2 A conforming installing service **SHOULD**

- accept the pre-spec suffixed topics for one stable cycle, so a client
  that has not moved to `data.service_name` keeps working (§8);
- unregister its handlers on shutdown, so a stopped service stops
  answering for an environment it no longer owns.

### 7.3 A conforming client **MUST**

- accept more than one reply to a broadcast (§6);
- bound its wait, since a `service_name` may name a service that is not
  running (§6);
- treat `error in pip subprocess` as describing one environment rather
  than the whole broadcast (§4.2).

### 7.4 A conforming client **SHOULD**

- set `data.service_name` whenever it knows which service loads the
  plugin family being installed, and leave it absent otherwise (§2.2);
- emit the canonical topics rather than the pre-spec suffixed forms,
  which are removed after one stable cycle (§8).

---

## 8. Pre-spec forms

Deployments predating this specification address one service with a
suffixed topic rather than a payload field:

| Pre-spec topic | Canonical equivalent |
|----------------|----------------------|
| `ovos.pip.install.<service_name>` | `ovos.pip.install` with `data.service_name` |
| `ovos.pip.uninstall.<service_name>` | `ovos.pip.uninstall` with `data.service_name` |

These are catalogued, not specified. They put the target in the topic,
which OVOS-MSG-1 §2.1.1 reserves for the payload, and they match neither
enumerable dotted pattern that clause allows.

An installer **SHOULD** keep answering them for one stable cycle so a
client that has not moved is not broken, and **SHOULD** log their use as
deprecated. They are removed at the next major version. A client
**MUST NOT** emit them, and a specification **MUST NOT** define a new
topic of this shape.

---

## See also

- [OVOS-MSG-1](msg-1.md) — the envelope, the `reply` derivation, and
  §2.1.1, which puts a Message's target in the payload rather than the
  topic.
