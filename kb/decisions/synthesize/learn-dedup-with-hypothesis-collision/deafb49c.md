---
kind: pragmatic
type: policy
domain: [mcp, synthesize, dedup]
confidence: 0.85
sources: 0
entities: [internal/edge/mcp/learn.go, knomit_learn, dedupCluster, 'type: hypothesis']
refs: ['src://7b4887ce51d9/internal/edge/mcp/learn.go@0610c20083fcef9e84f42ab0311a14708bcb6193:fa685369f268f0187648eb3af58af3b643566432', 'src://7b4887ce51d9/internal/engine/embeddings/model.go@0610c20083fcef9e84f42ab0311a14708bcb6193:2c02a9a8ef02d5a41f33369aacd352d1ca0d26af', 'src://knomit/.claude/plans/2026-03-21-hypothesize-design.md@0938d83', 'kb://3ec012f5b4d2/kb/decisions/synthesize/dedup-merge-tiebreak/02422cde.md', 'kb://3ec012f5b4d2/kb/invariants/store/batch-write-atomicity/71c6dd17.md']
---
# On dedup collision with existing hypothesis: write observation, retract hypothesis, link via refs

When knomit_learn detects a near-duplicate of an existing fact AND the existing fact has type=hypothesis: apply retract-and-link instead of a normal dedup merge. Specifically: (1) write the new observation as normal; (2) retract the hypothesis; (3) add the hypothesis's path to the observation's refs (provenance: confirmed hypothesis X). This is the canonical 'hypothesis confirmed by observation' lifecycle. The change lives in internal/edge/mcp/learn.go's dedup branch. Dedup between observations (no hypothesis involved) is unchanged — normal merge. Distinct from the symmetric case where reflect/review identifies a hypothesis as confirmed/retracted via the transition-detection pass.

CONSEQUENCE: because both halves must land together — the observation write AND the hypothesis retraction — this path goes through BatchWriteFacts so it is one commit. See kb/invariants/store/batch-write-atomicity/71c6dd17.md; this is the motivating case for that invariant.

WHAT THIS DOES NOT MEAN — THE TRIGGER IS NOT '≥0.92' (corrected 2026-07-22): this fact previously pinned the collision trigger at ≥0.92 similarity. That literal is stale. internal/edge/mcp/learn.go reads `store.EmbedderThresholds(batchEmb).Dedup` at call time — a PER-MODEL calibrated cutoff, currently 0.82. Do not reimplement this test with a hardcoded number: against a model calibrated lower than your literal, the collision branch simply never fires and confirmed hypotheses pile up un-retracted alongside the observations that confirm them. See kb/decisions/synthesize/dedup-merge-tiebreak/02422cde.md.

UPDATE (#642, merge 77896f0a): two corrections. (1) The title and the lifecycle text above say 'confirmed hypothesis', but the branch fires on SIMILARITY to the stored hypothesis, not on a verdict, so it equally retracts a hypothesis the new observation REFUTES. 'Settled' is the accurate word; the wording here is still wrong and is tracked in issue #641 (https://github.com/knomit/knomit/issues/641), which is open and NOT fixed by #642. (2) The retraction is no longer silent: learn adds a response note per subsumed hypothesis ('retracted hypothesis <path> (settled by this fact, cited in refs); to keep it, re-learn it with distinct_from: [<new fact path>]') and a 'settled: <path>' commit line; `retracted` and `commit` stay the caller's own retract paths. The undo is re-learning with distinct_from, never knomit_update. See kb/gotchas/mcp/learn-hypothesis-report/612294ef.md and the ruling in kb/decisions/mcp/learn-hypothesis-report/e8f1082b.md.
