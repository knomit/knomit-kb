---
kind: pragmatic
type: policy
domain: [mcp, synthesize, dedup]
confidence: 0.85
sources: 0
entities: [internal/mcp/learn.go, knomit_learn, dedupCluster, 'type: hypothesis']
refs: ['src://knomit/internal/mcp/learn.go@9f8780e5', 'src://knomit/internal/embeddings/model.go@9f8780e5', 'src://knomit/.claude/plans/2026-03-21-hypothesize-design.md@0938d83', kb/decisions/synthesize/dedup-merge-tiebreak/02422cde.md, kb/invariants/store/batch-write-atomicity/71c6dd17.md]
---
# On dedup collision with existing hypothesis: write observation, retract hypothesis, link via refs

When knomit_learn detects a near-duplicate of an existing fact AND the existing fact has type=hypothesis: apply retract-and-link instead of a normal dedup merge. Specifically: (1) write the new observation as normal; (2) retract the hypothesis; (3) add the hypothesis's path to the observation's refs (provenance: confirmed hypothesis X). This is the canonical 'hypothesis confirmed by observation' lifecycle. The change lives in internal/mcp/learn.go's dedup branch. Dedup between observations (no hypothesis involved) is unchanged — normal merge. Distinct from the symmetric case where reflect/review identifies a hypothesis as confirmed/retracted via the transition-detection pass.

CONSEQUENCE: because both halves must land together — the observation write AND the hypothesis retraction — this path goes through BatchWriteFacts so it is one commit. See kb/invariants/store/batch-write-atomicity/71c6dd17.md; this is the motivating case for that invariant.

WHAT THIS DOES NOT MEAN — THE TRIGGER IS NOT '≥0.92' (corrected 2026-07-22): this fact previously pinned the collision trigger at ≥0.92 similarity. That literal is stale. internal/mcp/learn.go reads `store.EmbedderThresholds(batchEmb).Dedup` at call time — a PER-MODEL calibrated cutoff, currently 0.82. Do not reimplement this test with a hardcoded number: against a model calibrated lower than your literal, the collision branch simply never fires and confirmed hypotheses pile up un-retracted alongside the observations that confirm them. See kb/decisions/synthesize/dedup-merge-tiebreak/02422cde.md.
