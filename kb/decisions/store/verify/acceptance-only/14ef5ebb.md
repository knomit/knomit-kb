---
kind: pragmatic
type: policy
domain: [verify, store, sync, fleet]
confidence: 0.9
sources: 1
entities: [reconcileMain, reconcileMainTo, verify_walk.go, walkHistory, verify_context, 000028_drop_verify_context]
refs: ['src://7b4887ce51d9/internal/store/remote_reconcile.go@7550aef5d96a8d873f36c8b3a3d781446edbe8a2:f17d5ff159390ddbb86ebd82b41734bc37f36f49', 'src://7b4887ce51d9/internal/store/verify_walk.go@7550aef5d96a8d873f36c8b3a3d781446edbe8a2:be472191b2f1457165543872b6d040e681b4d7b5', 'src://7b4887ce51d9/internal/store/migrate/repo/000028_drop_verify_context.up.sql@7550aef5d96a8d873f36c8b3a3d781446edbe8a2:8c580fb7b2ff564aa2d6fb9bfd1fdc01261730a8']
---
# F09 verifies a change once, at the gate that advances a KB's main; no instance re-verifies history on fetch, clone, reconcile or merge (the fold, anchor and verify_context are gone)

F09 PR 5, user ruling: "changes are verified when they're accepted, not every time when a branch is merged" and the transitive-trust ruling. reconcileMain moves the local upstream to origin's tip (reconcileMainTo) with no verification; first contact (CloneFrom, InitFromRemote, InitSubscription) sets refs at origin's tips. Deleted: verifyAdvance, the anchor ref refs/knomit/verified/*, verify_context (repo migration 000028 drops it; 000027 is never edited), the history fold, verify_signers, [verify].operator_key and the RootOfTrust wiring, MainReconcileResult.Verify (a JSON field; dev-only), the verify-failed SSE event and knomit_verify_refused_total.

Kept: SSHSIG verification (verify_sig.go), M3 (verify_m3.go), E4 (verify_e4.go, with walkHistory and the accept lookup in verify_walk.go; the accept list stays the repo-DB verify_accepted table on dev), fail-closed signing.

MISREADING: this does not mean nothing is ever checked: the acceptance gate (store.CheckRange / knomit verify ci) checks the candidate range, and knomit verify audit re-checks a whole branch on demand. A breach is handled forensically, not by re-verifying on every read.
