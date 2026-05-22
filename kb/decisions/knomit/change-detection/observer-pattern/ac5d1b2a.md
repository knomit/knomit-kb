---
type: principle
domain: [change-detection, indexing, sse]
confidence: 0.85
sources: 0
entities: [store.Service, observe.Observer, repos.Hub, idx.Sync, synthesize]
refs: [.claude/plans/2026-03-14-observer-change-detection-design.md]
---
# Use observer on commits, not inline idx.Upsert/Delete at each call site

Knomit replaced two redundant mechanisms with a single observer on git commits:

1. **MCP handlers** previously called `idx.Upsert`/`idx.Delete` manually after every write (learn.go, update.go, retract.go, prune.go, dedup.go, distill.go). This duplicated index-update logic everywhere.
2. **SSE handler** previously pushed HEAD via a 30s heartbeat ticker.

Both were redundant because all writes go through the same in-process `store.Service` — we can observe HEAD changes at the source and react once. The observer fires after each commit; a debounced callback runs `idx.Sync(gs)` (diff git from `meta.last_commit` to HEAD) and `hub.BroadcastStatus(hash)` (SSE push).

**Why kept inline for synthesize**: the synthesize pipeline (dedup, distill) reads the index between writes — searches for near-duplicates, writes the winner, deletes the loser, then searches again. A debounced async observer can't provide that immediate consistency. The design therefore kept synthesize's inline Upsert/Delete and relied on the observer's eventual `idx.Sync` to no-op afterward. (Verify current code: synthesize today calls `gs.DeleteFact`/`gs.WriteFile` only — confirm whether the index is still updated inline before relying on this.)

Design doc: `.claude/plans/2026-03-14-observer-change-detection-design.md` (status: Draft, but implemented in commits 8029733, 57c38e1, 6026e92 with deviations — see [[architecture-observer-wiring]]).
