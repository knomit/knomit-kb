---
type: observation
domain: [store, search, mcp, refactor-backlog]
confidence: 0.8
sources: 0
entities: [RecentFacts, recentFactsSearch, SearchQuery, internal/mcp/query.go]
refs: ['src://knomit/.claude/plans/2026-05-14-pragmatic-kind-followups.md@0938d83']
---
# RecentFacts param pack should become a struct before adding more filters

RecentFacts / recentFactsSearch in internal/store/search_query.go already take 11 positional parameters. SearchQuery.IncludeKinds/ExcludeKinds exist and the SQL filter is reachable via the public Search(SearchQuery) method, but they are NOT threaded through RecentFacts because adding two more []string positional args would push past readability. RIGHT SHAPE: refactor the parameter pack into a RecentFactsOpts/SearchFilters struct first; after that exists, adding kind filters is one field. Same deferral applies to /knomit_query MCP tool exposing include_kinds/exclude_kinds (mirrors existing include_types/exclude_types). Touches: internal/store/search_query.go, internal/store/interfaces.go:38, internal/web/handlers_facts_collection.go, mock + test callers in internal/synthesize/. Also: internal/mcp/ has NO _test.go files — handler-level e2e tests (learn pragmatic policy → query it back) is a pre-existing gap.
