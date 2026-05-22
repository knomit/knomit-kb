---
type: principle
domain: [store, git, commits]
confidence: 0.95
sources: 0
entities: [notifyCommit, AppendCommitLog, im.Sync, writeFile, deleteFile, batchWrite, mergeIntoBranch, reconcileAgentMerge]
refs: ['src://knomit/internal/store/branch_commit.go@307b67d', 'src://knomit/internal/store/fact_write.go@307b67d', 'src://knomit/internal/store/branch_merge.go@307b67d']
---
# Every branch-ref mutation MUST call repoHandler.notifyCommit while holding the branch lock

Every mutation that advances a branch ref MUST call repoHandler.notifyCommit(ctx, branch, hash) while still holding the per-branch write lock. notifyCommit does three things in order: (1) AppendCommitLog(ctx, branch, hash) — branch-agnostic commit_log row + branch_commits visibility row; (2) im.Sync(ctx, branch) — brings branch_facts / facts_vec / graph in sync with the new tree; (3) onCommit(branch, hash) external observer (e.g. SSE). Mutation paths that call it: factIndex.writeFile / deleteFile / batchWrite, branch_merge.mergeIntoBranch (FF and 3-way merge), remote_reconcile_merge.reconcileAgentMerge, remote_reconcile_rebase.replayOntoUpstream. Skipping notifyCommit leaves SQL stale relative to git — trips Verify's facts-coherence check. Anti-pattern: advancing a ref via storer.SetReference without immediately calling notifyCommit.
