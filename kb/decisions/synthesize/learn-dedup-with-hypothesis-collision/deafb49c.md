---
kind: pragmatic
type: policy
domain: [mcp, synthesize, dedup]
confidence: 0.85
sources: 0
entities: [internal/mcp/learn.go, knomit_learn, dedupCluster, 'type: hypothesis']
refs: ['src://knomit/.claude/plans/2026-03-21-hypothesize-design.md@0938d83']
---
# On dedup collision with existing hypothesis: write observation, retract hypothesis, link via refs

When knomit_learn detects a near-duplicate (≥0.92 similarity) of an existing fact AND the existing fact has type=hypothesis: apply retract-and-link instead of normal dedup merge. Specifically: (1) write the new observation as normal; (2) retract the hypothesis; (3) add the hypothesis's path to the observation's refs (provenance: confirmed hypothesis X). This is the canonical 'hypothesis confirmed by observation' lifecycle. Change lives in internal/mcp/learn.go's dedup branch. The dedup behavior between observations (no hypothesis involved) is unchanged — normal merge. Distinct from the symmetric case where reflect/review identifies a hypothesis as confirmed/retracted via the transition-detection pass.
