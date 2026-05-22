---
kind: pragmatic
type: policy
domain: [synthesize, search, embedding]
confidence: 0.95
sources: 0
entities: [ScopedCluster, dedupCluster, QueryByPath, facts_vec, SearchOptions]
refs: ['src://knomit/internal/synthesize/cluster.go@307b67d', 'src://knomit/internal/synthesize/dedup.go@307b67d']
---
# ScopedCluster and dedupCluster use QueryByPath — NEVER re-embed indexed facts

ScopedCluster (neighbor lookup) and dedupCluster (pair discovery) both use store.SearchOptions{QueryByPath: <member.Path>} instead of re-embedding seed/cluster bodies. QueryByPath resolves each member's stored vector via a SQL subquery in the sqlite-vec MATCH operand — ONE SQL statement replaces an ONNX inference + KNN. Every cluster member is already an indexed fact whose 768-dim vector lives in facts_vec from when it was learned, computed over the same title+body content we'd otherwise re-embed here. Anti-pattern: building a query string from seed.Body and calling Search(Text: body) — wastes inference cost AND uses a different content slice than the indexed vector (search results would differ from what graph similarity already computed). This is a deliberate optimization documented in review.go and cluster.go comments.
