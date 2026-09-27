---
kind: pragmatic
type: policy
domain: [verify, ci, security]
confidence: 0.85
sources: 1
entities: [knomit verify ci, store.VerifyCheckout, CheckoutVerdict, tools/ci/verify-signatures.yml, KNOMIT_VERIFY_OPERATOR_KEY, merge-agent-branches.yml]
motifs: [one-implementation-two-sites]
refs: ['src://7b4887ce51d9/internal/store/verify_checkout.go@de15855bbc3b96f0db86e519dd86d6e37e259a0a:18e23de381dd547c5c0da48b6ccd02de280b9cc7', 'src://7b4887ce51d9/cmd/verify_signatures.go@de15855bbc3b96f0db86e519dd86d6e37e259a0a:a8bdbeaad1dbe0490c641c93932fac7ce6e0b169', 'src://7b4887ce51d9/tools/ci/verify-signatures.yml@de15855bbc3b96f0db86e519dd86d6e37e259a0a:64ad2250f2980aaa961d9d57707beb62e6e3f1cc', 'src://7b4887ce51d9/cmd/verify_signatures_test.go@de15855bbc3b96f0db86e519dd86d6e37e259a0a:ee9b4f26ee328f404d6b5284f8a90b7b079139d7', 'src://7b4887ce51d9/main.go@de15855bbc3b96f0db86e519dd86d6e37e259a0a:2ccd89e4dca4bd4aa3541e45d9c44a297bffbb12']
---
# `knomit verify ci` is the forge-side F09 check: it runs the SAME fold over a plain git checkout, so policy comes from the fold's context and never the candidate head's ontology file; it has no accept list

`knomit verify ci --upstream origin/main --candidate <rev>` (cmd/verify_signatures.go -> store.VerifyCheckout) opens a plain git repository (bare or a work tree) with go-git, folds the CANDIDATE's history from the root exactly as a fresh knomit clone would, and restricts the verdict to the commits the candidate adds to the upstream. It boots no app and needs no knomit home.

EXIT CONTRACT (what the forge job reads; pinned as LITERALS by TestVerifyCI_ExitCodes / TestVerifyCI_CouldNotRunIsExitTwo in cmd, and main's error-to-code path by TestExitCodeOf in main_test.go): 0 mergeable, 1 blocked, 2 could not run. Commands never call os.Exit (dev's TestCmd_NoProcessExit); they return cmd.ExitCodeError and main.exitCodeOf maps it. A test comparing against the constant exitFailed instead of 2 does NOT pin the contract: changing the constant keeps it green.

BLOCKED (CheckoutVerdict.Blocked) = unrooted (a policy change and no operator key; the message says "blocked: unrooted"), OR mode log/enforce with a refused new commit, OR a new commit whose policy-change or unreadable setting the fold only NOTED (log keeps the old context and reports; off reports an unauthorised enable). The last is stricter than log's "report and ignore" on purpose: refusing costs the forge nothing (the branch stays open), and merging the note would leave main's ontology file saying one mode while every instance folds another. With verification off and no policy change, nothing blocks.

WHY NOT A STOCK-GIT SCRIPT (the proposal's first idea: git verify-commit + allowed_signers + merge-tree): the gate on #319 required the CI to evaluate the FOLD's context, never the head's ontology file. A candidate can carry an unauthorised change to verify_signatures in its own file; reading that file would let it switch its own check off. Re-implementing the fold in shell would be a second implementation to drift. The CI runs the same code as every instance.

OPERATOR KEY in CI: from KNOMIT_VERIFY_OPERATOR_KEY, a repository VARIABLE set by an admin, never a file in the repo (that would be self-referential).

LIMITS (stated in the command help and the template): the job has no accept list, so a commit or merge needing a waiver (a criss-cross auto-merge) blocks and is merged by the operator. Direct pushes that bypass the workflow are covered by each instance's own verification, not by the job. GitHub runs a push event's workflow FROM THE PUSHED BRANCH, so whoever can push an agent branch can edit .github/workflows/ there and drop the step; the per-instance gate still refuses the result. tools/ci/verify-signatures.yml is the step to add to a KB repo's merge-agent-branches workflow before its merge step; its KNOMIT_VERSION is a SET-ME placeholder until a release ships `verify ci`.
