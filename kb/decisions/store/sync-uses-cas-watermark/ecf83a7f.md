---
type: principle
domain: [store, sync, concurrency]
confidence: 0.9
sources: 0
entities: [last_commit, casLastCommit, meta table, BEGIN IMMEDIATE, _busy_timeout, lockBranch]
refs: ['src://knomit/.claude/plans/2026-03-30-context-tx-concurrency-design.md@0938d83']
---
# Sync uses CAS on last_commit watermark; per-branch Sync mutex removed

Concurrent Sync calls do NOT use a per-branch mutex. Instead they use compare-and-swap on the last_commit watermark: 'UPDATE meta SET value = ? WHERE key = ? AND value = ?' returns rowsAffected; losers see 0 rows and discard their work. Safe because facts are already upserted via COW dedup (idempotent). Per-file Upsert/Delete get their own implicit transaction — Sync does NOT wrap many files in one giant transaction (would hold the write lock too long and hurt concurrency). The CAS watermark prevents duplicate work without a long-held lock. SQLite's BEGIN IMMEDIATE provides write-serialization at the storage layer; other writers block on BeginTx via _busy_timeout (default 5000ms). The old per-branch git lockBranch sync.Map was REMOVED — SQLite tx serialization provides the same guarantee.
