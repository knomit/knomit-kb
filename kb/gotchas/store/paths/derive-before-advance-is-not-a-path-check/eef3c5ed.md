---
type: observation
domain: [store, paths, git, index]
confidence: 0.85
sources: 1
entities: [deriveBeforeAdvance, notifyCommit, im.Sync, SetReference, ValidTreePath, DiffTree, ListAllWithHash]
motifs: [check-separated-from-use]
refs: ['src://7b4887ce51d9/internal/store/fact_write.go@1e437c7f3326f0599816203d578d251322b1c678:e36554124999bb8f2f1d4b8de8b1a825e294b87b', 'src://7b4887ce51d9/internal/store/path_changes.go@1e437c7f3326f0599816203d578d251322b1c678:ec4ccda479de308a962b16e87a47152c9ff5d6a7', 'src://7b4887ce51d9/internal/store/branch_commit.go@1e437c7f3326f0599816203d578d251322b1c678:c01e5c18a37c99095c7669c4aac3b407db011c44', 'kb://3ec012f5b4d2/kb/invariants/store/path-changes/never-fails-a-write/eb72351e.md', 'https://github.com/knomit/knomit/issues/384']
---
# deriveBeforeAdvance refuses a malformed tree (empty entry name) before the ref moves, but NOT a control-character name — only im.Sync, after SetReference, reaches go-git's ValidTreePath

In the store's write sequence (build tree → storeCommit → deriveBeforeAdvance → SetReference → notifyCommit/im.Sync), the derivation step before the ref moves decodes the new trees, so a tree go-git cannot DECODE fails there and the ref stays put: learn with `category: "x//y"` was refused with `path changes: tree <hash>: malformed tree: empty filename` and the head did not move (observed at origin/dev c6c2625b). A NUL byte fails even earlier, at tree encode (`malformed filename`).

A tree that decodes fine but holds a name go-git's ValidTreePath rejects — a line feed, tab, DEL — passes derivation. ValidTreePath runs only on path-level READS (FindEntry, the tree walker, DiffTree), which first happen in im.Sync inside notifyCommit, AFTER SetReference. That is the gap #384 fell through.

Consequence: do not treat deriveBeforeAdvance as a pre-commit validity gate for paths. Path rules must be enforced before the commit is built (store.validatePath), or they fire after the ref has moved.

Relation to kb/invariants/store/path-changes/never-fails-a-write/eb72351e.md: that invariant's 'a write never ends with its ref moved and its call failed' is about HISTORY DERIVATION only. notifyCommit's im.Sync still runs after the ref moves, and #384 was exactly a write that ended ref-moved and call-failed. Do not read that invariant as covering the index sync.

Also: derivation runs only when commit_log is available, and a refusal there still leaves the built blob/tree/commit objects as loose garbage in the object store.
