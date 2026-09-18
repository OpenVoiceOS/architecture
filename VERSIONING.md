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

## Removed mechanisms

A specification **removes** a mechanism when it takes away a class of
behaviour and names no successor: there is nothing to migrate to, only
something to stop doing. This is different from a **replaced** mechanism,
which has a named successor in the predecessor-topic mapping of
[appendix/divergences.md](appendix/divergences.md) and migrates by speaking
both for one stable cycle.

An implementation **MAY** keep emitting a removed mechanism for one stable
release cycle after it adopts the specification that removes the mechanism, so
that a consumer which still reads it is not broken by a producer that upgrades
first. It **MUST** log a deprecation that names the release which removes it,
and it **MUST NOT** begin to read one it did not already read: the window
exists for consumers that predate the removal, not for new dependants. After
that cycle a conforming implementation neither emits nor reads it.

One cycle is the rule, not a judgement made once per mechanism. Where a
removal needs a different window, or keeps only part of the mechanism, the
record for it in [appendix/divergences.md](appendix/divergences.md) states
which part and why, as the out-of-band session pushes do in their V1 removal
scope. A record that states no window takes the one cycle above.
