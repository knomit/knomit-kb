---
type: principle
domain: [store, resolver, git, graph]
confidence: 0.9
sources: 0
entities: [resolveTargetCommit, firstParentCommit, internal/store/derived_from.go, internal/store/search_index.go, rebuildGraph, branch_commits]
refs: ['src://knomit/.claude/plans/2026-05-01-topological-ordering-followup.md@0938d83']
---
# Edge target_commit resolution walks first-parent, NEVER orders by committed_at

resolveTargetCommit (and rebuildGraph Phase B's per-ref resolution) walks first-parent ancestry via (*repoHandler).firstParentCommit, NOT ordering by commit_log.committed_at DESC. Reason: wall-clock ordering breaks when a merge commit pulls in a foreign-branch revision of `path` whose committed_at is greater than the local branch's most recent change to `path`, but the merge resolved by taking the local side (StrategyLocalWins) — SQL would pick the foreign commit; first-parent walk correctly stays on the local-side history. Walk: start at sourceCommit, check commit_log for (current, refPath); on hit return (unless action='deleted'); otherwise descend to first parent. Stop at root, or when parent ∉ branch_commits(B). The storytest scenario 8_merge_with_local_wins_topological_resolution pins this. NOTE: rebuildGraph Phase A ordering (by MIN(committed_at) on path) is left as-is — outer write-order convenience, doesn't affect correctness since each ref is resolved independently by first-parent walk inside graphAddDerivedFromAtCommitTx.
