---
type: observation
domain: [web, repos, store, lens, testing, errors]
confidence: 0.85
sources: 1
entities: [resolveLensFactMount, lensFactMount, Binding.ByID, RepoInstance.ID, handleHALChanges, handleHALSetOrigin, getOrigin, httpx.WriteStoreUnavailable, TestLensFactMount_RoleDecidesTheUnavailableAnswer]
motifs: [unreachable-branch-needs-hook, fail-closed-masks-cause]
refs: ['src://7b4887ce51d9/internal/edge/web/handlers_lenses_read.go@1a860912e66c8cfd6da45c4c13d775aef55c58ed:7a41936ee6656893ac7596dac201fbe0a260f58c', 'src://7b4887ce51d9/internal/edge/web/handlers_changes.go@1a860912e66c8cfd6da45c4c13d775aef55c58ed:992dcc43ba7447920bdcc6f21445d7272c3665b8', 'src://7b4887ce51d9/internal/edge/web/handlers_origin_hal.go@1a860912e66c8cfd6da45c4c13d775aef55c58ed:e814cb7018cf6a8945d10d0aadfa9d77a3e1f193', 'src://7b4887ce51d9/internal/edge/web/handlers_lenses_unavailable_test.go@1a860912e66c8cfd6da45c4c13d775aef55c58ed:ddbad77b07ad316e43d4f05079de2f40760d7ec7', 'kb://3ec012f5b4d2/kb/decisions/web/read-store-unavailable-503/d5d606a6.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/lifecycle/write-403-before-store-503/624fcd06.md', 'https://github.com/knomit/knomit/pull/620']
---
# Not every web route reaches its store-unavailable 503 over HTTP: GET /changes only in a race (409 identity first), a qualified lens path to a down store is the unmounted 404 (so kb://<write-id>/ answers 404 while a bare path answers 503), the read-mount store-down branch needs a hook, and PUT origin's non-store getOrigin error is an untested 500

Edges of the one store-unavailable 503 (kb/decisions/web/read-store-unavailable-503/d5d606a6.md), recorded so nobody reads the decision as "every route answers 503 over HTTP when its store is down". Verified at 1a860912 by reading the code unless marked PR-reported.

- GET /repos/{repo}/.../changes: the handler needs the repo's wire id first, which comes from the store and is cleared by Identify's exit, so a store-less or closed repo answers 409 "Repo identity unresolved" BEFORE the ChangesUnder read. Its writeStoreUnavailable (the `aerr != nil` branch after ri.WithRead in handlers_changes.go) is reachable only in a race, where the store goes away between the identity check and the read.
- Lens single-fact, QUALIFIED path to a closed mount: Binding.ByID compares the id against rt.RI.ID(), and a closed instance has lost its id (Identify's exit), so the path resolves as UNMOUNTED and answers the unmounted 404, not the 503 and not answersNotFound. Consequence (user-stated known quirk, no leak): while the WRITE store is down, a bare path answers 503 but kb://<write-id>/ answers 404.
- The READ-mount store-down branch of lensFactMount.answersNotFound (readMount && storeUnavailable) is therefore unreachable over HTTP without a fault hook; it is pinned at the decision by TestLensFactMount_RoleDecidesTheUnavailableAnswer (PR-reported as passing), and the call-site mutant "pass false for the .knomit/ read" survives for that reason (PR-reported).
- PUT origin: getOrigin's prior-origin read answers the 503 when the error is storeUnavailable, and any OTHER getOrigin error answers 500 "Failed to get origin" with the error text. That 500 is untested: it needs a DB fault.
- A REST write to a closed machine-backed repo still answers 403 first (kb/gotchas/repos/lifecycle/write-403-before-store-503/624fcd06.md); that message is out of this PR's scope and stays on triage.

CONSEQUENCE: a test that wants the 503 over HTTP must use a store-less instance WITH an identity, or drive the helper directly; a closed machine-backed repo exercises the 409/403/unmounted-404 first.

DOES NOT MEAN: that the lens read-mount 404 or the write-mount 503 is wrong at these edges: the unmounted 404 is the correct answer for an id that no longer resolves, and it leaks nothing.

NAMES AFTER 12b (PR #622, merge commit 3e67923b on dev, re-read there): writeStoreUnavailable in this fact is now httpx.WriteStoreUnavailable (internal/edge/web/httpx/errors.go). The reachability matrix is unchanged; the handlers and tests it cites stay in internal/edge/web. Package overview: kb/architecture/web/httpx/ec668e7b.md.
