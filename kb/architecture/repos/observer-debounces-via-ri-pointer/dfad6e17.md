---
type: observation
domain: [repos, store, observer]
confidence: 0.95
sources: 0
entities: [observe.New, ri.onCommit, obs.Notify, broadcastStatus]
motifs: [handle-outlives-target]
refs: ['src://knomit/internal/repos/builder.go@307b67d']
---
# Commit observer debounces 1s; callback re-reads ri.svc under RLock so it follows SwapStore

The commit observer is wired in repoBuilder.build(): obs := observe.New(time.Second, callback). The callback re-reads ri.svc under the read lock and calls currentSvc.IndexManager().Sync + hub.broadcastStatus(hash). CRITICAL: the callback captures the ri POINTER, NOT a field copy of svc — so when SwapStore replaces ri.svc, the observer automatically follows. ri.onCommit = func(_, hash string) { obs.Notify(hash) } is stored on the instance so SwapStore can re-apply it to the new svc via svc.SetOnCommit(ri.onCommit). Debounce window is 1 second — collapses rapid writes (e.g. synthesize pipeline doing 20 deletes) into a single index sync + SSE push. Anti-pattern: capturing currentSvc in the observer closure — it would become a stale pointer after the first SwapStore.
