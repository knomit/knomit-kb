---
kind: pragmatic
type: policy
domain: [repos, store, index]
confidence: 0.95
sources: 0
entities: [repoBuilder.setupIndex, InitFromRemote, IndexManager.Sync, upstreamMain]
refs: ['src://knomit/internal/repos/builder.go@307b67d']
---
# setupIndex MUST sync BOTH agentBranch AND upstreamMain when origin is configured

When cfg.Git.Origin is configured, repoBuilder.setupIndex MUST run svc.IndexManager().Sync(ctx, upstreamMain) IN ADDITION to syncing the agent branch. Reason: InitFromRemote populates commit_log for both agent/* AND the upstream during clone, but without an explicit index sync the upstream's branch_facts / facts_vec / graph tables would be EMPTY even though the cloned tree has content. Without this, Verify's facts-coherence check correctly fires on the upstream branch whenever the cloned tree has any facts. Anti-pattern: assuming clone-time commit_log population implies index population — clone only does git-side work; the SQL index needs an explicit sync per branch. Same applies to any future code path that calls InitFromRemote or otherwise populates commit_log without going through notifyCommit.
