---
kind: pragmatic
type: policy
domain: [verify, ci, security]
confidence: 0.85
sources: 1
entities: [knomit verify ci, store.CheckRange, RangeVerdict, cmd.ExitCodeError, tools/ci/verify-signatures.yml, KNOMIT_FLEET_URL, KNOMIT_FLEET_TOKEN, merge-agent-branches.yml]
motifs: [one-implementation-two-sites]
refs: ['src://7b4887ce51d9/cmd/verify_signatures.go@3e2e83b082a450210e5de0111fa55774ccb891e6:5cf6fdc480c76c9d2877e82bae81b7f2008771bd', 'src://7b4887ce51d9/cmd/verify_signatures_test.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:c854475087513d989f8de81b681d156b0153301c', 'src://7b4887ce51d9/tools/ci/verify-signatures.yml@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:7eb3b95cda36b680e4e214263d7c958a6050dd18', 'src://7b4887ce51d9/internal/store/verify_range.go@3e2e83b082a450210e5de0111fa55774ccb891e6:e468ce9d89bc4d9cb4687318e990de52545a1181', 'kb://3ec012f5b4d2/kb/invariants/store/verify/artifact-rules/fd3ab8ab.md', 'https://github.com/knomit/knomit/pull/330', 'https://github.com/knomit/knomit/pull/362', 'https://github.com/knomit/knomit/pull/514']
---
# `knomit verify ci` IS the F09 acceptance gate for a GitHub-hosted knowledge base: store.CheckRange over plain checkouts, with --fleet <fleet checkout>; it reads verify_signatures at the upstream's tip FIRST and exits 0 without touching the fleet when it is off

`knomit verify ci --candidate <rev> --fleet <fleet checkout>` (cmd/verify_signatures.go) is a thin wrapper over store.CheckRange (F09 PR 5): it runs where a change enters the knowledge base's main (the merge job), once, and boots no knomit instance.

ORDER: verify_signatures at the upstream's tip first. Off or absent: exit 0, nothing else runs, --fleet may be omitted. On: the fleet checkout's ontology must be the fleet preset, and every commit the candidate adds must be signed by the CURRENT key of the ONE active member record at its author's agent id (unsigned merges pass only M3).

EXIT CONTRACT (pinned as literals by TestVerifyCI_ExitCodes; main's error-to-code path by TestExitCodeOf): 0 mergeable (off, clean, or log with refusals, which are printed), 1 blocked (enforce with refusals), 2 could not run (no --fleet while on, an unreadable checkout, a checkout that is not a fleet, an unknown verify_signatures value). Commands never os.Exit; they return cmd.ExitCodeError and main.exitCodeOf maps it.

WORKFLOW TEMPLATE (tools/ci/verify-signatures.yml): the fleet repository's URL is the repository variable KNOMIT_FLEET_URL (plus an optional read-token secret KNOMIT_FLEET_TOKEN for a private fleet); the job clones it only when set, and the check exits 0 when the knowledge base is off, so adding the template to an off repository changes nothing. KNOMIT_VERSION is a SET-ME placeholder until a release ships `verify ci --fleet`. There is no key list and no operator key: the fleet repository's member records are the only source of who may sign.

WHY NOT A STOCK-GIT SCRIPT: the gate runs the same code (CheckRange) every knomit build ships, so the rules cannot drift between a shell reimplementation and the binary.

SUPERSEDED: the PR 4 design that folded the whole candidate history with an operator key and verify_signers (VerifyCheckout) was removed with the fold in PR 5.

UPSTREAM DEFAULT (fix/no-hardcoded-consensus-branch): --upstream no longer defaults to origin/main. Unset, it is origin/<branch> for the branch refs/remotes/origin/HEAD names (cmd originHeadUpstream). It exits 2 when origin/HEAD is unset, or names an agent/exp/okf branch, and says to pass --upstream or run `git remote set-head origin --auto`. The template runs that set-head (actions/checkout does not set origin/HEAD), then fetches and omits --upstream, so its non-comment lines name no branch (TestVerifyCITemplate_NamesNoBranch). Pinned by TestVerifyCI_UpstreamDefaultsToOriginHead.

UPDATE (PR #514, verified at the merge commit): no new flags. The command's help text now describes the declared PKI files (when the fleet checkout holds the root, CRL and leaves that the fleet ontology's artifacts section declares, rules A and B apply) and the upstream's own artifact rules (read from the upstream's ontology at its tip, enforced only when verify_signatures is log or enforce). An upstream ontology that does not parse (for example a bad artifacts: value) is exit 2, never a pass. `verify ci` judges at the fleet revision it is given (--fleet-rev, default HEAD of the fleet checkout); a workflow that pins an old revision judges against that revision's CRL. Kb: kb/invariants/store/verify/artifact-rules/fd3ab8ab.md, kb/invariants/store/verify/rules-ab/73f5bf62.md.
