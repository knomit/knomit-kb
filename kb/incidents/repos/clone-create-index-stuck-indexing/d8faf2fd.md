---
type: observation
domain: []
confidence: 0.95
sources: 1
entities: [Manager.Create, Manager.openOne, repoBuilder.build, repoBuilder.startSync, RepoInstance.ActivateSync, RepoInstance.shutdown, Manager.Close, SwapStore, indexCtx, indexWg, healIndexBranches]
refs: ['src://knomit/internal/repos/manager.go@21be1b8', 'src://knomit/internal/repos/builder.go@21be1b8', 'src://knomit/internal/repos/instance.go@21be1b8', 'src://knomit/internal/repos/lifecycle.go@21be1b8', 'src://knomit/internal/repos/swapstore.go@21be1b8', 'src://knomit/internal/repos/lifecycle_test.go@21be1b8']
---
# Runtime clone-create left the search index pinned at 'indexing' forever; ActivateSync killed the background heal

Symptom: creating a repo at runtime via Repo Manager -> Clone remote produced a repo whose UI showed '⟳ Indexing…' with no progress (branch endpoint: index_state=indexing, index_done=0, index_total=0, index_commit="") that never completed. Restarting the server fixed it.

Root cause: Manager.Create (mode=clone) calls m.Add -> openOne, which launches the heavy index heal in a background goroutine, then immediately calls ri.ActivateSync(origin). openOne's heal and the reconcile loop shared ONE context+waitgroup (syncCtx/syncWg). ActivateSync -> startSync does syncCancel()+syncWg.Wait() to (re)start the reconcile loop, cancelling the in-flight heal. The heal goroutine then hit `if <ctx>.Err() != nil { return }` and exited WITHOUT markIndexReady/markIndexFailed, pinning IndexStatus at 'indexing'. Restart masked it because Start/Rescan open repos via openOne ONLY (no inline ActivateSync), so the heal completed. Reliable because the heal's rebuild does real SQL/embed work while the cancel is instant (cancellation observed at 'rebuild: ensure facts_vec: context canceled', before the embedder was even reached).

Fix: gave the background heal its own lifecycle — indexCtx/indexWg on RepoInstance, derived in repoBuilder.build, cancelled only by real teardown (RepoInstance.shutdown, Manager.Close, SwapStore), each waiting indexWg BEFORE syncWg (the heal's activate() does syncWg.Add(1) for the loop). startSync and DeactivateSync touch only syncCtx/syncWg now, so ActivateSync no longer disturbs the heal. See decision kb/decisions/repos/index-heal-separate-lifecycle.

Regression test: internal/repos/lifecycle_test.go TestCreate_CloneMode_ActivateSyncDoesNotKillIndex — real clone-create against a file:// bare remote seeded with one fact; asserts IndexStatus reaches 'ready'. Red before fix (stuck 'indexing'), green after; race-clean.
