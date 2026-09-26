---
type: observation
domain: [web, library, decisions]
confidence: 0.95
sources: 1
entities: [web/src/Library.tsx, loadMoreRef, loadingRef, lensLoadingRef, lensGenRef, recentGenRef, useLayoutEffect, requested-offset guard, paging snapshot]
motifs: [state-split-across-phases, defence-in-depth-cost]
refs: ['kb://3ec012f5b4d2/kb/gotchas/web/testing/library-loadmore-ref-passive-mirror/a965b6f6.md', 'src://7b4887ce51d9/web/src/Library.tsx@ec38e06d0c068538f2b52f3f82a61b11d9c9b6eb:a3b08cb6d4e9105dd712b3b03a9f236ca7e7b22a', 'src://7b4887ce51d9/web/src/Library.tsx@4b8962b8b342b8d97e2ac2c387d9965ad780ebd8:18bb8ccfaab1b7b4e3a919cc08e811ca15c17008#L417-L438', 'https://github.com/knomit/knomit/pull/273', 'https://github.com/knomit/knomit/issues/270', 'https://github.com/knomit/knomit/issues/275', 'https://github.com/knomit/knomit/pull/306']
---
# PR #273 fixes Library's duplicate page fetch at its cause (layout-effect loadMoreRef, synchronous loadingRef) and deliberately does NOT add a requested-offset guard; a real hardening belongs to a single paging snapshot

CONTEXT. knomit#270: Library's sentinel could run a stale loadMore between a commit and React's passive flush and re-fetch the offset it had just loaded (kb/gotchas/web/testing/library-loadmore-ref-passive-mirror). PR #273 fixed the cause. loadMoreRef is mirrored in useLayoutEffect, and the repo Recent branch now sets loadingRef.current = true synchronously, as the lens branch already did.

OPTIONS CONSIDERED.
1. Cause-only fix (chosen): the layout-effect mirror plus the synchronous loading-ref write in both branches, with commit-time regression tests.
2. Also add a requested-offset guard as defence in depth: remember the last offset requested and refuse any request at or below it.
3. Reviewer's alternative (#273 review): replace the separately mirrored paging state (render-phase loading refs, a layout-effect loadMore mirror, a passive scope-reset effect) with ONE paging snapshot, e.g. a ref holding {generation, nextOffset, inFlight, exhausted}, updated atomically where the request is issued and where its response lands.

RATIONALE.
- Option 2 fights retry semantics. A failed page fetch must stay retryable at the SAME offset, so the guard would need resetting on failure (the lens path's catch, and the repo path's catch) AND on every scope reset: the lens union effect that bumps lensGenRef, and the repo path's own reset effect. That is four reset sites across two code paths, each a new way to wedge paging shut: a missed reset means a list that silently refuses to page, the very symptom #270's scope-change test caught. That is more surface than the defect warranted once the stale closure could no longer run.
- Option 3 is the principled hardening. The root weakness is paging state spread over three update phases, which is also why the repo Recent path had no generation guard at #273: a page-2 response landing after a filter change appended old-scope rows (knomit#275). It is a redesign of Library's paging, not a flake fix, so it is out of #273's scope.

THE CHOICE. #273 ships option 1 only. The requested-offset guard is not added. Future hardening should go straight to option 3, a single paging snapshot, which subsumes the guard, the generation token and the loading flag.

AFTER #273. knomit#275 closed the scope window on the repo Recent path with the minimal fix, not option 3: `recentGenRef`, a generation token that mirrors `lensGenRef` (bumped by the Recent scope-reset effect, snapshotted in loadMore, checked in both its then and its catch). Both Library branches now drop a page requested for a previous scope. Option 3, the single paging snapshot, remains unbuilt and has no ticket; the user has not asked for it.

NON-SCOPE. This does NOT say a duplicate fetch is acceptable, and it does NOT rule out option 3; it rules out bolting option 2 onto the current split state. The two generation tokens are not option 3 either: paging state is still split across render-phase refs, a layout-effect mirror and passive scope-reset effects, and each token closes one window between them rather than removing the split.
