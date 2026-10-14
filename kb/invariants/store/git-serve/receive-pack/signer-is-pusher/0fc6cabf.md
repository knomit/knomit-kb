---
kind: pragmatic
type: policy
domain: [store, git-serve, fleet, verify, security]
confidence: 0.9
sources: 1
entities: [signerIsPusher, RuleNotPusher, receivePack.verify, judgeCommit, PushPolicy.Pusher, pushPolicy, auth.InstancePrincipal, CommitSigner.Fingerprint, pki.Fingerprint, D10]
motifs: [identity-by-key]
refs: ['kb://3ec012f5b4d2/kb/decisions/fleet/f11/push-verify-follows-mode/d9eb9ee4.md', 'kb://3ec012f5b4d2/kb/architecture/store/git-serve/receive-pack/7642c213.md', 'src://7b4887ce51d9/internal/store/httphandler_receivepack.go@1441b0448aeb33c133e1a7c48bc3329bfb44dfbd:eeead5f7c72d4b28cc227dfca261eb0ea25dc303#L336-L402', 'src://7b4887ce51d9/internal/store/verify_walk.go@1441b0448aeb33c133e1a7c48bc3329bfb44dfbd:3fa8fe01fbd06e04e795bf2f601fb1329dc36f6f', 'src://7b4887ce51d9/internal/web/gitremote.go@1441b0448aeb33c133e1a7c48bc3329bfb44dfbd:87c55711700dc695f0111f6a3cfd1601ab4d4bf9', 'src://7b4887ce51d9/internal/auth/listen_tls.go@1441b0448aeb33c133e1a7c48bc3329bfb44dfbd:1035d62bf3fea86a94cd70874251390a88cbb67b', 'src://7b4887ce51d9/internal/store/receivepack_closure_test.go@1441b0448aeb33c133e1a7c48bc3329bfb44dfbd:e73c77b5f053fa61bf692b23ab7a1f2e7c612c61', 'https://github.com/knomit/knomit/pull/511', 'src://7b4887ce51d9/internal/store/httphandler_receivepack.go@c9c599dee475771c4226212ac0889817c6b268d4:a397e188b9e58c24c8d70620ccb4ab5a3d4eb9dc', 'src://7b4887ce51d9/internal/store/receivepack_noname_test.go@c9c599dee475771c4226212ac0889817c6b268d4:f639eafd6fd3cc3b66714d491222e90a47a6b2c1', 'https://github.com/knomit/knomit/pull/513']
---
# Under verify_signatures=enforce a pushed new commit must be SSHSIG-signed by the PUSHER's own key (rule not-pusher), compared as full key fingerprints; log only logs it, off never checks

In `receivePack.verify`, a commit that `judgeCommit` accepted is then passed to `signerIsPusher`: if it carries a verifiable SSHSIG, `CommitSigner.Fingerprint` (pki.Fingerprint of the signing key, 64 hex) must equal `PushPolicy.Pusher`, which `internal/web/gitremote.go` `pushPolicy` sets to the cert principal's ID — `auth.InstancePrincipal` documents it as the FULL pki.Fingerprint of the client certificate's key (internal/auth/listen_tls.go). Both are the same key's full fingerprint; no name, branch suffix, agent id or e-mail is consulted for this rule (designer rule: no code may infer identity from names). The comparison is `strings.EqualFold` of the two 64-hex fingerprints (never the 8-hex short form, which appears only in the refusal text). A mismatch is a `Refusal{Rule: RuleNotPusher ("not-pusher")}`.

MODES: the rule rides the mode exactly like judgeCommit's rules — enforce refuses the push, log logs `would be refused (verify_signatures=log)` with rule not-pusher and accepts, off never runs it (verify returns before reading the fleet). Reason: under off and log an UNSIGNED commit is accepted anyway, so a signer rule there would stop nobody, and an always-on signature check on push was rejected by the user (kb/decisions/fleet/f11/push-verify-follows-mode/d9eb9ee4.md).

WHICH COMMITS: only new ones (newCommits stops at any commit the host holds). A member merging the host's main into its branch pushes other members' commits, but the host holds them, so only the member's own merge commit is judged. Commits the host does NOT hold that another member signed (relayed peer-to-peer, or merged from an origin main the host has not fetched yet) are refused under enforce. An unsigned merge that passed M3 has no signer and passes. CheckRange (`knomit verify ci`) has no pusher and does not apply this rule.

ABSENT PUSHER: none reachable — judgeRequest refuses an EMPTY Pusher (PR #513; it was `len(Pusher) < 8`) and pushPolicy sets Pusher only for a KindInstance/ViaCert principal; there is no non-mTLS push path.

TEST GOTCHA: receive-pack tests' `pushFP` is a fake fingerprint ("abcd1234"+zeros); any test pushing SIGNED commits under enforce must use the signer's real fingerprint as Pusher (`pusherOf` in receivepack_closure_test.go, whose branch is now 'agent/peer-opaque', no fp8) or it now fails not-pusher.

ONLY PUSH-TIME BINDING: since PR #513 deleted the branch-suffix rule, this rule is the ONLY push-time binding of content to the pusher, and it applies only under enforce (TestReceivePack_BranchNameCarriesNoOwnership).

GAP (PR #511 review F2): a push whose tip the host already holds sends no new commits, so this rule never runs for it: a member can point its own branch at another member's tip the host holds. That is not a content hole for `.knomit/` (the merge-time guard, kb/invariants/store/git-serve/pushed-branch-merge/private-path-guard/fbd12444.md, judges the merge result), but it means "every commit on a member's branch was signed by that member" does NOT follow from this rule.

Verified at merge commit 1441b044 (PR #511).
