---
type: observation
domain: []
confidence: 0.95
sources: 1
entities: [applyDedupMerge, mergeFacts, newFactWins, incomingIdentityWins, sameStatement, statementNotAppliedNotes, modelparams.Thresholds.Dedup, knomit_learn]
motifs: [silent-merge-loss, calibration-drift]
refs: ['src://7b4887ce51d9/internal/edge/mcp/learn.go@231e5711ef188edf9948b9abc41a742782e1472f:1bbfe5cc9d14848f1f746b682cf2e76207147e9c', 'src://7b4887ce51d9/internal/domain/modelparams/thresholds.go@231e5711ef188edf9948b9abc41a742782e1472f:ff98c50f6328ace368f92bb268d0b05181cd6e3b', 'src://7b4887ce51d9/internal/edge/mcp/learn_merge_keeps_statement_test.go@231e5711ef188edf9948b9abc41a742782e1472f:ca994b8823e5786ebc47c1c6bfda0e681276fdab', 'kb://3ec012f5b4d2/kb/principles/philosophy/dedup-not-append/adf1c4a5.md', 'kb://3ec012f5b4d2/kb/architecture/embeddings/cosine-threshold-calibration/c8254a76.md', 'https://github.com/knomit/knomit/pull/638']
---
# knomit_learn's dedup merge wrote a more confident SIBLING decision over the existing one (W0g2 over W0g1); the Dedup floor 0.82 had been outgrown by the corpus, and #638 raised it to the measured gap and stopped a merge replacing a different statement

**What happened.** On 2026-10-23 a W0g2 decision was learned. It matched the existing W0g1 decision (a sibling: one decision per step of a plan) at cosine 0.8616 under EmbeddingGemma, above the Dedup floor of 0.82. newFactWins gave the identity to the more confident fact, and mergeFacts wrote W0g2's title and body at W0g1's path. The sibling's statement was silently replaced.

**Why the floor failed.** 0.82 was ported from nomic's 0.92 and validated against a distinct same-category p99 of 0.77 on 712 facts. On 1,647 facts the corpus holds sibling families: one decision per step of a plan, and a guide's hub fact with its pages. Distinct same-directory p99 is now 0.874, and 21 pairs inside learn's merge scope (a category-directory prefix) sat above 0.82. Under nomic, seven distinct pairs in that scope sat above its 0.92.

**#638's fix.**
- Dedup is re-derived, not ported: the midpoint between the highest genuinely DISTINCT pair in learn's merge scope and the p01 of synthetic true near-duplicates, floored to 0.005. EmbeddingGemma: (0.8915 + 0.9214)/2 = 0.9065, floored to 0.905, catching 99.64% of synthetic near-duplicates. nomic: (0.9377 + 0.966)/2 = 0.952, floored to 0.95, catching 99.88%. The incident pair scores 0.8616 on gemma and 0.839 on nomic, so it is now below the floor.
- incomingIdentityWins = sameStatement && newFactWins. The existing text is kept unless the incoming fact states the same thing; statementNotAppliedNotes names every incoming fact whose title and body were not applied, and hypothesisMergeNote reports the stored (max) confidence.

**What this does NOT mean.**
- The floors are not model constants. They are this corpus's gap, measured once. Re-sweep with the method in thresholds.go when sibling families grow; a denser corpus can close the gap again.
- A merge above the floor still pools confidence, sources, refs and entities. The sameStatement guard protects only the title and body.
- A sibling in [ReflectNovelty, Dedup) inside the merge scope is now WRITTEN at its own path with a note and a `near:` commit line, not merged and not refused (see decisions/mcp/learn/in-scope-band-neighbour-written-not-refused).
