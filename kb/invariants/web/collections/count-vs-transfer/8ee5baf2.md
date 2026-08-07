---
kind: pragmatic
type: policy
domain: [web, api, lens, collections]
confidence: 0.95
sources: 1
entities: [internal/web/handlers_lenses_read.go, internal/web/handlers_facts_collection.go, internal/store/search_query.go, RecentFacts, maxLensRecencyDepth, maxLensSearchCandidates, lensFanoutDepth, TestLensFanoutDepth_OverflowingOffsetStillClamps]
refs: ['src://7b4887ce51d9/internal/web/handlers_lenses_read.go@a11f174ecd6ce171baeb046372151d7c77b3e141:0cc988a4d6ced809be08ff38638f2139e54e235d', 'src://7b4887ce51d9/internal/web/handlers_lenses_read_test.go@a11f174ecd6ce171baeb046372151d7c77b3e141:306819fc323c08d21ecc296d2c49fefa9cf34f45', 'src://7b4887ce51d9/internal/store/search_query.go@a11f174ecd6ce171baeb046372151d7c77b3e141:0c7b62cb06d9b027a1decccb55f5410a85b2a333', 'src://7b4887ce51d9/internal/web/handlers_facts_collection.go@a11f174ecd6ce171baeb046372151d7c77b3e141:fae1a2e17de23552606aed1b282f4cc4610381b6']
---
# A collection's `total` answers how many rows MATCH, never how many were fetched — bounding transfer must never bound the count

Two independent questions live in every paged collection endpoint, and they must be answered by two independent mechanisms:

1. **How many rows match this query.** Answer with a real `SELECT COUNT(*)` over the filter set. This is O(index), independent of `limit`/`offset`, and there is no case where it is unavailable — the store already computes it. `store.factQuery.RecentFacts` returns `([]RecentFactEntry, int, error)` where the int IS that count.
2. **How many rows to ship.** Answer with paging. Bound the transfer, never the count.

**The failure this encodes.** The lens union facts handler fetched a fixed 500 rows per mount, discarded each mount's returned count (`entries, _, err`), and reported `len(merged)` as `total`. A lens over a 1403-fact mount therefore said 500, while `/lenses/{l}/stats` — which sums each mount's real total — said 1403 for the identical corpus on the identical screen. The accurate number was being computed, returned, and thrown away once per mount per request.

**Consequence to act on.** When you add or review a collection endpoint: if `total` is derived from the length of anything you fetched, sliced, merged or deduped, it is wrong. Trace it to a COUNT(*). The repo facts collection (`handlers_facts_collection.go`) is the correct pattern; it always used the returned count.

**The transfer bound must itself be checked for NON-POSITIVE, not only for over-cap.** `lensFanoutDepth` computes the recency depth as `offset + limit`, clamping it to `maxLensRecencyDepth`. `offset` is any non-negative int a client can send, so `offset + limit` can WRAP to negative — and a negative depth passes a one-sided `> max` test, reaching the store as `LIMIT -1`, which SQLite reads as NO limit. Every mount then materialises its whole corpus: precisely the absurd-offset case the constant's doc comment promises to prevent, arrived at by CLEARING the guard rather than tripping it. The guard is now `if depth < 0 || depth > maxLensRecencyDepth`.

Generalised: **a computed depth must be checked for non-positive as well as over-cap, because a non-positive `Limit` disables the limit instead of tightening it.** Anywhere a bound is derived by arithmetic rather than taken as a literal, the derived value is not automatically a bound.

**Named misreading — "the cap is the only failure mode."** That is what makes a one-sided comparison look complete: the reviewer checks that a too-large depth is clamped, sees it is, and stops. The other end is not merely a smaller number, it is a different SEMANTIC — in SQLite, and in most drivers, a non-positive limit means unbounded. The two failure modes are opposite in arithmetic and identical in effect.

**What this does NOT mean.** It does not mean every total must be exactly the union cardinality regardless of cost. Where a union dedups across mounts, an exact count needs the overlap, which needs every mount's paths — the O(corpus) work paging exists to avoid. The rule is: be exact whenever exactness is computable from what you already hold (no mount truncated → the deduped length IS the cardinality), and fall back to the summed per-mount COUNT(*) — a documented upper bound, off by exactly the cross-mount path collisions — only when it is not. Reporting the fetched size is never one of the options.

It also does not mean relevance endpoints owe a corpus count: a text query retrieves a bounded candidate set by rank, so the size of that set is the honest answer there and no COUNT(*) applies.
