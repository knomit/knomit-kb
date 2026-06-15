---
kind: pragmatic
type: policy
domain: [repos, store, index]
confidence: 0.95
sources: 0
entities: [repoBuilder.setupIndex, InitFromRemote, IndexManager.Sync, upstreamMain]
refs: ['src://knomit/internal/repos/builder.go@307b67d', 'src://knomit/internal/repos/builder.go', 'src://knomit/internal/repos/manager.go', 'https://github.com/knomit/knomit/pull/82']
---
# setupIndex MUST sync BOTH agentBranch AND upstreamMain when origin is configured

When cfg.Git.Origin is configured, the initial index heal MUST sync BOTH the agent branch AND upstreamMain. Reason: InitFromRemote populates commit_log for both agent/* AND the upstream during clone, but without an explicit index sync the upstream's branch_facts / facts_vec / graph tables would be EMPTY even though the cloned tree has content — and Verify's facts-coherence check correctly fires on the upstream branch. Anti-pattern: assuming clone-time commit_log population implies index population; clone only does git-side work, the SQL index needs an explicit sync per branch.

MECHANISM (PR #82): this no longer happens synchronously inside repoBuilder.setupIndex. setupIndex now only RECORDS which branches need healing (b.indexBranches) and whether the schema is stale (b.indexStale). Manager.openOne then runs healIndexBranches(agentBranch + upstreamMain) in a BACKGROUND goroutine (watching b.syncCtx) so the HTTP server / web UI come up IMMEDIATELY instead of blocking ListenAndServe on a heavy rebuild. repoBuilder.build() constructs the RepoInstance but DEFERS recoverFromOrigin + startSyncLoops to b.activate(), which the background goroutine calls only AFTER the heal completes — so the reconcile/commit loops never race the initial index build. RepoInstance.IndexStatus() reports ready|indexing|error+done/total; the web UI shows an indexing banner and reads return partial results until ready. DisableBackgroundSync (test harnesses) takes the SYNCHRONOUS path (heal + activate inline) so the open->index-ready contract still holds. The both-branches invariant is now enforced inside healIndexBranches; any future path that calls InitFromRemote must route through it (or notifyCommit) so neither branch is left unindexed.
