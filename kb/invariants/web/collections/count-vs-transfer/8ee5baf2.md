---
kind: pragmatic
type: policy
domain: [web, api, lens, collections]
confidence: 0.95
sources: 1
entities: [internal/edge/web/handlers_lenses_read.go, internal/edge/web/handlers_facts_collection.go, internal/engine/store/index/search_query.go, RecentFacts, maxLensRecencyDepth, maxLensSearchCandidates, lensFanoutDepth, TestLensFanoutDepth_OverflowingOffsetStillClamps]
motifs: [measures-work-not-outcome]
refs: ['src://7b4887ce51d9/internal/edge/web/handlers_lenses_read.go@0610c20083fcef9e84f42ab0311a14708bcb6193:ae5d1a0be5f1640583d34af9997534bc5740da29', 'src://7b4887ce51d9/internal/edge/web/handlers_lenses_read_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:12b8bd312e6d8b40ed2673ed544008f023aa8c89', 'src://7b4887ce51d9/internal/engine/store/index/search_query.go@0610c20083fcef9e84f42ab0311a14708bcb6193:5a20922f4876029a0b68e4df70d95485bde0b170', 'src://7b4887ce51d9/internal/edge/web/handlers_facts_collection.go@0610c20083fcef9e84f42ab0311a14708bcb6193:d4f341a2c8269a134afe0992d8a9279dd0842a37']
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
