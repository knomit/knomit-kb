---
kind: pragmatic
type: policy
domain: [fleet, repos, web, mcp, invariant]
confidence: 0.92
sources: 1
entities: [RepoInstance.WritableBranch, RepoInstance.IsFleet, refuseUnwritableBranch, ErrFleetReadOnly, ErrLensWriteFleet, guardFleetRemoval, handleStartSynthesis, handleHALRepoPatch, handlePushedMerge, experiment tool]
motifs: [single-write-seam, gate-per-door]
refs: ['src://7b4887ce51d9/internal/repos/instance.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:538fdde66fd25d4337c51364c8718ce9c1d4583c', 'src://7b4887ce51d9/internal/web/readonly.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:b171a1dfccaf5fad1c076174722eb05cb260ef2c', 'src://7b4887ce51d9/internal/web/handlers_jobs.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:afcf6e7b33b6a5b9b4c83f814f71c077722c4326', 'src://7b4887ce51d9/internal/web/handlers_repos_hal.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:debe9eb1934d94b11449abe96bcf22e0075f090f', 'src://7b4887ce51d9/internal/web/handlers_pushed_branches.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:a92bbcfc3456e012b7e9593ba93a5cfbc9688dea', 'src://7b4887ce51d9/internal/web/handlers_experiments.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:54c0a1076c8ffea6fe9b559ebfd48111535c0d13', 'src://7b4887ce51d9/internal/mcp/experiment.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:74e2e6a79b68f3439b54c2f6e51c84ece3131f25', 'src://7b4887ce51d9/internal/mcp/writedest.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:ca0f35bc362a6308d32fe02ffdfc37c6e35d35db', 'src://7b4887ce51d9/internal/repos/manager.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:af23ef62e0174251b77eba847ac9cee1333c18bd', 'src://7b4887ce51d9/internal/repos/fleet.go@1a6c03f705737f5c7c50213ed39c2df63dc3e7ce:7b2ff52f46c74ce18ed062ffe9bfa93422895d75', 'kb://3ec012f5b4d2/kb/decisions/repos/subscription/two-layer-write-refusal/d3853515.md', 'kb://3ec012f5b4d2/kb/invariants/repos/experiment-eligibility/2e6138b8.md', 'https://github.com/knomit/knomit/pull/520']
---
# The fleet repository is read-only for every authored write (MCP, fact REST routes, experiments, synthesis-runs, repo PATCH) and may be a lens READ mount only; it is never archived, deleted, restored or connected through the generic doors — but origin attach/detach/upstream are NOT gated

A mounted repo whose ontology id is fleet (RepoInstance.IsFleet) is read-only for authored writes, through one seam plus named gates:
- RepoInstance.WritableBranch returns false for it on every branch (checked right after the ontology-error check), and binding writeOK, MCP learn/update/retract/review/hypothesize and the REST fact create/update/delete routes all derive from it. REST answers 403 through refuseUnwritableBranch with the text 'the fleet repository changes only through `knomit fleet`' (ErrFleetReadOnly).
- MCP experiment tool: every action except `list` is refused on the fleet repository. REST experiment open: 409.
- POST /repos/{repo}/branches/{branch}/synthesis-runs and PATCH /repos/{repo} (it commits README.md and LICENSE) answer 403 through refuseUnwritableBranch(ri.AgentBranch()). The gate is on the AGENT branch because that is what those handlers write, not the branch in the URL. A subscription now also gets the named 403 on those two routes.
- A lens naming the fleet repository as its WRITE repo: ErrLensWriteFleet (422 'Lens write repo is the fleet repository'). As a READ mount it is fine, and it appears in the repo list and REST reads.
- Lifecycle: guardFleetRemoval answers 409 fleet_repository in EVERY fleet state for archive and delete, pointing at `knomit fleet unregister` (the old archive escape for a stuck unregistration, the 'use_unregister' arm and the state switch are gone); restore and connect are refused by the reserved-id rule (see the baked-ontology reserved-id invariant).
- The ONE exemption: handlePushedMerge (the operator's merge of a pushed branch) is not refused on the fleet repository.

NOT GATED at the merged commit: origin attach, detach and upstream change (PUT/PATCH/DELETE origin, and the origin-session commit that is not a swap). They change git config and control.db, not repository content, and PUT origin is today the only way to rotate a member's forge token. Be aware that detaching a member's origin turns consensus: auto on there (no origin). Refusing them is one IsFleet check in each handler if wanted.

CONSEQUENCE: the fleet repository has NO store-level read-only flag (a subscription has one: see the two-layer write refusal decision), so a new REST route or tool that writes repository content (facts, files, git) must call refuseUnwritableBranch or WritableBranch itself; nothing underneath will refuse it. The synthesis-runs and PATCH routes were found missing in review of this PR, so audit every new mutating route in router.go the same way.

DOES NOT MEAN: knomit's own fleet code cannot write the repository. It writes through the store and never asks WritableBranch.
