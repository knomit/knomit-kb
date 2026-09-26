---
kind: pragmatic
type: policy
domain: [store, verify, repos, clone]
confidence: 0.9
sources: 1
entities: [InitFromRemote, InitSubscription, CloneFrom, checkOwnLineage, ErrForeignLineage, BranchACreateReads, initClone]
motifs: [decide-before-ref-moves, input-attacker-cannot-choose]
refs: ['src://7b4887ce51d9/internal/store/repo.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:bbca27df6c44343bc5284b7e026e23db0ebffae6', 'src://7b4887ce51d9/internal/store/verify_e4.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:c0ede5d6d076433056fd9a7e09ebed621f3b0127', 'src://7b4887ce51d9/internal/store/verify_first_contact_test.go@023ac43d295fa3338ecbd4a72ddcf624e18e119e:06d8c1769804507553bbd3897043c87c8c8128b5']
---
# F09 first contact (InitFromRemote, InitSubscription, CloneFrom) folds from the root BEFORE setting any local ref, and E4 adopts origin's copy of the own agent branch only if every commit beyond the verified upstream is signed by the instance's own key — in every mode

FIRST CONTACT: InitFromRemote, InitSubscription and CloneFrom call verifyAdvance with no local ref BEFORE any SetReference. The local upstream is set to the verified target (origin's tip for an off repo), and InitFromRemote bootstraps the agent branch and the watermark from that target, not from origin's tip. A fresh clone therefore reaches the same anchor as a long-running instance and never indexes, merges or re-pushes a refused commit. If nothing at all is acceptable, create fails with a VerifyFailedError.

E4 (repo.go, verify_e4.go checkOwnLineage), in EVERY mode including off: origin/<agent branch> is adopted only if every commit reachable from it and not from the VERIFIED upstream carries an SSHSIG by this instance's own key (full fingerprint). Otherwise a WARN is logged and the branch bootstraps from the verified upstream; the first push then replaces the remote copy. A store with no signer cannot know its own key and refuses adoption, so initClone sets the signer. E4 is the one check that trusts the own key alone: it reads nothing from the repo, so a forged remote cannot disarm it.

KNOWN CONSEQUENCE: an instance whose remote agent branch holds UNMERGED commits from before signing existed (measured: debian-dev-24971f37 on cyberai-kb has 4 unsigned commits ahead of main) loses them from adoption on re-clone, and its first push overwrites them. They were never on main. BranchACreateReads/ProbeInitialized predict adoption from the remote's refs only, so they cannot foresee an E4 refusal.
