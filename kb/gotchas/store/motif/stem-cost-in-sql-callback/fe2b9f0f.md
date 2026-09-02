---
type: observation
domain: [store, motif, performance, web, lens]
confidence: 0.95
sources: 1
entities: [textnorm.Stem, textnorm.Tokens, store.groupingKey, knomit_motif_key, motifClusterKeyExpr, motifIndex.ClustersUnder, motif_aliases, go-pluralize, BenchmarkStemCold, internal/textnorm/textnorm.go, internal/store/motif_alias.go, internal/store/sqlfuncs.go]
motifs: [helper-becomes-hot-loop, two-conditions-hide-cost]
refs: ['src://7b4887ce51d9/internal/store/motif_alias.go@752e282d84bbdb0ff7fdfa30a3537fad30a1a7f8:ff5c7fb7f4bb586ba108df73f03bd191446a2e05', 'src://7b4887ce51d9/internal/store/sqlfuncs.go@752e282d84bbdb0ff7fdfa30a3537fad30a1a7f8:a3eb60ef8895d1224d60215cce956b59eef92aaf', 'src://7b4887ce51d9/internal/textnorm/textnorm.go@752e282d84bbdb0ff7fdfa30a3537fad30a1a7f8:0513f506af6b126c74d6416cbf70ea962715f0ee', 'src://7b4887ce51d9/internal/textnorm/stem_memo_bench_test.go@752e282d84bbdb0ff7fdfa30a3537fad30a1a7f8:af46887dff134b41cf9d5582dcb5748c9b420ac8', 'kb://3ec012f5b4d2/kb/decisions/lens/motif/cross-mount-cluster-identity/1d35c7b0.md', 'https://github.com/knomit/knomit/pull/183']
---
# textnorm.Stem costs ~25µs per token and the knomit_motif_key SQL callback pays it PER ROW — but only on branches motif_aliases does not cover, and that second condition is why it hid

go-pluralize's `Singular` runs a table of backtracking regexes: `textnorm.Stem` measured **~25µs per token** uncached, and `textnorm.Tokens` on a four-token motif ~45µs (BenchmarkStemCold / BenchmarkTokensCold). `store.groupingKey` calls Tokens, and `groupingKey` is registered as the SQLite function `knomit_motif_key` (sqlfuncs.go), which `motifClusterKeyExpr` embeds in the `resolved` CTE of `ClustersUnder`. So the price was paid per token, per motif, PER ROW, inside the query, on every request.

THE CONDITION IS COMPOUND, AND BOTH HALVES MATTER — this is the part that compresses into a falsehood:

1. **Per row, not per distinct motif.** One instrumented `GET /lenses/all/motifs` over five mounts made 576 / 732 / 68 / 640 / 0 groupingKey calls, against branches holding only ~131 DISTINCT motifs. The CTE is evaluated more than once and it is keyed per row, so call count tracks (fact, motif) pairs, not vocabulary size.
2. **Only where the alias table does not cover the branch.** `motifClusterKeyExpr` is `COALESCE(NULLIF(a.cluster_key,''), NULLIF(knomit_motif_key(<col>),''), <col>)`, and COALESCE short-circuits — a branch whose motifs all have a stored `cluster_key` NEVER calls the function, and there is nothing to save there. Coverage is per branch_id in `motif_aliases`, so a freshly created agent branch (which a repo takeover makes) starts at ZERO coverage and pays for every row; a partially covered branch pays for the rest.

MEASURED, on a five-mount lens over real corpora: with zero coverage a single repo's `/motifs` took ~40ms and the lens 126.8ms; with partial coverage the same repo took 9.5ms. groupingKey was 90–96% of `ClustersUnder`'s wall time on every mount that called it.

WHY IT HID: neither condition alone is enough to notice. The helper is pure, cheap-looking and correct; the SQL reads as one query; and on a freshly rebuilt corpus the cost is exactly zero. It only appears on a corpus with facts written since the last alias rebuild — which is the ordinary working state, and the state a benchmark on a clean fixture will not reproduce.

CONSEQUENCE FOR A CONSUMER: before concluding that a motif surface is slow because of the lens fan-out, check alias coverage for the branch being read (`SELECT branch_id, count(*) FROM motif_aliases`) — the lens cost is exactly the SUM of its mounts, with no superlinearity and no server-side N+1, so a slow lens means a slow mount. And treat any pure helper reached from a registered SQL function as being in a hot loop: the call site is a row, not a request.

WHAT THIS DOES NOT MEAN: not that the federation or the per-request union rebuild was the problem — that hypothesis was measured and rejected. Not that `ClustersUnder` is slow in general; on a fully alias-resolved branch it never enters this path. And not that the fix was to cache the union — see the sibling decision fact on memoizing Stem.

FOUND BY: a macOS `sample` stack profile under load (every on-CPU leaf was regexp backtracking under go-pluralize), then a build instrumented to count and time groupingKey calls per ClustersUnder. Two earlier measurement attempts produced wrong numbers — one measured a server still finishing its first-boot index rebuild, and one measured a binary that was never actually running, because the pkill pattern did not match the running command line and the old process kept the port.
