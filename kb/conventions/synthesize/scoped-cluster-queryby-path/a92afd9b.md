---
kind: pragmatic
type: policy
domain: [synthesize, search, embedding]
confidence: 0.95
sources: 0
entities: [ScopedCluster, dedupCluster, QueryByPath, facts_vec, SearchOptions]
refs: ['src://7b4887ce51d9/internal/corpus/synthesize/cluster.go@0610c20083fcef9e84f42ab0311a14708bcb6193:37ec8bc8e3b5633e4e55ca5bae378c7cfeacff46', 'src://7b4887ce51d9/internal/corpus/synthesize/dedup.go@0610c20083fcef9e84f42ab0311a14708bcb6193:b8c401ef89c4b1ad3ed2eacadb40373efdeadc07']
---
# ScopedCluster and dedupCluster use QueryByPath — NEVER re-embed indexed facts

ScopedCluster (neighbor lookup) and dedupCluster (pair discovery) both use store.SearchOptions{QueryByPath: <member.Path>} instead of re-embedding seed/cluster bodies. QueryByPath resolves each member's stored vector via a SQL subquery in the sqlite-vec MATCH operand — ONE SQL statement replaces an ONNX inference + KNN. Every cluster member is already an indexed fact whose 768-dim vector lives in facts_vec from when it was learned, computed over the same title+body content we'd otherwise re-embed here. Anti-pattern: building a query string from seed.Body and calling Search(Text: body) — wastes inference cost AND uses a different content slice than the indexed vector (search results would differ from what graph similarity already computed). This is a deliberate optimization documented in review.go and cluster.go comments.
