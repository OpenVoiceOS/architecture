# Spec versioning policy

Version numbers in this repository carry compatibility semantics anchored to
the pre-specification behavior of the OVOS stack:

| Version | Meaning |
| --- | --- |
| **V0** | The de facto, undocumented status quo — the behavior the stack ships before a subsystem is formalized. V0 is never written down as a spec; it is the reference point. |
| **V1** | A formalization of behavior that is **compatible with V0**. A V0 component keeps working against a V1 implementation, even if degraded (missing optional fields, reduced guarantees, legacy namespaces honored). |
| **V2** | Behavior that is **not backwards compatible** with V0. Adopting it requires coordinated migration (e.g. the `legacy_namespace` configuration gate). |

Until launch day, every spec in this repository MUST be classified as V1 or
V2. The classification is part of the spec header. Rules of thumb:

- A spec that documents existing message flows, adds optional fields, or
  introduces parallel namespaces while the legacy ones keep working → **V1**.
- A spec that renames or removes message types, changes payload semantics, or
  requires consumers to change before producers (or vice versa) → **V2**.
- A single spec MAY contain V1 sections and V2 sections only if the V2 parts
  are explicitly gated (configuration flag) and the ungated behavior is V1.

## Version and Revision

Every spec header carries two fields:

- **`Version`** — the spec's V0/V1/V2 **compatibility class** as defined
  above. It answers exactly one question: is this contract backwards
  compatible with the pre-spec status quo? A change to normative content
  that does not alter that answer MUST NOT change `Version`.
- **`Revision`** — a monotonic **within-class edit counter**, starting at
  `1` for the class's first published text. Every pull request that alters
  a spec's normative content MUST increment its `Revision` by one.
  Non-normative edits (typo and formatting fixes, cross-reference
  corrections, examples) MUST NOT increment it.

A refinement — tightening a requirement, adding an optional field,
clarifying a contract while the compatibility class stays intact — bumps
`Revision`, not `Version`. `Version` changes only when the edit changes
the compatibility class itself (e.g. a V1 spec adopts a rename that breaks
V0 consumers → `Version: 2`); a `Version` change resets `Revision` to `1`
for the new class. A spec is cited by class (`OVOS-MSG-1 v1`); `Revision`
disambiguates which text of that class is meant (`v1 rev 3`).

## Applicability

This document is policy-only: it does not itself amend any spec header. A
spec header gains its `Revision` field lazily, in that spec's next
substantive pull request, not through a bulk migration across the repository.

Until a spec's header carries a `Revision` field, `Version` remains the only
lever available for any normative change to that spec — including a
refinement that, under the Version/Revision split above, would otherwise be
`Revision`-only. A spec SHOULD gain `Revision: 1` the first time such a
refinement lands, so that the split applies to it going forward; the pull
request that adds the field records the refinement as the class's next
`Version` bump per the pre-split convention, exactly as CHANGELOG.md already
groups every entry under its spec's `### N` (`Version`) heading, never under
a `Revision` subheading.

Once a spec carries `Revision`, the Version/Revision split above governs:
Version changes only when the edit changes the compatibility class itself;
prose, example, citation, or scope refinements that leave the class intact
bump Revision instead. Until a spec carries `Revision`, its prior `Version`
history MUST be read under the old single-counter policy that predates this
document.

## The 1.0 definition

The compatibility classes define the project roadmap. The stack starts at V0
(the undocumented status quo — beta). Each subsystem is formalized as V1, then
migrated to V2 where the spec demands incompatible change. **OVOS is fully
spec compliant when every subsystem operates on V2 — that state is the
"breakthrough" in "from beta to breakthrough", and it is the 1.0 release
criterion.**
