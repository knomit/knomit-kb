---
type: observation
domain: [mcp, learn, dedup, scenario]
confidence: 0.85
sources: 1
entities: [applyDedupMerge, checkSameSubjectCollisions, citedIn, namedIn, distinct_from, learn_dedup, mergeFacts, internal/mcp/learn.go, internal/mcp/learn_same_subject.go]
motifs: [escape-hatch-unreachable, silent-merge-loss]
refs: ['kb://3ec012f5b4d2/kb/gotchas/mcp/learn/dedup-search-scope-is-a-string-prefix/047f3cdd.md', 'kb://3ec012f5b4d2/kb/decisions/mcp/learn/distinct-from-honoured-by-merge/91cf6418.md', 'kb://3ec012f5b4d2/kb/decisions/mcp/learn/same-subject-refusal/f77be375.md', 'src://7b4887ce51d9/internal/mcp/learn.go@087f420f1d07833459f52cca48496d0da75fbc5e:935ca00014550e710a28e880e864676c788b66c4', 'src://7b4887ce51d9/internal/mcp/learn_same_subject.go@087f420f1d07833459f52cca48496d0da75fbc5e:4b470560e41c6b2cc5b43287f56b312791d6aa55', 'src://7b4887ce51d9/internal/synthesize/scenario_load.go@087f420f1d07833459f52cca48496d0da75fbc5e:f787748b27a732088d24b3702e4b97824f724b80', 'https://github.com/knomit/knomit/pull/499']
---
# knomit_learn's dedup merge folds ANY near-duplicate non-hypothesis fact under the incoming category prefix, even one the incoming fact cites; citing it escapes only the later same-subject gate, and the escape from the merge is distinct_from

At 087f420f applyDedupMerge (internal/mcp/learn.go) runs a Limit 1 search (learn.go:773) over the incoming fact's category directory as a raw path PREFIX, so the candidate may be in the same category, a descendant category, or a prefix-sibling category (see the scope gotcha). The match is declined ONLY for: the incoming topic or the match's topic having learn_dedup: off (learn.go:727-746 and the candidate check before consumed), the match already consumed by an earlier fact in the same call (learn.go:809), the match being retracted in the same call (learn.go:816), or the match being named in that input's distinct_from (learn.go:838). A hypothesis match that the incoming non-hypothesis fact subsumes is retracted and the incoming fact written at its own path. Otherwise the incoming fact is folded into the match and written at the EXISTING path. Nothing in this stage looks at the incoming fact's refs: a fact that CITES the near-duplicate is folded into it all the same.

Citing a fact escapes only the same-subject gate: checkSameSubjectCollisions skips a candidate already cited in refs (citedIn, learn_same_subject.go:468, defined at 511) and one named in distinct_from. That gate runs AFTER the merge (learn.go call sites 1114 then 1131) and is skipped for a fact the merge touched. So `refs` is not an escape from the merge.

Consequence: to keep a fact separate from a near-identical neighbour within the category prefix, name the neighbour in distinct_from (or reword it away from the neighbour); do not rely on citing it. Scenario worlds hit this: the loader reports a dedup merge that folds a world fact into another as an invalid world. Misreading to avoid: this is not 'any near-duplicate anywhere is merged'; the reach is bounded by the category-directory string prefix and the embedder's dedup threshold, and without an embedder nothing is merged.
