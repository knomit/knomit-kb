---
type: observation
domain: [store, sqlite, performance, testing]
confidence: 0.85
sources: 1
entities: [wal_autocheckpoint, PRAGMA wal_checkpoint, rebuildCommitLog, WriteFact, lockBranch, _journal_mode=WAL, internal/store/branch_commit_log.go, knomit#366, knomit#403]
motifs: [next-caller-pays, timing-measures-environment]
refs: ['src://7b4887ce51d9/internal/store/service.go@f42ebf2f50dc2df46a77f7509e8c41d68bb43817:9a229982458f0f86afb9c35856bba60a8eae5f07', 'src://7b4887ce51d9/internal/store/path_history_index_test.go@f42ebf2f50dc2df46a77f7509e8c41d68bb43817:70ee11d9569b512c27598348ff14517ab9a1e301#L264-L316', 'https://github.com/knomit/knomit/issues/366', 'https://github.com/knomit/knomit/issues/403', 'https://github.com/knomit/knomit/actions/runs/36863649277']
---
# After a bulk write such as rebuildCommitLog, the FIRST later commit pays SQLite's automatic WAL checkpoint of the whole backlog (7-20 MB after a 2000-commit re-derive): 0.2-0.6 s idle, 6-11 s on a loaded Windows disk, inside whichever COMMIT comes first

Store DBs run WAL (service.go:105) with SQLite's default wal_autocheckpoint (1000 pages). A bulk operation that writes many batches, measured on rebuildCommitLog re-deriving 2000 commits, leaves a 7-20 MB WAL. The next COMMIT on that connection pool, whatever statement it is (Tx.Commit in upsert/deriveCommits, SetReference, SetEncodedObject, PRAGMA optimize at search_index.go:415), runs the passive auto-checkpoint: it copies the backlog into the main file and syncs it. On a store write that happens while the writer holds its lockBranch.

Measured during #366 (Windows 10, instrumented worktree):
- idle: the commit after the rebuild ran 0.2-0.6 s past the rebuild's end;
- with 3 goroutines doing write+fsync churn: 6-11 s, once 14.76 - 3.88 s;
- same load with wal_autocheckpoint=0: the stall vanished (worst write = rebuild + about 20 ms).
On CI (run 36863649277, windows-2025 store leg) a write logged 6.8 s while the rebuild took 3.06 s.

Consequences:
(1) A test that times individual writes around a bulk operation measures disk speed, not locking.
(2) In production, the first user write after a bulk operation can be slow on a slow disk. #403 tracks having bulk operations checkpoint their own WAL.

Not meant: this is not lock contention with the bulk operation. The operation has already returned, and none of its transactions are open.
