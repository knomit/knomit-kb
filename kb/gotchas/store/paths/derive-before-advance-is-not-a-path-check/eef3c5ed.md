---
type: observation
domain: [store, paths, git, index]
confidence: 0.85
sources: 1
entities: [deriveBeforeAdvance, notifyCommit, im.Sync, SetReference, ValidTreePath, DiffTree, ListAllWithHash]
motifs: [check-separated-from-use]
refs: ['src://7b4887ce51d9/internal/engine/store/facts/fact_write.go@0610c20083fcef9e84f42ab0311a14708bcb6193:41d2f4f71bffe86dd1fae5d96eaacaa774674bb5', 'src://7b4887ce51d9/internal/engine/store/internal/vcs/path_changes.go@0610c20083fcef9e84f42ab0311a14708bcb6193:ea28a0ab2e86a722f3525971ec0c2a81159d90c6', 'src://7b4887ce51d9/internal/engine/store/internal/vcs/branch_commit.go@0610c20083fcef9e84f42ab0311a14708bcb6193:6941daef90e8275e2809018b55aca733dafa1e70', 'src://7b4887ce51d9/internal/engine/store/fact_write_invalid_path_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:f274c41c7ad3a3275b9cbf0b3d14e93ee3fae00e', 'kb://3ec012f5b4d2/kb/invariants/store/path-changes/never-fails-a-write/eb72351e.md', 'kb://3ec012f5b4d2/kb/invariants/store/verify/tree-shape/fc07357f.md', 'kb://3ec012f5b4d2/kb/gotchas/store/verify/malformed-main-recovery/0cbaf27d.md', 'https://github.com/knomit/knomit/issues/384']
---
# deriveBeforeAdvance refuses a malformed tree (empty entry name) before the ref moves, but NOT a name go-git refuses to read back (control character, git~1, .git disguise); only im.Sync, after SetReference, reaches go-git's ValidTreePath

The store's write sequence is: build tree → storeCommit → deriveBeforeAdvance → SetReference → notifyCommit/im.Sync. The derivation step before the ref moves decodes the new trees, so a tree go-git cannot DECODE fails there and the ref stays put. Empty segments (`kb//x.md`, `/kb/x.md`, `kb/x/`) are refused this way with `path changes: tree <hash>: malformed tree: empty filename`, and the head does not move. A NUL byte fails even earlier, at tree encode (`malformed filename`).

A tree that decodes fine but holds a name go-git's ValidTreePath rejects passes derivation. Examples: a line feed, tab or DEL; `git~1`; a `\.` component; a backslash-only name; a zero-width-joined `.git`. ValidTreePath runs only on path-level READS (FindEntry, TreeEntryFile, the tree walker), which first happen in im.Sync inside notifyCommit, AFTER SetReference. That is the gap #384 fell through.

Consequence: do not treat deriveBeforeAdvance as a pre-commit validity gate for paths. Path rules must be enforced before the commit is built (store.validatePath), or they fire after the ref has moved. This is also why validatePath can leave empty segments out of scope without risking a poisoned branch, at the cost of a late 500 (TestWriteFact_EmptySegmentRefusedBeforeRefMoves pins it).

Relation to kb/invariants/store/path-changes/never-fails-a-write/eb72351e.md: that invariant's claim, 'a write never ends with its ref moved and its call failed', covers HISTORY DERIVATION only. notifyCommit's im.Sync still runs after the ref moves, and #384 was exactly a write that ended with the ref moved and the call failed. Do not read that invariant as covering the index sync.

Derivation runs only when commit_log is available. A refusal there leaves the built blob, tree and commit objects as loose garbage in the object store.

UPDATE (PR #523, verified at merge commit 3ac59b57): the same 'ref moved, call failed' shape (#384) arises for a MALFORMED TREE (a Dir entry naming a blob or a missing object): populateCommitLog cannot diff it ('a tree is missing: object not found'), and notifyCommit runs it AFTER SetReference. Before #523 a receive-pack push of such a tree was admitted by the gate, moved the branch ref, and then answered 'could not be registered: populate commit_log after fast-forward'. The gate now refuses new malformed trees (kb/invariants/store/verify/tree-shape/fc07357f.md), so this route is closed for pushes; but a malformed commit already on a branch still makes every later write on it move the ref and then error (kb/gotchas/store/verify/malformed-main-recovery/0cbaf27d.md). MISREADING: 'deriveBeforeAdvance guards the tree' stays false: it checks an empty entry name only.
