---
kind: pragmatic
type: policy
domain: [fact, strings, dedup]
confidence: 0.95
sources: 0
entities: [UnionStrings, AppendUnique, dedup.mergeFacts]
refs: ['src://knomit/internal/fact/strings.go@307b67d']
---
# UnionStrings preserves insertion order (a first then b's new elements); used by dedup.mergeFacts

UnionStrings(a, b []string) []string (strings.go) returns a deduplicated union preserving INSERTION ORDER — a's elements first (in their order), then b's NEW elements (in their order). NOT a set operation that drops ordering. Used by dedup.mergeFacts for Domain/Entities union when merging two facts (the winner's tags come first). Companion AppendUnique(slice, s) is the single-element variant — appends only if not already present, also preserves order. Anti-pattern: implementing a custom union with map[string]struct{} iteration — Go map iteration is random, so the merged tags would shuffle between runs and produce noisy git diffs for facts that were semantically unchanged.
