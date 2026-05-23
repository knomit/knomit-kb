---
type: principle
domain: [store, sync, concurrency]
confidence: 0.9
sources: 0
entities: [last_commit, casLastCommit, meta table, BEGIN IMMEDIATE, _busy_timeout, lockBranch]
refs: ['src://knomit/.claude/plans/2026-03-30-context-tx-concurrency-design.md@0938d83', 'src://knomit/internal/store/branch.go@77f70cf']
---
# Sync uses CAS on last_commit watermark; per-branch Sync mutex removed

Concurrent Sync calls do NOT use a per-branch mutex for WATERMARK DEDUP. Instead they use compare-and-swap on the last_commit watermark stored in `meta`: 'UPDATE meta SET value = ? WHERE key = ? AND value = ?' returns rowsAffected; losers see 0 rows and discard their work. Safe because facts are already upserted via COW dedup (idempotent). Per-file Upsert/Delete get their own implicit transaction — Sync does NOT wrap many files in one giant transaction (would hold the write lock too long and hurt concurrency). SQLite's BEGIN IMMEDIATE provides write-serialization at the storage layer; other writers block via _busy_timeout (default 5000ms). CORRECTION (earlier version of this fact was wrong): the per-branch RWMutex map `repoHandler.branchMu` (driving lockBranch / lockBranchRead in internal/store/branch.go) STILL EXISTS — it has NOT been removed. It serializes git-ref WRITES through notifyCommit so readers (including a concurrent Verify) never see torn state between SetReference and the index sync. The two mechanisms address different concerns: branchMu protects torn-state on git+SQL writes (see [[notify-commit-single-chokepoint]] and [[branch-lock-spans-notify]]); the CAS watermark dedups concurrent Sync ticks that would otherwise re-walk the same commits.
