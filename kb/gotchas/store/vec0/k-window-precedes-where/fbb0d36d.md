---
type: pattern
domain: [store, sqlite, embeddings]
confidence: 0.9
sources: 1
entities: [titleKNNOverfetch, TopTitleNeighbours, fact_titles_vec, facts_vec, RelevantMethodologyForFact, graphBuildSimilarityEdges]
motifs: [limit-applied-before-filter]
refs: ['src://7b4887ce51d9/internal/store/abstraction.go@b57785357448cbbca9b3b43607fe33987f5522f2:d6720b7410d42f2f17dfefbaba68a7a140dd8293', 'https://github.com/knomit/knomit/pull/99']
---
# vec0 applies its `k` window BEFORE any WHERE clause — a KNN asking for exactly k and then filtering comes back short, or empty on a repo carrying many superseded revisions

`vec0` KNN is not a filterable scan. It selects its `k` nearest rows FIRST, and only then does SQLite apply the query's WHERE clause. Everything the WHERE removes is subtracted from an already-closed window.

**The consequence.** A query that asks for exactly `k` and then filters — to live facts, to one branch, to one `kind` — comes back SHORT. On a repo carrying many superseded revisions it can come back EMPTY, because the k nearest rows in the vec table may all be dead versions. The result is not an error and not obviously wrong; it is a plausible-looking short list.

**The remedy.** Over-fetch by a documented factor and filter down. `TopTitleNeighbours` asks for `k * titleKNNOverfetch` (4) and then applies its live/epistemic/branch predicates. `graphBuildSimilarityEdges` pays the same tax as `knnK+1`.

**The named misreading.** "The WHERE clause narrows the search" — it does not. It narrows the RESULT of a search that already finished. Adding a predicate to a vec0 KNN never makes it look harder for matches; it only throws away matches it already found.

This was already implicit in `RelevantMethodologyForFact`; recording it makes it findable before the next author writes the naive query.
