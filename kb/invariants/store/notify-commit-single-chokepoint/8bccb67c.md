---
type: principle
domain: [store, git, commits]
confidence: 0.95
sources: 0
entities: [NotifyCommit, notifyCommit, vcs.Repo, AppendCommitLog, im.Sync, touchExperimentActivity, writeFile, deleteFile, batchWrite, mergeIntoBranch, reconcileAgentMerge]
motifs: [bypass-defeats-guarantee]
refs: ['src://knomit/internal/store/branch_commit.go@307b67d', 'src://knomit/internal/store/fact_write.go@307b67d', 'src://knomit/internal/store/branch_merge.go@307b67d', 'src://7b4887ce51d9/internal/store/internal/vcs/branch_commit.go@a2adfe0876473cda822cce2f3b9c2ccfa3774987:ed1528c7cd165e693d157ae7f97b1cf8e5d244de', 'src://7b4887ce51d9/internal/store/fact_write.go@a2adfe0876473cda822cce2f3b9c2ccfa3774987:36b02269f140db54872f7c16fd86b1fb638694aa']
---
# Every branch-ref mutation MUST call repoHandler.notifyCommit while holding the branch lock

Every mutation that advances a branch ref MUST call NotifyCommit(ctx, branch, hash) — vcs.Repo.NotifyCommit in internal/store/internal/vcs/branch_commit.go since hier-07, reached from the parent as rh.NotifyCommit through the embedded *vcs.Repo (formerly repoHandler.notifyCommit) — while still holding the per-branch write lock. NotifyCommit does four things in order: (1) AppendCommitLog(ctx, branch, hash) — branch-agnostic commit_log row + branch_commits visibility row; (2) im.Sync(ctx, branch) — brings branch_facts / facts_vec / graph in sync with the new tree (skipped when no index manager is set); (3) touchExperimentActivity(ctx, branch); (4) onCommit(branch, hash) external observer (e.g. SSE). The index manager and onCommit are injected by the parent as values (SetIndexManager, SetOnCommit); the child never calls up. Mutation paths that call it: factIndex.writeFile / deleteFile / batchWrite, branch_merge.mergeIntoBranch (FF and 3-way merge), remote_reconcile_merge.reconcileAgentMerge, remote_reconcile_rebase.replayOntoUpstream. Skipping notifyCommit leaves SQL stale relative to git — trips Verify's facts-coherence check. Anti-pattern: advancing a ref via storer.SetReference without immediately calling notifyCommit.
