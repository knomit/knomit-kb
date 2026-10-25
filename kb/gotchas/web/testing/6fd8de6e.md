---
type: observation
domain: [web, testing, routing]
confidence: 0.9
sources: 1
entities: [newTestRouter, lensFromURL, repoFromURL, LensMiddleware, RepoMiddleware, TestRouter_RouteTableMatchesGolden, timeNow, webExpNow, api/kb, harness_test.go]
motifs: [narrow-test-double, harness-over-model]
refs: ['src://7b4887ce51d9/internal/edge/web/api/kb/harness_test.go@33eb030738eeefc277e8744f5f8c71e39336251f:905e33862a3236ba49f7a318bb40118b93d1e202', 'src://7b4887ce51d9/internal/edge/web/api/kb/expires_test.go@33eb030738eeefc277e8744f5f8c71e39336251f:7bb335e71f6c9f7ea37e62d0a27bb62459a58b6b', 'src://7b4887ce51d9/internal/edge/web/repo_middleware.go@33eb030738eeefc277e8744f5f8c71e39336251f:26e792992258c9c0bc71015d8622eef985070f74', 'src://7b4887ce51d9/internal/edge/web/lens_middleware.go@33eb030738eeefc277e8744f5f8c71e39336251f:16d498b7468ca663b2ad1af8bf9500543d26da10', 'src://7b4887ce51d9/internal/edge/web/api/origins/harness_test.go@33eb030738eeefc277e8744f5f8c71e39336251f:b25a79dd6e0764ecef5efccf1a098b48e51c47af', 'https://github.com/knomit/knomit/pull/630']
---
# api/kb's package-private HTTP tests use newTestRouter(m) (harness_test.go), not web's Server: its lensFromURL maps EVERY resolve error to 404 and its repoFromURL omits the storeless 409 and the archived detail, so never assert lens-unavailable or repo-unavailable statuses through it

Verified at 33eb0307 (PR #630, 12f; the finding is from the #630 review). newTestRouter runs kb.Mount(branch, lens, "kb", nil) behind COPIES of the {repo}, {branch} and {lens} resolution only. It has no auth edge, no writeGate, no readOnlyGate, and no api/manage routes. So a kb test creates lenses with Manager.CreateLens and repos with Manager.Create (which returns once the repo is serving and indexed), and needs no fromLoopback.

THE COPIES ARE NARROWER THAN THE REAL MIDDLEWARE. lensFromURL maps every repos.ResolveLensBinding error to 404 'Lens not found'; the real LensMiddleware (internal/edge/web/lens_middleware.go) answers 503 for a lens-registry-unavailable or lens-unavailable error and 404 only for an unknown lens. repoFromURL answers 404 whenever the repo is not found; the real RepoMiddleware answers 409 'Repo unavailable' for a registered name with no live store and carries the archived detail for an archived repo. CONSEQUENCE: never assert a lens-unavailable or repo-unavailable status (503, or the storeless 409, or the archived detail) through newTestRouter: it either cannot produce it or produces the wrong one, and a test written against it would pin the harness, not the server. Such a test drives the full web router (Server.NewAPIRouter in internal/edge/web).

WHERE TESTS GO. A test that must see auth, the write gate or the lifecycle routes together with a KB route stays in internal/edge/web and drives Server.NewAPIRouter. A test that needs a kb internal lives in api/kb and drives newTestRouter. Two tests moved there because they need kb internals: the expiry REST tests (expires_test.go), which pin `timeNow`, an unexported var in api/kb; and TestFactWrite_ClosedRepo_RefusedAndWritesNothing, which calls writeFact, deleteFact and the other write helpers.

THE PINNED CLOCK IS LOAD-BEARING. webExpNow (2026-10-01) is in the past relative to real time, so the 'soon' fact counts as unexpired only under the pinned timeNow. A kb change that reads time.Now() directly in fact_view.go instead of timeNow() makes TestREST_FactReadCarriesExpiry red (PR-reported sabotage: f.IsExpired(time.Now())).

DOES NOT MEAN: that the kb harness proves middleware order or nesting. TestRouter_RouteTableMatchesGolden in web does that. api/origins and api/manage have their own harness_test.go with their own copies (a fixture fix may need making in each).
