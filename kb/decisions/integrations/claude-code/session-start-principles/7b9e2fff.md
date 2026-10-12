---
type: observation
domain: [integrations, claude-code, hooks, bridge]
confidence: 0.9
sources: 1
entities: [SessionContext, fetchGlobalPrinciples, GlobalPrinciplesURL, FilterGlobalPrinciples, handleHALFactsCollection, RecentFacts, globalsPageSize]
motifs: [silent-truncation-keeps-newest]
refs: ['src://7b4887ce51d9/tools/bridge/knomitapi/session.go@ca7a18212ec0192ee83e231f4e07c2194af25b33:5b4c8c19840d73f0570ace3014727832e85e8d73', 'src://7b4887ce51d9/tools/bridge/knomitapi/facts.go@ca7a18212ec0192ee83e231f4e07c2194af25b33:2e98b566bb79918e1bbe11e57357fb2721ea0585', 'src://7b4887ce51d9/tools/bridge/knomitapi/session_test.go@ca7a18212ec0192ee83e231f4e07c2194af25b33:4005511dff0c1a0bbed923ff51be4d4957a43541', 'https://github.com/knomit/knomit/pull/494']
---
# The SessionStart hook injects EVERY global designer principle, ordered by path — the /facts handler has no sort and orders by committed_at DESC, so any cap silently keeps only the newest

The PROJECT PRINCIPLES block that SessionContext (shared by the Claude Code SessionStart hook and the Antigravity PreInvocation hook) prints contains every fact under kb/principles/ whose entities include designer and whose domain includes global. It used to be capped at 7 (maxGlobals); because the /facts handler (handleHALFactsCollection -> RecentFacts) accepts no sort parameter and always orders by committed_at DESC, path ASC, the cap kept only the 7 most recently committed principles, so older ones such as anti-patterns/solution-quality-pre-release never reached a session and a new principle fell out after seven later writes.

Now fetchGlobalPrinciples pages with limit=500 (the handler's maxLimit) following offset, with a 20-page runaway guard, then sorts by path so the block is stable across sessions and does not reshuffle when any principle is written. maxInvariants, maxRecent, the recent-minus-already-shown logic and the no-globals invariants fallback are unchanged.

Consequence: do not reintroduce a cap or a count-based truncation on global principles; if the block ever needs trimming, the order to trim by is a deliberate choice, not the server default. Misreading to avoid: sort=recent on /facts does not change the order, it is the only order. Admission to the block is the two-axis filter (designer AND global), not recency.
