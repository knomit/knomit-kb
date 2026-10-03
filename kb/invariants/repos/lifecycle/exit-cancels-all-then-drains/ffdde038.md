---
kind: pragmatic
type: policy
domain: [repos, lifecycle, concurrency, index, invariant]
confidence: 0.95
sources: 1
entities: [Machine.exitStages, Life.cancel, lockBranch, healIndexBranches, runReconcileLoop, TestWalkthrough_ArchiveMidIndexWithAParkedSyncRound]
motifs: [cancel-before-wait, lock-not-ctx-aware]
refs: ['src://7b4887ce51d9/internal/repos/machine.go@c619a91b101684b5b8ef0c6e86650f04d0f112f5:62c0ad5df41d1fc2f5991dff0d070ef64ba0e82a', 'src://7b4887ce51d9/internal/store/branch.go@c619a91b101684b5b8ef0c6e86650f04d0f112f5:e87aa9bc7848d55f11c6f6e3e7ef1b0d8dad4150', 'src://7b4887ce51d9/internal/repos/machine_test.go@c619a91b101684b5b8ef0c6e86650f04d0f112f5:62372ac202fccae6dc571e5bca2e64a9d6b238a9', 'kb://3ec012f5b4d2/kb/invariants/store/index/branch-locking/f4911495.md']
---
# When several stages exit, Machine.exitStages cancels EVERY affected life first and only then drains and Exits them newest-first — draining one before cancelling the index job deadlocks on lockBranch, which is not ctx-aware

A rewind or Unmount exits Sync, Serve, Index (and foundations) together. A sync round can be parked on lockBranch(upstream) while the index job holds it for a whole branch rebuild; lockBranch is a plain sync.RWMutex (internal/store/branch.go), so cancelling the round's ctx does not release it. Only the holder returning does — at its next batch boundary, once the INDEX life is cancelled. Draining Sync (newest) before cancelling Index would wait forever.

So exitStages loops twice: first l.cancel() on every life in the set, then, newest-first, l.wg.Wait(), stage Exit, lives[k]=nil. Newest-first keeps Exit order the reverse of Enter (Sync before Open's store close). Regression test: TestWalkthrough_ArchiveMidIndexWithAParkedSyncRound holds the index job on the branch lock with a sync round parked behind it and requires the Unmount to complete.

Any new multi-stage unwind must go through exitStages; do not hand-roll cancel+wait per stage.
