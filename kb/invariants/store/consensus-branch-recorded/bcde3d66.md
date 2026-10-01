---
kind: pragmatic
type: policy
domain: [store, repos, remote, consensus]
confidence: 0.9
sources: 1
entities: [UpstreamBranch, localConsensusBranch, recordConsensusBranch, consensus_branch, SetOrigin, InitRepoWithUpstream, InitFromRemote, TestMission_ReadmeKnomitHostedRoute]
motifs: [false-universal-default, record-not-assume]
refs: ['src://7b4887ce51d9/internal/store/consensus_branch.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:c5454ba97845409dc554ae0af8d3bf348d7364ef', 'src://7b4887ce51d9/internal/store/consensus_branch_test.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:847cbd115af25c2f23d110d1765169b37f13d868', 'src://7b4887ce51d9/internal/store/service.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:9a229982458f0f86afb9c35856bba60a8eae5f07', 'src://7b4887ce51d9/internal/store/repo.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:3476937d358dc4433881ee27fd4be5f80b22159d', 'src://7b4887ce51d9/cmd/mission_route_test.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:9b0cdc8cde84c5b285ed64dc1cd376399f34dd82', 'kb://3ec012f5b4d2/kb/gotchas/repos/consensus/no-origin-upstream-is-main/3f21db55.md', 'kb://3ec012f5b4d2/kb/invariants/repos/clone-persists-resolved-upstream/ca2246c6.md', 'kb://3ec012f5b4d2/kb/invariants/store/branch-roles/962ca98d.md', 'https://github.com/knomit/knomit/pull/358', 'https://github.com/knomit/knomit/pull/362']
---
# A repo's consensus branch is RECORDED in its repo database (meta key consensus_branch) whenever it is established — local init, clone, subscription, every SetOrigin with a branch — and UpstreamBranch answers the origin's branch, else the recorded one; removing the origin must never change it, and no code may fall back to a literal branch name

internal/store/consensus_branch.go (F08 PR D, reviewer blocker B1); an instance of kb/invariants/store/branch-roles (a branch holds a role only because an explicit record says so). Before it, UpstreamBranch returned "main" for any repo with no origin row, so the mission template's knomit-hosted route (clone a trunk or master repository, then DELETE /api/v1/repos/<repo>/origin) switched the host to a zero-tip "main": skills, recipes and the consensus/conflicts settings read from nothing, and peers failed to clone ("consensus branch does not exist in this store").

The rule:
- Writers: InitRepoWithUpstream records the branch it creates; InitFromRemote (deferred, on success) the branch it RESOLVED; InitSubscription likewise; Service.SetOrigin records o.Branch for every non-nil origin with a branch (so a PUT origin that changes the branch keeps the record current, and repos cloned before the key get it at their next open, because SetOrigin runs at open).
- Reader: UpstreamBranch = origin row's Branch, else remoteIndex's cached value, else meta, else (a repo written before the key) the single refs/heads branch that is not agent/*, exp/* or okf/*, which is then recorded. Several candidates: warned once, the first by name used, NOT recorded. No candidate: "".
- Why meta: the name names one of the repo's own refs and must survive exactly what the refs survive; control.db's origin row is exactly what a detach deletes.

CONSEQUENCE: a knomit-hosted (no-origin) repo can have any consensus branch name, and consensus: auto acts on it (cmd/mission_route_test.go runs trunk and master, host restart included). Since fix/no-hardcoded-consensus-branch, DELETE origin also starts the local loop at once (RepoInstance.StartLocalSync; kb/gotchas/repos/origin/detach-starts-no-local-loop is now historical). MISREADING: InitRepo still CREATES a branch nobody named, called store.DefaultConsensusBranch. That constant is the one named default, and it is used only at birth; the created name is recorded and read back (kb/invariants/store/consensus-branch/one-named-default). It is never a fallback for a repo whose branch is unknown.

Pinned by TestConsensusBranch_RecordedNotDefaulted (sabotages: the literal "main" restored → all three subtests red; SetOrigin not recording → 'removing the origin keeps its branch' red) and TestMission_ReadmeKnomitHostedRoute (the literal restored → host consensus branch "main" with no tip, peer clone fails).
