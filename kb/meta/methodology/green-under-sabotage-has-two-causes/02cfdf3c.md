---
kind: pragmatic
type: heuristic
domain: [methodology, testing, review]
confidence: 0.9
sources: 1
entities: [edgeReadFails, ScopedCluster, groupByCategory, pruneVerdictCertifiesDistinct, validatePrunePaths, TestInstanceConfig]
motifs: [guarantee-provided-elsewhere, test-mode-hides-condition]
refs: ['src://7b4887ce51d9/internal/synthesize/progression_test.go@7537940296cdc44dc489d10ac62eab5de162310d:1258d89b378477636a9feb6c4a0a02e26015c3ce', 'src://7b4887ce51d9/internal/synthesize/progression.go@7537940296cdc44dc489d10ac62eab5de162310d:4644551ffdb9e4b032e1a324298c069fa62202d7', 'src://7b4887ce51d9/internal/synthesize/cluster.go@7537940296cdc44dc489d10ac62eab5de162310d:704f41023bdb8044fd7482bae7001a00899f5e44', 'kb://3ec012f5b4d2/kb/meta/methodology/fixture-discrimination-precondition/f88dc045.md', 'kb://3ec012f5b4d2/kb/meta/methodology/guard-setup-outruns-hazard/2b230458.md', 'kb://3ec012f5b4d2/kb/meta/methodology/prove-the-regression-test-fails/d321b280.md', 'kb://3ec012f5b4d2/kb/gotchas/synthesize/cluster/category-scoped-neighbour-search/a408264a.md', 'https://github.com/knomit/knomit/pull/170', 'https://github.com/knomit/knomit/pull/164']
---
# A guard that stays GREEN under sabotage failed for one of two reasons — nothing exercises its call site, or a DIFFERENT line is quietly providing the guarantee you attributed to it — and the second is invisible to reading

Breaking a guard deliberately and watching it stay green is the only way either of these surfaces. They look identical from outside — a passing test — and they need different repairs.

**CAUSE 1 — the fixture cannot REACH the phenomenon at production config.** Not "the values fail to differ" (that is the discrimination precondition, a sibling fact) but "the code path under test never runs at all". Concrete: a gate suppressing clustered work-items stayed green when deleted, because at the production Louvain resolution NO community survives filterSmallClusters on a test-sized corpus — measured at 4, 8, 15 and 30 tightly-grouped facts, all zero clusters. There was never an item to suppress, so "none present" was a statement about an empty queue.

THE REPAIR IS TO REACH A DIFFERENT REAL CODE PATH, not to loosen production config from the test. ScopedCluster documents a fallback: when the subgraph edge read FAILS it groups by category directory, which for a single-directory fixture is exactly one cluster, deterministically. A test double whose SubgraphEdges returns an error is therefore a real path rather than a contrivance — and it stays inside the package under test, where widening a shared test-config struct would not. Then assert the phenomenon EXISTS (here: the prune item is present) so the absence being tested is suppression rather than emptiness.

**CAUSE 2 — a different line already provides the guarantee.** The guard is present, correct, and redundant with something downstream, so removing it changes nothing observable. Concrete: an explicit "an empty decision list is not an all-KEEP" check stayed green when deleted, because a separate coverage check ("every fact must have a decision") already rejects an empty answer for any non-empty item. The two only diverge on a ZERO-item input, where both loops run zero times and the predicate returns vacuous true.

THE REPAIR IS A JUDGEMENT, AND IT GOES BOTH WAYS. Ask whether the guard is REACHABLE at all:
- Genuinely unreachable → DELETE it. An unreachable guard invites a vacuous test and implies a hazard that does not exist. (Done for an output de-dup pass whose invariant was already enforced by an upstream seen-set.)
- Reachable but currently redundant → KEEP it, and extend the test to the case where the two diverge. A predicate that is correct only because of what its caller happens to do next is a trap for the next caller.

CONSEQUENCE a reader must act on: when a sabotage comes back green, do NOT conclude "the guard is fine, my sabotage was wrong". Determine WHICH cause it is, because cause 2 means your documentation is already false — you have attributed a guarantee to a line that is not providing it, and the next person to refactor the line that IS providing it will silently remove the protection.

WHAT THIS DOES NOT MEAN: it is not an argument for deleting redundant guards on sight (see the reachable case), and it is not the same as the fixture-discrimination precondition, which is about two values failing to differ rather than about a path never executing or a guarantee living elsewhere.
