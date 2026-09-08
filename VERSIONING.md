# Spec versioning policy

Each specification carries an integer `Version` field in its header. The
number is a **compatibility class**, not a per-revision counter: `1` for a
formalization that a pre-existing component keeps working against, `2` for
one that requires coordinated change. The header carries the class and
nothing else: revisions within a class are recorded as entries in
[CHANGELOG.md](CHANGELOG.md), one per pull request that changes a requirement
or a wire surface, and in the git history.

See [appendix/versioning.md](appendix/versioning.md) for the full compatibility-class
policy.
