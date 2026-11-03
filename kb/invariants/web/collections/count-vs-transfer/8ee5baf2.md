---
kind: pragmatic
type: policy
domain: [web, api, lens, collections]
confidence: 0.95
sources: 1
entities: [internal/edge/web/handlers_lenses_read.go, internal/edge/web/handlers_facts_collection.go, internal/engine/store/index/search_query.go, RecentFacts, maxLensRecencyDepth, maxLensSearchCandidates, lensFanoutDepth, TestLensFanoutDepth_OverflowingOffsetStillClamps]
motifs: [measures-work-not-outcome]
refs: ['src://7b4887ce51d9/internal/edge/web/api/kb/handlers_lenses_read.go@db0f1daf9086a91e603cdfcadad4f6719c5aef5e:31b80935c93cb6f198fcac312c18a80cee0f642c', 'src://7b4887ce51d9/internal/edge/web/handlers_lenses_read_test.go@db0f1daf9086a91e603cdfcadad4f6719c5aef5e:c76e15c4c99cde4beae3693cd03954fab969f28d', 'src://7b4887ce51d9/internal/engine/store/index/search_query.go@db0f1daf9086a91e603cdfcadad4f6719c5aef5e:f04234d895ac4290c8ceaf28760afbfd1768c77c', 'src://7b4887ce51d9/internal/edge/web/api/kb/handlers_facts_collection.go@db0f1daf9086a91e603cdfcadad4f6719c5aef5e:fc9ae5c74fcb89ea30be8cd6da5d6f7b0b07cf0a', 'src://7b4887ce51d9/internal/edge/web/api/kb/handlers_search_hal.go@db0f1daf9086a91e603cdfcadad4f6719c5aef5e:74206b328785c5d21784ff3e9edc7f59c884e022', 'https://github.com/knomit/knomit/pull/735']
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

SCOPE NOTE (PR #735): GET /lenses/{lens}/search is a RELEVANCE list. No COUNT(*) of "rows that match" exists for it: per-mount depth (federate.MaxSearchCandidates) already bounds what can match. Its `total` is therefore defined as the length of the ranked list that offset pages through: the fused, deduped union cut to materialize.MaxFactSetFacts. Every offset below it has a row, and the lens FactSet of the same search has exactly that many units. Before #735 it was the uncut union size. This rule still governs the /facts collections, whose total is a real match count.

The repo route GET /repos/{repo}/branches/{branch}/search carries the same `total` since the #735 review: index.SearchPage.Total, the depth-500 ranked list's length. The web shows it, and the lens total, as the relevance list's count (library-count / data-count). Neither is a COUNT(*) of matches.
