---
type: observation
domain: [repos, testing, agent-id]
confidence: 0.9
sources: 1
entities: [repos.Manager.AgentID, Deps.AgentID, testsigner.FixtureAgentID, vcs.SetTestFallbackAgentID, Repo.OwnAgentID, TestManagerAgentID_NoneGivenReportsNone]
motifs: [test-mode-hides-condition]
refs: ['src://7b4887ce51d9/internal/repos/manager.go@32414d74394f47dbd74aa3af88ec972a4f8e2394:5a38d4589b8c6e0ccd767b79d0d0ab4244c6d128', 'src://7b4887ce51d9/internal/testsupport/testsigner/testsigner.go@32414d74394f47dbd74aa3af88ec972a4f8e2394:ccbfb55ce4a460cb4022f82d65bb99a91a454cc2', 'src://7b4887ce51d9/internal/repos/agent_id_paths_test.go@32414d74394f47dbd74aa3af88ec972a4f8e2394:f398a6a0e319b7b3232ee1fa75a5f460b68ebd3c', 'src://7b4887ce51d9/test/testenv/storyboard.go@32414d74394f47dbd74aa3af88ec972a4f8e2394:8187f2da589428b07709409c4b30048c7ade0186', 'src://7b4887ce51d9/internal/store/internal/vcs/agent_id.go@32414d74394f47dbd74aa3af88ec972a4f8e2394:7b662e074731cf05369557457d0a9b1874c1d791', 'https://github.com/knomit/knomit/pull/584']
---
# repos.Manager.AgentID answers Deps.AgentID alone, in a test binary as in production: a test that builds a Manager over Deps{AgentBranch: ...} with no AgentID fails Start with "instance identity: no stored identity and none to create (empty agent id or branch)"

Since E7a (PR #584, merged as 32414d74; re-read there) Manager.AgentID is `return m.deps.AgentID` (internal/repos/manager.go). It no longer falls back to the installed test fixture (it used to read store.TestFallbackAgentID, a test seam, from production code; that forward is deleted). TestManagerAgentID_NoneGivenReportsNone (internal/repos/agent_id_paths_test.go) pins that a Manager built with only Deps{AgentBranch: ...} reports "" and does not derive an id from the branch name or from the fixture signer.

CONSEQUENCE: a test that builds a Manager and calls Start must set `AgentID: testsigner.FixtureAgentID(<branch>)` (internal/testsupport/testsigner). That is the same id a store authors as when it was given none, so existing expectations did not change. About 125 Deps{} literals in internal/repos, internal/web, test/testenv and test/storytests were edited to add it (test/testenv/storyboard.go is one). Without it Start fails with: instance identity: no stored identity and none to create (empty agent id or branch).

MISREADING: this does not mean the store-level fallback was removed. vcs.SetTestFallbackAgentID (installed through storetest.SetFallbackAgentID) and its unexported reader testFallbackAgentID, used by Repo.OwnAgentID alone, stay, so a bare store opened in a test still authors without SetAgentID. Nor does production set an id any differently. Falsifiability reported by the PR author, not re-run by the curator: with the fallback read removed and no fixture edit, about 60 repos tests fail at Start; agent_id_paths_test.go keeps asserting that the persisted id differs from the fixture's.
