---
kind: pragmatic
type: policy
domain: [synthesize, motif, effort]
confidence: 0.95
sources: 1
entities: [EffortNormal, review_effort_normal_test.go, motif backfill, motif alias, motif define]
refs: ['src://7b4887ce51d9/internal/synthesize/review_strategy.go@b57785357448cbbca9b3b43607fe33987f5522f2:0893cd93feb551600da68a4c7fe9023f92d9bb2c', 'src://7b4887ce51d9/internal/synthesize/motif_backfill.go@b57785357448cbbca9b3b43607fe33987f5522f2:71a1e6879fd0ede27cb983f0dc7c066a4c6c17b7', 'src://7b4887ce51d9/internal/synthesize/motif_alias.go@b57785357448cbbca9b3b43607fe33987f5522f2:59e4a8b5233405e14f4d36ae9eace3cafc3d0327', 'src://7b4887ce51d9/internal/fact/motif.go@b57785357448cbbca9b3b43607fe33987f5522f2:5a4d3a0af836755797d4ec47246c5a134d90f9c9', 'src://7b4887ce51d9/internal/synthesize/dedup.go@b57785357448cbbca9b3b43607fe33987f5522f2:167c9d4727768037e4f9a0859805b2a742e38cf8', 'kb://3ec012f5b4d2/kb/decisions/synthesize/motif/derived-writer-rules/aba23e53.md', 'https://github.com/knomit/knomit/pull/102']
---
# The three motif VOCABULARY passes are gated at effort ≥ medium because EffortNormal guarantees byte-identical OUTPUT — but the mechanical motif merge is ungated and runs at every effort, including on the runtime learn path

**THE RULE, scoped precisely.** Three passes — alias resolution, blind definitions, and backfill — run only when `d.Effort.MaintainsVocabulary()` (effort ≥ medium). The alias REBUILD runs first and unconditionally inside that gate.

**Why the gate is placed by OUTPUT, not by cost.** All three spend LLM budget, but that is not the reason. Backfill fires on any authored fact LACKING a motif — which is every fact on a motif-free corpus, exactly the corpus MN5's `review_effort_normal_test.go` uses. Running it at normal effort would change what a normal-effort session PRODUCES, not merely what it costs. The gate is the boundary of a correctness guarantee; anything that can alter output at normal effort belongs above it, whatever it costs.

**WHAT THIS DOES NOT MEAN — the dangerous compression.**
- **It does NOT mean "motifs are inert below medium effort".** The mechanical motif union `fact.MergeMotifs` is NOT effort-gated and has two callers, neither behind this gate: `internal/synthesize/dedup.go` (review dedup-merge, every effort) and `internal/mcp/learn.go` (dedup-on-learn — the RUNTIME write path, which has no effort concept at all). Motifs are written, merged and trimmed at normal effort and outside review sessions entirely.
- **It does NOT mean the guarantee is violated by that.** The merge is mechanical and deterministic, so it does not change normal-effort OUTPUT on the MN5 corpus — which is precisely why it needs no gate. Cost is not the criterion; output stability is.
- **It does NOT mean "gate any new motif code at medium".** Ask the actual question: can this run on a corpus with no motifs, and would it change output there? Mechanical and deterministic → no gate needed. Anything LLM-driven or corpus-state-dependent → gate it, or it starts mutating output the moment the corpus gains its first motif.

**The standing evidence.** `internal/synthesize/review_effort_normal_test.go` is byte-identical to `dev` at every commit across all six PRs of the campaign (#99, #100, #102, #105, #106, #109), and the check is diff-based, so it PASSES rather than silently skipping.

**Ordering note.** The alias rebuild runs BEFORE selection, not after: the judge's verdicts are recorded against the membership their clusters have NOW, and a rebuild that ran only afterwards would stamp every verdict with membership one session stale, retiring decisions the moment they were made. Its earlier absence was a bootstrap DEADLOCK whose every symptom read as a correct-looking zero.
