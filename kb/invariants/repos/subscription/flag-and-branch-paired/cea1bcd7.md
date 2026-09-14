---
kind: pragmatic
type: policy
domain: [repos, subscription, branches, invariant]
confidence: 0.95
sources: 1
entities: [RepoInstance.subscribed, RepoInstance.Subscribed, RepoInstance.ReadBranch, RepoInstance.AgentBranch, RepoInstance.WritableBranch, repoBuilder.subscribed, NewTestInstanceWithDeps, internal/repos/instance.go, internal/repos/builder.go]
motifs: [paired-fields-must-agree, failure-presents-as-success]
refs: ['src://7b4887ce51d9/internal/repos/instance.go@a1be0404ff37b9c342a90cb9241e17eab053d9f0:3dd5d4798afe37e604a02351f46ecd9534139c62', 'src://7b4887ce51d9/internal/repos/builder.go@a1be0404ff37b9c342a90cb9241e17eab053d9f0:b8ede3f32b31e8c834332134889968227ef34fa3', 'kb://3ec012f5b4d2/kb/decisions/lens/writes-agent-branch-only/a8589a83.md', 'https://github.com/knomit/knomit/pull/186']
---
# RepoInstance.subscribed is true ONLY alongside an empty agentBranch — the two are one state written in two places, and a mismatch fails silently

A subscription is represented by TWO fields that must always agree: `RepoInstance.subscribed == true` AND `agentBranch == ""`. Whoever sets one MUST set the other.

They are read by different consumers, which is why a mismatch is dangerous rather than merely untidy:
- `WritableBranch(branch)` keys on the BRANCH — it returns false only because `branch == ri.agentBranch` cannot hold for any non-empty branch when agentBranch is "".
- The repo DTO (`mode: "subscribe"`, omitted `agent_branch`) and the lens write-repo gate key on the FLAG.

So a repo built with `subscribed: true` but a non-empty agentBranch is simultaneously advertised as a read-only subscription AND writable on that agent branch. Nothing errors, nothing logs; the refusal that should have happened simply does not.

**Consequence for a consumer:** never derive one from the other at a call site, and never "fix" an apparent inconsistency by setting just one. `NewTestInstanceWithDeps` enforces the pairing for tests; a production constructor has to uphold it itself.

**What this does NOT mean:** it is not the claim that an empty agentBranch implies a subscription. The flag is explicit precisely so that "has no agent branch" and "is a subscription" stay distinguishable — a repo whose ontology failed to establish is also unwritable, and is not a subscription. See [[kb/decisions/lens/writes-agent-branch-only/a8589a83.md]] for the classification the branch half rides on.
