---
type: observation
domain: [store, verify, git-serve, fleet]
confidence: 0.85
sources: 1
entities: [commitTreeShape, factIndex.deleteFile, deletePath, removeEntry, fileExists, populateCommitLog, notifyCommit, RuleMalformedTree]
motifs: [fail-closed-wedge, ref-moved-call-failed]
refs: ['src://7b4887ce51d9/internal/store/verify_tree.go@3ac59b570c689e1983161f8cfa33e016e4330273:fe0bd683744c2edea784ecd776f2f8177a5f8fce', 'src://7b4887ce51d9/internal/store/fact_write.go@3ac59b570c689e1983161f8cfa33e016e4330273:07e3d2a31c1dd4d8b6019733d90249ce13f76f85', 'src://7b4887ce51d9/internal/store/plumbing.go@3ac59b570c689e1983161f8cfa33e016e4330273:ca81d953596293acdf765904ad688cc6d50839b9', 'src://7b4887ce51d9/internal/store/fact_read.go@3ac59b570c689e1983161f8cfa33e016e4330273:caac2fd0a1fbfdb23fbd7140a56da338aa5f09f8', 'src://7b4887ce51d9/internal/store/verify_tree_test.go@3ac59b570c689e1983161f8cfa33e016e4330273:f58a0b502e19755ed07688f3e791f6b0587be416', 'https://github.com/knomit/knomit/pull/523']
---
# A malformed tree on main wedges the gate for every later commit (each has it as a parent), in every verify_signatures mode and on every hosting; the way out is a write no gate judges: on a knomit host a host-local delete of the bogus entry, on a forge a push past the required `knomit verify ci` check; commit_log is NOT repaired, so a fresh repository is the clean recovery

Found in PR #523 (fix round 3, merge commit 3ac59b57) by reading the code and a throwaway probe test (deleted, never committed). The recovery route is stated in the comment on commitTreeShape.

THE WEDGE. The tree-shape rule checks the commit's tree and every IMMEDIATE parent's tree. If main already holds a commit with a malformed tree (pre-fix history, or an object fetched from a forge), every later commit has it as a parent and is refused malformed-tree, including the commit that repairs it. This holds in every verify_signatures mode on a knomit host, because receive-pack's closure checks parents too. It fails closed, which is deliberate. In the fleet repository a poisoned main also makes the loader fail, so CheckRange returns an error (exit 2) for every later commit.

WAYS OUT. (1) On a KNOMIT HOST: a host-local write to main, factIndex.deleteFile of the bogus entry's path. It calls no judge; fileExists finds the entry by name (FindEntry) and deletePath/removeEntry drop it whatever its mode. Probe: DeleteFact(main, "kb/zz-bogus") left main on a well-formed tree, and a push on top then passed the closure and created its branch, because only immediate parents are checked, so commits on top of the repair are admitted again. (2) On a FORGE: no path exists in knomit's code; the only way is a push to main that skips the required `knomit verify ci` check (an admin bypass of branch protection).

WHAT THE REPAIR DOES NOT FIX (known limitation). commit_log: populateCommitLog cannot diff the malformed commit ('a tree is missing: object not found'). After the local repair every local write on that branch still runs notifyCommit -> AppendCommitLog -> populateCommitLog AFTER SetReference has moved the ref, so the write returns an error although the ref moved (the #384 shape, 'ref moved, call failed'). A push after the repair returned 'could not be registered: populate commit_log after fast-forward' yet its branch WAS created. So a FRESH REPOSITORY is the clean recovery; the local repair un-wedges only the gate.

MISREADING. 'The local delete restores the repository' is false: commit_log stays wedged. 'A forge can repair it through knomit' is false. This is a pre-existing limit of commit_log, not new in #523; the gate now refuses to admit new malformed trees, so it arises only from history that predates it.

PINNED BY: TestCheckRange_MalformedParentTreeRefused and TestReceivePack_MalformedUpstreamTipRefused pin the parent-tree half of the check (a parent loop over ParentHashes[:0] turned both red); the recovery route itself was verified by a probe, not by a committed test.
