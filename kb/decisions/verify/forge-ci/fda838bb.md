---
kind: pragmatic
type: policy
domain: [verify, ci, security]
confidence: 0.85
sources: 1
entities: [knomit verify ci, store.VerifyCheckout, CheckoutVerdict, tools/ci/verify-signatures.yml, KNOMIT_VERIFY_OPERATOR_KEY, merge-agent-branches.yml]
motifs: [one-implementation-two-sites]
refs: ['src://7b4887ce51d9/internal/store/verify_checkout.go@d43978447eab2b7b3125cbac8dd34dcf434d1ca0:23ec8a2e17b3cc03a574022159d368056943a5a5', 'src://7b4887ce51d9/cmd/verify_signatures.go@d43978447eab2b7b3125cbac8dd34dcf434d1ca0:c3383e2b3f3bb35551d98b642aa4eba2a1fbcf24', 'src://7b4887ce51d9/tools/ci/verify-signatures.yml@d43978447eab2b7b3125cbac8dd34dcf434d1ca0:9e9d2e0d4d1e36ed7c8d92026052fc451b73d7a5']
---
# `knomit verify ci` is the forge-side F09 check: it runs the SAME fold over a plain git checkout, so policy comes from the fold's context and never the candidate head's ontology file; it has no accept list

`knomit verify ci --upstream origin/main --candidate <rev>` (cmd/verify_signatures.go -> store.VerifyCheckout) opens a plain git repository (bare or a work tree) with go-git, folds the CANDIDATE's history from the root exactly as a fresh knomit clone would, and restricts the verdict to the commits the candidate adds to the upstream. Exit 0 mergeable (or verification off), 1 blocked (a new commit refused, or a policy change it cannot judge), 2 could not run. It boots no app and needs no knomit home.

WHY NOT A STOCK-GIT SCRIPT (the proposal's first idea: git verify-commit + allowed_signers + merge-tree): the gate on #319 required the CI to evaluate the FOLD's context, never the head's ontology file. A candidate can carry an unauthorised change to verify_signatures in its own file; reading that file would let it switch its own check off. Re-implementing the fold in shell would be a second implementation to drift. The CI runs the same code as every instance.

OPERATOR KEY in CI: from KNOMIT_VERIFY_OPERATOR_KEY, a repository VARIABLE set by an admin, never a file in the repo (that would be self-referential). Without it, a candidate carrying a policy change is blocked (unrooted).

LIMITS (stated in the command help and the template): the job has no accept list, so a commit or merge needing a waiver (a criss-cross auto-merge) blocks and is merged by the operator. Direct pushes that bypass the workflow are covered by each instance's own verification, not by the job. tools/ci/verify-signatures.yml is the step to add to a KB repo's merge-agent-branches workflow before its merge step; its KNOMIT_VERSION is a SET-ME placeholder until a release ships `verify ci`.
