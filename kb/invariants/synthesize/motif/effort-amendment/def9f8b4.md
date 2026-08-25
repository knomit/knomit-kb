---
kind: pragmatic
type: policy
domain: [synthesize, motif, effort]
confidence: 0.9
sources: 1
entities: [EffortNormal, review_effort_normal_test.go, motif backfill, motif alias, motif define]
refs: ['src://7b4887ce51d9/internal/synthesize/motif_backfill.go@b57785357448cbbca9b3b43607fe33987f5522f2:71a1e6879fd0ede27cb983f0dc7c066a4c6c17b7', 'src://7b4887ce51d9/internal/synthesize/motif_alias.go@b57785357448cbbca9b3b43607fe33987f5522f2:59e4a8b5233405e14f4d36ae9eace3cafc3d0327', 'https://github.com/knomit/knomit/pull/102']
---
# All three motif vocabulary passes are gated at effort ≥ medium, because EffortNormal guarantees BYTE-IDENTICAL output — and the test that proves it is the one a backfill would break

**The rule.** Alias resolution, blind definitions, and the backfill pass — every vocabulary pass — are gated at effort ≥ medium. `EffortNormal` must produce byte-identical output regardless of corpus motif state.

**Why this specific gate, and not a weaker one.** Backfill fires on every fact of a motif-FREE corpus — which is exactly the corpus `review_effort_normal_test.go` (MN5) uses. Ungated, the very test that guarantees normal-effort stability is the first thing the feature breaks. The gate is not conservatism about cost; it is what keeps the guarantee provable.

**The standing evidence.** `internal/synthesize/review_effort_normal_test.go` is byte-identical to `dev` at every commit across all six PRs of the campaign (#99, #100, #102, #105, #106, #109) — and the check is diff-based, so it PASSES rather than silently skipping.

**The named misreading.** "Effort ≥ medium" is not a performance tier for motif work. It is the boundary of a correctness guarantee: anything that can alter output at normal effort belongs above it, whatever it costs.

**Consequence for a new motif-adjacent pass.** Before adding one, ask whether it can run on a corpus with no motifs. If yes, it must be gated — a pass that no-ops on motif-free corpora by accident rather than by gate will start mutating output the moment the corpus gains its first motif.
