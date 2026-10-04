---
kind: pragmatic
type: policy
domain: [repos, lifecycle, concurrency, index, invariant]
confidence: 0.95
sources: 1
entities: [Machine.exitStages, Life.cancel, lockBranch, healIndexBranches, runReconcileLoop, TestWalkthrough_ArchiveMidIndexWithAParkedSyncRound]
motifs: [cancel-before-wait, lock-not-ctx-aware]
refs: ['src://7b4887ce51d9/internal/repos/machine.go@12a3bb8901b6f78ca19fbc37f99beeb9a243f667:6abc9980c7e763ad4c0a5dbf95737ea48067d273', 'src://7b4887ce51d9/internal/store/branch.go@12a3bb8901b6f78ca19fbc37f99beeb9a243f667:5f6030bd8d84a6d8e50e4adc4f2c77d71212f0b4', 'src://7b4887ce51d9/internal/repos/machine_test.go@12a3bb8901b6f78ca19fbc37f99beeb9a243f667:c084328c756937c2124a5c4711c0cef3b5919a4f', 'kb://3ec012f5b4d2/kb/invariants/store/index/branch-locking/f4911495.md', 'https://github.com/knomit/knomit/pull/414']
---
# When several stages exit, Machine.exitStages cancels EVERY affected life first and only then drains and Exits them newest-first — draining one before cancelling the index job deadlocks on lockBranch, which is not ctx-aware

A rewind or Unmount exits Sync, Serve, Index (and foundations) together. A sync round can be parked on lockBranch(upstream) while the index job holds it for a whole branch rebuild; lockBranch is a plain sync.RWMutex (internal/store/branch.go), so cancelling the round's ctx does not release it. Only the holder returning does — at its next batch boundary, once the INDEX life is cancelled. Draining Sync (newest) before cancelling Index would wait forever.

So exitStages loops twice: first l.cancel() on every life in the set, then, newest-first, l.wg.Wait(), stage Exit, lives[k]=nil. Newest-first keeps Exit order the reverse of Enter (Sync before Open's store close). Regression test: TestWalkthrough_ArchiveMidIndexWithAParkedSyncRound holds the index job on the branch lock with a sync round parked behind it and requires the Unmount to complete.

Any new multi-stage unwind must go through exitStages; do not hand-roll cancel+wait per stage.
