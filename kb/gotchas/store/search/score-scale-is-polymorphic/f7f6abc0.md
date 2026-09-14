---
type: observation
domain: [store, search, embeddings]
confidence: 0.9
sources: 1
entities: [store.SearchResult, SearchOptions.MinSimilarity, params.Thresholds, factQuery.Search, internal/store/search_query.go]
motifs: [same-name-different-scale]
refs: ['src://7b4887ce51d9/internal/store/search_query.go@920978789f251c550d62f292b9878125b90027d3:8be8e6b74702c0a4e607c7fc62339c6256c7d8cf', 'kb://3ec012f5b4d2/kb/decisions/mcp/learn/same-subject-refusal/f77be375.md', 'kb://3ec012f5b4d2/kb/decisions/mcp/learn/same-subject-band-endpoints/0c5a0529.md', 'https://github.com/knomit/knomit/pull/187']
---
# store.SearchResult.Score is cosine*100 on the vector path and a literal 100 on the text-less path — comparing it against a params.Thresholds value silently does nothing

`store.SearchResult.Score` is NOT on the same scale as the cosine thresholds in `params.Thresholds`, and it is not even on ONE scale:

- Vector path: `Score = cosine * 100.0`, i.e. 0–100.
- Text-less path (`Text`, `QueryByPath` and `QueryVec` all empty): `Score` is the LITERAL 100 for every row, and `MinSimilarity` is ignored entirely — the query returns everything matching the non-text filters.

Meanwhile `SearchOptions.MinSimilarity` is 0–1, so ONE struct carries both conventions: you pass a cosine in and get a percentage back.

THE FAILURE IS SILENT, WHICH IS WHY THIS IS WORTH KNOWING. A post-filter written as `if result.Score < th.Dedup { … }` compares something like `71.0 < 0.82`, which is false for every row — so a gate built on it accepts nothing, refuses nothing, and no test fails unless a test asserts the POSITIVE case. The same-subject gate was specified with exactly that comparison and would have shipped as a permanent no-op; it divides by 100 before any threshold comparison.

CONSEQUENCE: divide `Score` by 100 before comparing it to any `params.Thresholds` field, and never reach the text-less path with a threshold in hand — there, `Score` is not a similarity at all. A caller relying on `MinSimilarity` must ensure `Text`, `QueryVec` or `QueryByPath` is set, or the filter is not applied.

WHAT THIS DOES NOT MEAN: not "Score is broken". 0–100 is a reasonable display scale for a ranked result list, which is what it was built for. The trap is that its scale is invisible at the call site and differs by code path.
