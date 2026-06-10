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

Within a class, editorial revisions bump the spec's own revision number in
its header; compatibility class changes (V1 → V2) are a new spec version, not
a revision.
