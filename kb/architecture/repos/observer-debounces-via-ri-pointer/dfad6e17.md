---
type: observation
domain: [repos, store, observer]
confidence: 0.95
sources: 0
entities: [observe.New, ri.onCommit, obs.Notify, broadcastStatus]
motifs: [handle-outlives-target]
refs: ['src://knomit/internal/repos/builder.go@307b67d', 'src://7b4887ce51d9/internal/corpus/repos/observer.go@0610c20083fcef9e84f42ab0311a14708bcb6193:4085c1a820be8fa3b2749150e4a089abd5563cd0', 'src://7b4887ce51d9/internal/corpus/repos/stages.go@0610c20083fcef9e84f42ab0311a14708bcb6193:a8de1ef38d51cf564edee4a5e8260adee574176c', 'src://7b4887ce51d9/internal/corpus/repos/manager.go@0610c20083fcef9e84f42ab0311a14708bcb6193:85952799dd62df86b0def2737f81bbe252e42cf0']
---
# Commit observer debounces 1s; its callback Acquires the current store each time it fires, so it follows SwapStore

The commit observer is installed by the Open stage (stages.go): r.observer.Store(newCommitObserver(time.Second, callback)), where commitObserver (internal/corpus/repos/observer.go) is the repos package's own unexported debouncer. The callback holds NO store pointer: each time it fires it calls r.Acquire() for the CURRENT store, then IndexManager().SyncLocked(ctx, r.ReadBranch()) and hub.broadcastStatus(hash), so after a SwapStore it automatically uses the new store. ri.onCommit (built once with the instance in manager.go) calls ri.observer.Load().Notify(hash), and the Open stage re-applies it to every (re)opened store with svc.SetOnCommit(r.onCommit). Debounce window is 1 second — collapses rapid writes (e.g. synthesize pipeline doing 20 deletes) into a single index sync + SSE push; a notification that arrives while the callback runs is deferred, not dropped. Anti-pattern: capturing a *store.Service in the observer closure — it would become a stale pointer after the first SwapStore.
