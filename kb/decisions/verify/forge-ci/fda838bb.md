---
kind: pragmatic
type: policy
domain: [verify, ci, security]
confidence: 0.85
sources: 1
entities: [knomit verify ci, store.CheckRange, RangeVerdict, cmd.ExitCodeError, tools/ci/verify-signatures.yml, KNOMIT_FLEET_URL, KNOMIT_FLEET_TOKEN, merge-agent-branches.yml]
motifs: [one-implementation-two-sites]
refs: ['src://7b4887ce51d9/cmd/verify_signatures.go@ba41ef143c2fbd30615660652678d28cb5c7bacf:b53e7f5cfa260c1eeb396b701b39e2a82b55f4f6', 'src://7b4887ce51d9/cmd/verify_signatures_test.go@ba41ef143c2fbd30615660652678d28cb5c7bacf:33a2200163eb70de2edf7f13a506d7230bc5d076', 'src://7b4887ce51d9/tools/ci/verify-signatures.yml@ba41ef143c2fbd30615660652678d28cb5c7bacf:ca11ae5f4fca6ccac46d88440141c92490182b9e', 'src://7b4887ce51d9/internal/store/verify_range.go@ba41ef143c2fbd30615660652678d28cb5c7bacf:e77b933b4320cf7751e631b50ebc80c8a690cb59', 'https://github.com/knomit/knomit/pull/330']
---
# `knomit verify ci` IS the F09 acceptance gate for a GitHub-hosted knowledge base: store.CheckRange over plain checkouts, with --fleet <fleet checkout>; it reads verify_signatures at the upstream's tip FIRST and exits 0 without touching the fleet when it is off

`knomit verify ci --candidate <rev> --fleet <fleet checkout>` (cmd/verify_signatures.go) is a thin wrapper over store.CheckRange (F09 PR 5): it runs where a change enters the knowledge base's main (the merge job), once, and boots no knomit instance.

ORDER: verify_signatures at the upstream's tip first. Off or absent: exit 0, nothing else runs, --fleet may be omitted. On: the fleet checkout's ontology must be the fleet preset, and every commit the candidate adds must be signed by the CURRENT key of the ONE active member record at its author's agent id (unsigned merges pass only M3).

EXIT CONTRACT (pinned as literals by TestVerifyCI_ExitCodes; main's error-to-code path by TestExitCodeOf): 0 mergeable (off, clean, or log with refusals, which are printed), 1 blocked (enforce with refusals), 2 could not run (no --fleet while on, an unreadable checkout, a checkout that is not a fleet, an unknown verify_signatures value). Commands never os.Exit; they return cmd.ExitCodeError and main.exitCodeOf maps it.

WORKFLOW TEMPLATE (tools/ci/verify-signatures.yml): the fleet repository's URL is the repository variable KNOMIT_FLEET_URL (plus an optional read-token secret KNOMIT_FLEET_TOKEN for a private fleet); the job clones it only when set, and the check exits 0 when the knowledge base is off, so adding the template to an off repository changes nothing. KNOMIT_VERSION is a SET-ME placeholder until a release ships `verify ci --fleet`. There is no key list and no operator key: the fleet repository's member records are the only source of who may sign.

WHY NOT A STOCK-GIT SCRIPT: the gate runs the same code (CheckRange) every knomit build ships, so the rules cannot drift between a shell reimplementation and the binary.

SUPERSEDED: the PR 4 design that folded the whole candidate history with an operator key and verify_signers (VerifyCheckout) was removed with the fold in PR 5.

UPSTREAM DEFAULT (fix/no-hardcoded-consensus-branch): --upstream no longer defaults to origin/main. Unset, it is origin/<branch> for the branch refs/remotes/origin/HEAD names (cmd originHeadUpstream). It exits 2 when origin/HEAD is unset, or names an agent/exp/okf branch, and says to pass --upstream or run `git remote set-head origin --auto`. The template runs that set-head (actions/checkout does not set origin/HEAD), then fetches and omits --upstream, so its non-comment lines name no branch (TestVerifyCITemplate_NamesNoBranch). Pinned by TestVerifyCI_UpstreamDefaultsToOriginHead.
