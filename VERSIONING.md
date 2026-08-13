# Spec versioning policy

Each specification carries an integer `Version` field in its header. The
number is a **compatibility class**, not a per-revision counter: `1` for a
formalization that a pre-existing component keeps working against, `2` for
one that requires coordinated change. Editorial revisions bump the revision
number in the spec header, not `Version`.

See [appendix/versioning.md](appendix/versioning.md) for the full compatibility-class
policy.
