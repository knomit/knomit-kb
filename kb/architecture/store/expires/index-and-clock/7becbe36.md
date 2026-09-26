---
type: observation
domain: [store, mcp, web, expires, index]
confidence: 0.9
sources: 1
entities: [facts.expires, facts.expires_at, GraphSchemaVersion, rebuildFacts, knomit_parse_fact, SearchOptions.Now, pagedRowState.AsOf, snapshotClock, migration 000026]
refs: ['src://7b4887ce51d9/internal/store/migrate/repo/000026_facts_expires.up.sql@c33b89b3e0af176702e18c411d8431e8a64c7e55:0c9a1558f6738832ea97e822c2aaf42bdcb93f5d', 'src://7b4887ce51d9/internal/store/search_index.go@c33b89b3e0af176702e18c411d8431e8a64c7e55:7a0f69bc0b73d76db0267379fc700f56e38a23c7', 'src://7b4887ce51d9/internal/mcp/query.go@c33b89b3e0af176702e18c411d8431e8a64c7e55:88efa74123b3dc00bb4f97f003eec895a37218bc', 'src://7b4887ce51d9/internal/store/search_query.go@c33b89b3e0af176702e18c411d8431e8a64c7e55:2699f31eb856014b814c5ea795d8f75ce510462a']
---
# `expires` is indexed as facts.expires (as written) + facts.expires_at (unix seconds), backfilled by the GraphSchemaVersion 5→6 forced rebuild; every read uses ONE clock — SearchOptions.Now set once per request, and a knomit_query cursor's pages reuse the snapshot's instant stored per row (pagedRowState.AsOf)

Index: migration 000026 adds nullable facts.expires (the frontmatter string, offset kept, for display) and facts.expires_at (unix seconds UTC, what every filter compares — RFC 3339 strings with different offsets do not order as text), plus a partial index on expires_at. BOTH facts-row writers set them: the incremental upsert and rebuildFacts (via the knomit_parse_fact UDF's `expires`/`expires_at` JSON keys). Backfill is NOT a data migration (derived state is regenerated from git): GraphSchemaVersion went 5→6, so every branch indexed by an older build reports NeedsRebuild on open and the rebuild fills the columns from the blobs — needed because a hand-written `expires:` could already sit in git while the old parser dropped it.

Clock: SearchOptions.Now is the read's clock, set ONCE per request by the handler (REST: timeNow() in the web package; MCP: parseQueryFilters), and the same value computes each row's `expired` marker, so filter and marker agree. A knomit_query cursor snapshot stores that instant in every row's pagedRowState.AsOf; resumed pages mark against it (snapshotClock), not page-serve time, so a query run with expired=false never shows expired:true on page 3. By-path reads (explain, REST fact view) and Highlights have no shared query clock and judge at the time of the call. The clock is the SERVER's.

WHAT THIS DOES NOT MEAN: there is no time-travel query and no anchor-at-commit evaluation — search is HEAD-only, and commit dates are not trusted as an anchor (rebase rewrites committer time).
