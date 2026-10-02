---
type: observation
domain: [store, paths, git, index]
confidence: 0.85
sources: 1
entities: [deriveBeforeAdvance, notifyCommit, im.Sync, SetReference, ValidTreePath, DiffTree, ListAllWithHash]
motifs: [check-separated-from-use]
refs: ['src://7b4887ce51d9/internal/store/fact_write.go@b95754cc557bd718b0cc3900f560bdd57efe0dce:4c6433f82ad0ca83e950d1941f445bb72519a3a6', 'src://7b4887ce51d9/internal/store/path_changes.go@b95754cc557bd718b0cc3900f560bdd57efe0dce:ec4ccda479de308a962b16e87a47152c9ff5d6a7', 'src://7b4887ce51d9/internal/store/branch_commit.go@b95754cc557bd718b0cc3900f560bdd57efe0dce:c01e5c18a37c99095c7669c4aac3b407db011c44', 'src://7b4887ce51d9/internal/store/fact_write_invalid_path_test.go@b95754cc557bd718b0cc3900f560bdd57efe0dce:d8c3caa121913cf0d2f73dc302960dc1dd572266', 'kb://3ec012f5b4d2/kb/invariants/store/path-changes/never-fails-a-write/eb72351e.md', 'https://github.com/knomit/knomit/issues/384']
---
# deriveBeforeAdvance refuses a malformed tree (empty entry name) before the ref moves, but NOT a name go-git refuses to read back (control character, git~1, .git disguise); only im.Sync, after SetReference, reaches go-git's ValidTreePath

The store's write sequence is: build tree → storeCommit → deriveBeforeAdvance → SetReference → notifyCommit/im.Sync. The derivation step before the ref moves decodes the new trees, so a tree go-git cannot DECODE fails there and the ref stays put. Empty segments (`kb//x.md`, `/kb/x.md`, `kb/x/`) are refused this way with `path changes: tree <hash>: malformed tree: empty filename`, and the head does not move. A NUL byte fails even earlier, at tree encode (`malformed filename`).

A tree that decodes fine but holds a name go-git's ValidTreePath rejects passes derivation. Examples: a line feed, tab or DEL; `git~1`; a `\.` component; a backslash-only name; a zero-width-joined `.git`. ValidTreePath runs only on path-level READS (FindEntry, TreeEntryFile, the tree walker), which first happen in im.Sync inside notifyCommit, AFTER SetReference. That is the gap #384 fell through.

Consequence: do not treat deriveBeforeAdvance as a pre-commit validity gate for paths. Path rules must be enforced before the commit is built (store.validatePath), or they fire after the ref has moved. This is also why validatePath can leave empty segments out of scope without risking a poisoned branch, at the cost of a late 500 (TestWriteFact_EmptySegmentRefusedBeforeRefMoves pins it).

Relation to kb/invariants/store/path-changes/never-fails-a-write/eb72351e.md: that invariant's claim, 'a write never ends with its ref moved and its call failed', covers HISTORY DERIVATION only. notifyCommit's im.Sync still runs after the ref moves, and #384 was exactly a write that ended with the ref moved and the call failed. Do not read that invariant as covering the index sync.

Derivation runs only when commit_log is available. A refusal there leaves the built blob, tree and commit objects as loose garbage in the object store.
