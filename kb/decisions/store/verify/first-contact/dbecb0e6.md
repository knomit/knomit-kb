---
kind: pragmatic
type: policy
domain: [store, verify, repos, clone]
confidence: 0.9
sources: 1
entities: [InitFromRemote, InitSubscription, CloneFrom, checkOwnLineage, ErrForeignLineage, BranchACreateReads, initClone]
motifs: [decide-before-ref-moves, input-attacker-cannot-choose]
refs: ['src://7b4887ce51d9/internal/store/repo.go@fa932c1154b16d56c412d3350ccb1757934e2979:0c7a45c0c2878431c01baefb03211ff51b29ec06', 'src://7b4887ce51d9/internal/store/verify_e4.go@fa932c1154b16d56c412d3350ccb1757934e2979:428e6d032e0ca7bf1829ba6bc01d7cf781289a68', 'src://7b4887ce51d9/internal/store/verify_first_contact_test.go@fa932c1154b16d56c412d3350ccb1757934e2979:4de9e3f5a1700791d8529eacce18031e2e7ee6ce', 'kb://3ec012f5b4d2/kb/invariants/store/verify/anchor/9cc970dd.md', 'https://github.com/knomit/knomit/pull/319']
---
# F09 first contact (InitFromRemote, InitSubscription, CloneFrom) folds from the root BEFORE setting any local ref; E4 adopts origin's copy of the own agent branch only if every commit beyond the verified upstream is signed by the own key (or accepted), and otherwise FAILS the create — it never starts a fresh lineage

FIRST CONTACT: InitFromRemote, InitSubscription and CloneFrom call verifyAdvance with no local ref BEFORE any SetReference. The local upstream is set to the verified target (origin's tip for an off repo), and InitFromRemote bootstraps the agent branch and the watermark from that target, not from origin's tip. A fresh clone therefore reaches the same anchor as a long-running instance and never indexes, merges or re-pushes a refused commit. If nothing at all is acceptable, create fails with a VerifyFailedError.

E4 (repo.go, verify_e4.go checkOwnLineage), in EVERY mode including off, also BEFORE any local ref is set: origin/<agent branch> is adopted only if every commit reachable from it and not from the VERIFIED upstream carries an SSHSIG by this instance's own key (full fingerprint) or is on this instance's accept list (verify_accepted). Otherwise the create FAILS with a ForeignLineageError that lists every refused commit and the two remedies: merge that branch into the upstream on the forge and clone again, or `knomit verify accept <commit>` each one and clone again. No local upstream or agent branch is created and the remote is left untouched. A store with no signer cannot know its own key, so every such commit is refused; initClone sets the signer.

WHY FAIL RATHER THAN BOOTSTRAP (user ruling): bootstrapping a fresh lineage under the same name makes the first push overwrite the remote branch, silently discarding its commits. The motivating case, measured: debian-dev-24971f37 on cyberai-kb holds 4 unsigned (pre-signing) commits ahead of main. TestE4_PreSigningCommitsAreNotDiscarded reproduces it: refused with all 4 listed, then adopted once they are accepted.

E4 is the one check that trusts the own key alone: it reads nothing from the repo, so a forged remote cannot disarm it. BranchACreateReads/ProbeInitialized predict adoption from the remote's refs only, so they cannot foresee an E4 refusal; the create reports it.
