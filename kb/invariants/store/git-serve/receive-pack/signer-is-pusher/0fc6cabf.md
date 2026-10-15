---
kind: pragmatic
type: policy
domain: [store, git-serve, fleet, verify, security]
confidence: 0.9
sources: 1
entities: [signerIsPusher, RuleNotPusher, receivePack.verify, judgeCommit, PushPolicy.Pusher, pushPolicy, auth.InstancePrincipal, CommitSigner.Fingerprint, pki.Fingerprint, D10]
motifs: [identity-by-key]
refs: ['kb://3ec012f5b4d2/kb/decisions/fleet/f11/push-verify-follows-mode/d9eb9ee4.md', 'kb://3ec012f5b4d2/kb/architecture/store/git-serve/receive-pack/7642c213.md', 'kb://3ec012f5b4d2/kb/decisions/store/verify/m3-first/b3bb33da.md', 'src://7b4887ce51d9/internal/store/httphandler_receivepack.go@3ac59b570c689e1983161f8cfa33e016e4330273:9a3f65deedf153857fcfb56334be4a0ff22780ff', 'src://7b4887ce51d9/internal/store/verify_walk.go@3ac59b570c689e1983161f8cfa33e016e4330273:db3bd26323b2c6f5fdeb5fa53deaf003ad15a6fa', 'src://7b4887ce51d9/internal/store/receivepack_rules_ab_test.go@3ac59b570c689e1983161f8cfa33e016e4330273:3c737e408f8c5a354dcc1a07f8d90b00803ecbdd', 'src://7b4887ce51d9/internal/web/gitremote.go@3e2e83b082a450210e5de0111fa55774ccb891e6:924414e11610604cb8f8d572dd40dfc700721950', 'src://7b4887ce51d9/internal/auth/listen_tls.go@1441b0448aeb33c133e1a7c48bc3329bfb44dfbd:1035d62bf3fea86a94cd70874251390a88cbb67b', 'src://7b4887ce51d9/internal/store/receivepack_closure_test.go@3ac59b570c689e1983161f8cfa33e016e4330273:ed82a00a8045e47403b0fa1bece36b82f219f3ba', 'src://7b4887ce51d9/internal/store/receivepack_noname_test.go@3ac59b570c689e1983161f8cfa33e016e4330273:6381c990c5981551546355cf74c88c7ab65b5aab', 'https://github.com/knomit/knomit/pull/511', 'https://github.com/knomit/knomit/pull/513', 'https://github.com/knomit/knomit/pull/514', 'https://github.com/knomit/knomit/pull/523']
---
# Under verify_signatures=enforce a pushed new commit must be SSHSIG-signed by the PUSHER's own key (rule not-pusher), compared as full key fingerprints; log only logs it, off never checks

In `receivePack.verify`, a commit that `judgeCommit` accepted is then passed to `signerIsPusher`: if it carries a verifiable SSHSIG, `CommitSigner.Fingerprint` (pki.Fingerprint of the signing key, 64 hex) must equal `PushPolicy.Pusher`, which `internal/web/gitremote.go` `pushPolicy` sets to the cert principal's ID — `auth.InstancePrincipal` documents it as the FULL pki.Fingerprint of the client certificate's key (internal/auth/listen_tls.go). Both are the same key's full fingerprint; no name, branch suffix, agent id or e-mail is consulted for this rule (designer rule: no code may infer identity from names). The comparison is `strings.EqualFold` of the two 64-hex fingerprints (never the 8-hex short form, which appears only in the refusal text). A mismatch is a `Refusal{Rule: RuleNotPusher ("not-pusher")}`.

MODES: the rule rides the mode exactly like judgeCommit's rules — enforce refuses the push, log logs `would be refused (verify_signatures=log)` with rule not-pusher and accepts, off never runs it (verify returns before reading the fleet). Reason: under off and log an UNSIGNED commit is accepted anyway, so a signer rule there would stop nobody, and an always-on signature check on push was rejected by the user (kb/decisions/fleet/f11/push-verify-follows-mode/d9eb9ee4.md).

WHICH COMMITS: only new ones (newCommits stops at any commit the host holds). A member merging the host's main into its branch pushes other members' commits, but the host holds them, so only the member's own merge commit is judged. Commits the host does NOT hold that another member signed (relayed peer-to-peer, or merged from an origin main the host has not fetched yet) are refused under enforce. A merge that passed M3 without a VERIFYING signature (unsigned, or a signature present but invalid) has no signer and passes; a VERIFYING signature on an M3-clean merge is still compared with the pusher (not-pusher; TestReceivePack_CleanMergeSignatureStates). CheckRange (`knomit verify ci`) has no pusher and does not apply this rule.

ABSENT PUSHER: none reachable — judgeRequest refuses an EMPTY Pusher (PR #513; it was `len(Pusher) < 8`) and pushPolicy sets Pusher only for a KindInstance/ViaCert principal; there is no non-mTLS push path.

TEST GOTCHA: receive-pack tests' `pushFP` is a fake fingerprint ("abcd1234"+zeros); any test pushing SIGNED commits under enforce must use the signer's real fingerprint as Pusher (`pusherOf` in receivepack_closure_test.go, whose branch is now 'agent/peer-opaque', no fp8) or it now fails not-pusher.

ONLY PUSH-TIME BINDING: since PR #513 deleted the branch-suffix rule, this rule is the ONLY push-time binding of content to the pusher, and it applies only under enforce (TestReceivePack_BranchNameCarriesNoOwnership).

GAP (PR #511 review F2): a push whose tip the host already holds sends no new commits, so this rule never runs for it: a member can point its own branch at another member's tip the host holds. That is not a content hole for `.knomit/` (the merge-time guard, kb/invariants/store/git-serve/pushed-branch-merge/private-path-guard/fbd12444.md, judges the merge result), but it means "every commit on a member's branch was signed by that member" does NOT follow from this rule.

Verified at merge commit 1441b044 (PR #511).

PR #514 (verified at the merge commit): verifyModeAt now also returns the parsed ontology (it is parsed when verification is on, so the judge can apply the artifact rules and know whether the repository is the fleet); the not-pusher rule is unchanged and still runs after judgeCommit accepts.

UPDATE (PR #523, verified at merge commit 3ac59b57): on a clean (M3) merge this check protects nothing: the pusher can strip or corrupt the signature and the same merge is admitted (user ruling: a clean merge has no content of its own; kb/decisions/store/verify/m3-first/b3bb33da.md). That is harmless only because the merge's parents are either new in this push (judged on their own) or commits the host already holds. For every OTHER signed commit the check matters as before. The closure also refuses a malformed tree in every mode before this rule is reached (kb/invariants/store/verify/tree-shape/fc07357f.md).
