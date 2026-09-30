---
kind: pragmatic
type: policy
domain: [store, repos, consensus, invariant]
confidence: 0.9
sources: 1
entities: [DefaultConsensusBranch, ErrNoConsensusBranch, ChooseConsensusBranch, TestNoHardcodedBranchLiterals, Service.ConfigureRemote, remoteIndex.Sync, Origins.Set, UpstreamBranch]
motifs: [false-universal-default, single-named-default]
refs: ['src://7b4887ce51d9/internal/store/consensus_branch.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:c5454ba97845409dc554ae0af8d3bf348d7364ef', 'src://7b4887ce51d9/internal/store/consensus_branch_literals_test.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:3508b3643c9570cdebcc99b55b53f5f580f92d22', 'src://7b4887ce51d9/internal/store/consensus_branch_choose_test.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:a06b835e8ab9f7424955fe1ba21560eda87c73fc', 'src://7b4887ce51d9/internal/store/service.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:9a229982458f0f86afb9c35856bba60a8eae5f07', 'src://7b4887ce51d9/internal/store/remote_sync.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:9f7107ba83d3d664c575c07ff6e07f49525ae08e', 'src://7b4887ce51d9/internal/repos/origins.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:598a8a0af55a6525aed4454414a1f9eb332870f0', 'src://7b4887ce51d9/cmd/verify_signatures.go@754cf2c0a9b0f8fc3eabb93459f9e0e3a7c8ccb3:643e3e8fd75ca8b2f80cc27c133b522b67143680', 'kb://3ec012f5b4d2/kb/invariants/store/consensus-branch-recorded/bcde3d66.md']
---
# No code names a consensus branch by convention: the only branch-name literal is store.DefaultConsensusBranch, used only when a consensus branch is CREATED with nothing to detect it from; every other path takes the name from a request, the remote, or the recorded branch, or refuses with ErrNoConsensusBranch

The user's ruling: "Hardcoded values are never good" (fix/no-hardcoded-consensus-branch, the follow-up to #358).

The rule:
- store.DefaultConsensusBranch (internal/store/consensus_branch.go) is defined once. It is used where knomit CREATES the consensus branch and nobody named it: InitRepo/InitRepoWithUpstream with an empty name (a local create; preset/custom modes carry no branch), initFromEmptyRemote (the seed path; an empty remote advertises no HEAD), and the probe's empty-remote answer, which mirrors the seed path. It is a constant and not git's init.defaultBranch, because the name matters only at birth (it is recorded and read back from then on). Reading the running account's gitconfig would make two hosts created by the same request disagree.
- A remote's consensus branch comes from store.ChooseConsensusBranch (see kb/decisions/store/consensus-branch/remote-rule-head-not-name).
- The public entries into the fetch/reconcile machinery REFUSE an empty name with store.ErrNoConsensusBranch: Service.ConfigureRemote, remoteIndex.Sync (an origin with no Branch), repos.Origins.Set/SetBranch. The internal paths below them (configureRemote, fetchOrigin, reconcileNow, reconcileMain, reconcileAgent, reconcileAgentMerge, reconcileAgentRebase) carry no default at all, because "" cannot reach them.
- Readers use the recorded branch (store.UpstreamBranch): the fleet's members/keys, PUT /origin with no branch, tools/calibrate --branch. `verify ci --upstream` defaults to refs/remotes/origin/HEAD's branch and refuses when that is unset or is an agent/exp/okf branch. Replay with no DefaultBranch uses the target clone's HEAD.

ENFORCED BY TestNoHardcodedBranchLiterals (internal/store/consensus_branch_literals_test.go). It parses the non-test Go files of cmd/, internal/, tools/okf and tools/calibrate with go/ast and fails on any string literal equal to "main"/"master" or ending in "/main"/"/master". The single allowed literal is the constant's definition, which must occur exactly once. Comments and prose do not count. Neither does the pkitest helper, whose "master" is a key directory.

CONSEQUENCE: to fall back to a branch, read UpstreamBranch() or refuse. Adding `if b == "" { b = "main" }` anywhere turns the scan red.

MISREADINGS: DefaultConsensusBranch is NOT a read fallback. Using it where a repo's branch is unknown reintroduces the bug #358 fixed (a trunk repo reading a zero-tip main). The word "main" in prose, UI copy and the OpenAPI text, meaning the consensus role, is allowed (the principle that main names the role). Web UI strings are not covered by the scan; vitest pins those sites (StepBranch, StepReview, RemoteStatus, api.getAgentBranch).
