---
type: observation
domain: []
confidence: 0.95
sources: 1
entities: [web/src/Library.tsx, loadMoreRef, loadingRef, lensLoadingRef, lensGenRef, useLayoutEffect, requested-offset guard, paging snapshot]
motifs: [state-split-across-phases, defence-in-depth-cost]
refs: ['src://7b4887ce51d9/web/src/Library.tsx@0a8d24d811af6e1a79a6637a0113d39139681635:a3b08cb6d4e9105dd712b3b03a9f236ca7e7b22a', 'https://github.com/knomit/knomit/pull/273', 'https://github.com/knomit/knomit/issues/270', 'kb://3ec012f5b4d2/kb/incidents/web/library/paging-duplicate-page/5866b86c.md', 'kb://3ec012f5b4d2/kb/gotchas/web/testing/library-loadmore-ref-passive-mirror/a965b6f6.md']
---
# PR #273 fixes Library's duplicate page fetch at its cause (layout-effect loadMoreRef, synchronous loadingRef) and deliberately does NOT add a requested-offset guard; a real hardening belongs to a single paging snapshot

CONTEXT. knomit#270: Library's sentinel could run a stale loadMore between a commit and React's passive flush and re-fetch the offset it had just loaded (kb/incidents/web/library/paging-duplicate-page). PR #273 fixed the cause. loadMoreRef is mirrored in useLayoutEffect, and the repo Recent branch now sets loadingRef.current = true synchronously, as the lens branch already did.

OPTIONS CONSIDERED.
1. Cause-only fix (chosen): the layout-effect mirror plus the synchronous loading-ref write in both branches, with commit-time regression tests.
2. Also add a requested-offset guard as defence in depth: remember the last offset requested and refuse any request at or below it.
3. Reviewer's alternative (#273 review): replace the separately mirrored paging state (render-phase loading refs, a layout-effect loadMore mirror, a passive scope-reset effect) with ONE paging snapshot, e.g. a ref holding {generation, nextOffset, inFlight, exhausted}, updated atomically where the request is issued and where its response lands.

RATIONALE.
- Option 2 fights retry semantics. A failed page fetch must stay retryable at the SAME offset, so the guard would need resetting on failure (the lens path's catch, and the repo path's catch, which has no generation token) AND on every scope reset: the lens union effect that bumps lensGenRef, and the repo path's own reset effect. That is four reset sites across two code paths, each a new way to wedge paging shut: a missed reset means a list that silently refuses to page, the very symptom #270's scope-change test caught. That is more surface than the defect warranted once the stale closure could no longer run.
- Option 3 is the principled hardening. The root weakness is paging state spread over three update phases, which is also why the repo Recent path has no generation guard: a page-2 response landing after a filter change appends old-scope rows (the follow-up issue). It is a redesign of Library's paging, not a flake fix, so it is out of #273's scope.

THE CHOICE. #273 ships option 1 only. The requested-offset guard is not added. Future hardening should go straight to option 3, a single paging snapshot, which subsumes the guard, the generation token and the loading flag.

NON-SCOPE. This does NOT say a duplicate fetch is acceptable, and it does NOT rule out option 3; it rules out bolting option 2 onto the current split state. It also does not claim the repo Recent path is scope-safe: it lacks lensGenRef's equivalent, which is tracked separately.
