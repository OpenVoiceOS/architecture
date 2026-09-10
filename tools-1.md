# Agent Tools Bus Contract Specification

**Spec ID:** OVOS-TOOLS-1 · **Version:** 1 · **Status:** Draft

This specification defines the bus surface by which a component finds
and calls the tools a persona can use: the topics a toolbox host
answers, the topics a tool service answers on behalf of every toolbox
it loads, the payload field that addresses one toolbox among several,
the tool definition shape, and the replies.

A toolbox is a plugin that groups tools, each with a typed argument
schema and a typed output schema. A persona offers those tools to a
model; the model asks for one by name; the host runs it and returns
the output. Over the bus, any component can do the same.

Dependencies: OVOS-MSG-1 (envelope, `reply` and `response`
derivations).

The key words **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**,
**RECOMMENDED**, **MAY** are used as in RFC 2119.

---

## 1. Scope

This specification defines: the tool definition shape, the toolbox
discovery and call topics, the payload field that addresses one
toolbox, the tool service topics that aggregate every loaded toolbox,
the reply payloads and error form, and the pre-spec per-toolbox topic
it replaces.

It does **not** define: how a toolbox implements a tool, the Python
plugin contract a toolbox is written against, how a persona chooses
tools for a model, the HTTP renderings of the same tools (UTCP, MCP,
OpenAI function calling), or any authorization of who may call a
tool. Authorization is a layer-2 concern (OVOS-MSG-1 §3.4).

---

## 2. Governing rule

Two clauses of OVOS-MSG-1 fix the shape of every topic below.

§2.1.1: "Nothing about a Message's target travels in a dotted topic:
addressing beyond the topic is **payload** (`data`), and the routing
pair (§3.2–§3.3) is owned by the `reply` swap (§5.2) alone."

§5.3: "Equivalent to `reply(T + ".response", D')`. A `response` is a
`reply` whose topic is the source topic suffixed with `.response`.
Topics defined in other specifications **MAY** rely on the
`.response` suffix convention to mark a Message as the answer to a
prior one."

Every topic in this specification is therefore a static dotted string,
the toolbox a request is for travels in `data.toolbox_id`, and every
answer is the `response` derivation of the request.

---

## 3. Tool definition

A tool is described by one object:

| Field | Type | Meaning |
|-------|------|---------|
| `name` | string | Tool name, unique inside its toolbox. |
| `description` | string | What the tool does, written for a model. |
| `argument_schema` | object | JSON Schema of the call arguments. |
| `output_schema` | object | JSON Schema of the result. |
| `toolbox_id` | string | The toolbox that owns the tool. |

A toolbox **MUST** give every tool a `name` that is unique within the
toolbox. Two toolboxes **MAY** own tools of the same name; the pair
`(toolbox_id, name)` is the identity of a tool on the bus.

A `toolbox_id` is an identifier the toolbox chooses, stable across
restarts, with no `:` and no `.`. Two toolboxes on one bus **MUST
NOT** share a `toolbox_id`.

---

## 4. Toolbox topics

A toolbox host is a component that loads one or more toolboxes and
binds each to the bus. It **MUST** subscribe to both topics for each
toolbox it hosts.

### 4.1 `ovos.persona.tools.discover`

A broadcast. Every toolbox host answers for every toolbox it hosts,
one `response` per toolbox:

```json
{
  "type": "ovos.persona.tools.discover.response",
  "data": {
    "toolbox_id": "wordnet",
    "tools": [ { "name": "define_word", "description": "...",
                 "argument_schema": { }, "output_schema": { },
                 "toolbox_id": "wordnet" } ]
  }
}
```

The request carries no payload fields this specification reads. A
host **MUST** refresh its tool list before answering, so the answer
reflects tools discovered after start.

### 4.2 `ovos.persona.tools.call`

A call to one tool in one toolbox:

```json
{
  "type": "ovos.persona.tools.call",
  "data": {
    "toolbox_id": "wordnet",
    "name": "define_word",
    "kwargs": { "word": "serendipity", "lang": "en" }
  }
}
```

