---
type: observation
domain: [repos, sync, origin, decision]
confidence: 0.95
sources: 1
entities: [DeleteOrigin, StartLocalSync, runLocalReconcileLoop, examples/mission/README.md]
refs: ['src://7b4887ce51d9/internal/web/handlers_origin_hal.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:995452bda1cda6330785f5abc3b7837f4c20523a', 'src://7b4887ce51d9/internal/repos/builder.go@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:65708c064cb52961253e6180d1c730bf4d6f3eb7', 'src://7b4887ce51d9/examples/mission/README.md@4e262b610ebb13e41d4bdc1f749079c7962c2f5b:8ddee8ae5b2f18a4390612c970bc79b2fcfdd4b0', 'kb://3ec012f5b4d2/kb/gotchas/repos/origin/detach-starts-no-local-loop/595f3e90.md', 'https://github.com/knomit/knomit/pull/362']
---
# Detaching an origin starts the origin-less reconcile loop at once (RepoInstance.StartLocalSync), instead of documenting a host restart

THE USER'S RULING (verbatim), on the #358 follow-up list that included "detaching an origin starting the local loop immediately": "ok". The surrounding ruling: "Hardcoded values are never good".

OPTIONS:
- (A) Keep the mission README's "restart the hosting instance" step. The local loop was chosen only at open. Rejected: a manual step every operator of the knomit-hosted route must remember, and until it is done nothing advances the consensus branch.
- (B) CHOSEN: DELETE /origin calls RepoInstance.StartLocalSync after the origin is removed. It drains the remote loop and starts runLocalReconcileLoop plus the experiment sweep on a fresh ctx, honouring DisableBackgroundSync.

ORDER is part of the decision: the start runs AFTER svc.SetOrigin(nil), because the local loop exits on a definite origin. The detach answers 204 even if the start fails (logged).

NON-SCOPE: it does not change when a repo that GAINS an origin switches loops (ActivateSync already does that). Details and tests: kb/gotchas/repos/origin/detach-starts-no-local-loop (historical). Shipped in #362, merged at 4e262b61.
