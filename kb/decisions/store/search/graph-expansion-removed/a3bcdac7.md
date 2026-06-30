---
type: observation
domain: [store, search, web, graph, performance]
confidence: 0.9
sources: 1
entities: [searchIndex.Search, SearchOptions.GraphHops, graphExpandSearch, handleSearch, internal/store/search_graph.go, internal/store/search_query.go, internal/web/handlers_search_hal.go, api.search]
refs: ['src://knomit/internal/store/search_query.go@caaed87', 'src://knomit/internal/web/handlers_search_hal.go@caaed87']
---
# Search is pure vector ranking; graph-augmented expansion (GraphHops/graphExpandSearch) was removed as unfinished, inert dead code

The web /search endpoint (handleSearch in handlers_search_hal.go) was the ONLY caller of searchIndex.Search that ever set GraphHops>0 — MCP knomit_query, the domains handler, and all of synthesize leave it 0. Its 'graph-augmented search' (graphExpandSearch in search_graph.go, introduced 2026-03-12 in commit cef891d) took the vector-KNN hits as seeds and pulled in graph NEIGHBORS via two Cypher queries — one over SIMILAR_TO edges, one over shared-entity (TAGGED) co-occurrence — each batched as a ~200-way `f.path = ".." OR ..` chain stringified into a cypher('...') literal (OR-chaining because the installed GraphQLite build lacks IN-list support). That double Cypher pass over the OR-chain was the ENTIRE multi-second search latency (~7.5s on the trunk repo; constant regardless of query text, since the vector phase and SQL hydration are sub-second).

It was both unfinished and inert: the maxHops parameter was never implemented (always exactly one hop, as its own doc comment admitted), it silently swallowed Cypher errors (flagged in an old code review), and — decisively — graph neighbors were scored BELOW every vector hit (capScore = minSeedScore-0.01) while the vector phase already returns up to 5x limit (250) candidates. So the final sort-by-score + trim-to-limit (50) discarded every neighbor in any query with >=50 vector matches above the floor, i.e. essentially all real queries. It changed no output.

Resolution: set GraphHops:0 on the endpoint (commit 0cd91fc), then removed the feature entirely (commit caaed87) — graphExpandSearch, the GraphHops field on SearchOptions, the gated call site in searchIndex.Search, and the regression test (the field's absence is now the guarantee). Search latency dropped ~7.5s -> <1s with identical results. NOTE: search is a single blocking GET (api.search -> /search), not streamed; a filter-bar spinner was added on the web side (state.searching, set by the Library relevance effect) since there is no incremental rendering to lean on.