`data.toolbox_id` and `data.name` are required. `data.kwargs` is an
object matching the tool's `argument_schema`; absent means `{}`.

A host **MUST** compare `toolbox_id` exactly and **MUST** ignore a
call naming a toolbox it does not host: it runs nothing and **MUST
NOT** answer, not even to decline. A refusal from every other host
would turn one addressed call into a burst of replies the caller
cannot tell from the real answer. A call naming no toolbox, or a
toolbox nobody hosts, is answered by nobody.

The host that owns the toolbox answers with one `response`:

```json
{
  "type": "ovos.persona.tools.call.response",
  "data": {
    "toolbox_id": "wordnet",
    "result": { "definitions": [ ["noun", "good luck in making unexpected and fortunate discoveries"] ], "examples": [] }
  }
}
```

`result` is the tool output validated against `output_schema`. On
failure the same topic carries `error` instead of `result`:

```json
{ "type": "ovos.persona.tools.call.response",
  "data": { "toolbox_id": "wordnet", "error": "ValueError: unknown tool 'x'" } }
```

A response **MUST** carry exactly one of `result` and `error`. `error`
is a string. When the tool ran and raised, the string starts with the
exception class name, then `: `, then the message. A tool that is not
in the toolbox, arguments that fail `argument_schema`, and an output
that fails `output_schema` are all errors on this topic, each stated in
plain words.

---

## 5. Tool service topics

A tool service is a component that loads every installed toolbox and
answers for all of them under one flat namespace, so a caller need not
know which toolbox owns a tool. The PHAL tools plugin is the shipped
tool service. A tool service **MUST** subscribe to the four topics.

| Topic | Request `data` | `response` `data` |
|-------|----------------|-------------------|
| `ovos.tools.list` | none read | `tools`: list of tool definitions (§3), every toolbox. |
| `ovos.tools.get` | `name` | one tool definition (§3), or `error`. |
| `ovos.tools.invoke` | `name`, `args` (object, absent means `{}`) | `name` and `result`, or `name` and `error`. |
| `ovos.tools.reload` | none read | `tools`: the list after every toolbox refreshed. |

`ovos.tools.get` and `ovos.tools.invoke` answer `error` in the §4.2
form when `name` is absent (`Missing required field: 'name'`) or names
no tool (`Unknown tool: '<name>'`), and with the class-name prefix when
the tool ran and raised. A tool service resolves `name` across
every toolbox it loaded; when two toolboxes own the same name the
service **MUST** answer with an error rather than pick one, since the
flat namespace cannot address the pair of §3.

A tool service is a toolbox host as well (§4) for the toolboxes it
loads, so both surfaces answer on one bus with one loaded set.

---

## 6. Predecessor topic

Shipped toolbox hosts subscribe to a per-toolbox call topic:

| Predecessor topic | Replacement | Notes |
|-------------------|-------------|-------|
| `ovos.persona.tools.<toolbox_id>.call` | `ovos.persona.tools.call` with `data.toolbox_id` | Identifier moves from the topic to the payload. Request and response payloads are unchanged. |

A host **SHOULD** keep answering the predecessor for one stable cycle
after it adopts §4.2, and a caller **SHOULD** send the replacement.
The predecessor is catalogued in appendix/divergences.md §5.7.

---

## 7. Conformance

A toolbox host conforms when: it answers `ovos.persona.tools.discover`
once per hosted toolbox with the §3 shape; it answers
`ovos.persona.tools.call` only for its own `toolbox_id`, with exactly
one of `result` and `error`; and it stays silent on a call for a
toolbox it does not host.

A tool service conforms when it answers the four §5 topics with the
stated payloads, resolves `name` across every loaded toolbox, and
reports a duplicate name as an error.

A conformance suite drives each check over a real bus with one hosted
toolbox that owns one tool, and asserts the payload values, the count
of responses, and the absence of a response where silence is required.
