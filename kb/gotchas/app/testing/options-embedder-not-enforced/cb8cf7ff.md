---
type: observation
domain: [app, testing, ci, embeddings]
confidence: 0.9
sources: 2
entities: [app.New, Options.Embedder, newProductionEmbedder, testenv.DeterministicEmbedder, internal/app/oauth_wiring_test.go, TestApp_NoOAuthBuildsNoIssuer, TestApp_SignedApprovalWired, TestNew_EmbedderRequired, TestBoot_RequireWithoutLocalListenerFails]
motifs: [opt-in-seam-regresses, convention-without-enforcement]
refs: ['kb://3ec012f5b4d2/kb/decisions/app/boot/options-embedder-test-seam/7bb9d36d.md', 'kb://3ec012f5b4d2/kb/gotchas/ci/race-workflow/internal-app-needs-real-embedder/9f6f09a4.md', 'kb://3ec012f5b4d2/kb/gotchas/app/boot/closers-not-run-on-failed-new/7e7a2250.md', 'https://github.com/knomit/knomit/pull/284', 'https://github.com/knomit/knomit/pull/283']
---
# Any internal/app test that calls app.New WITHOUT Options.Embedder silently brings the ONNX runtime and the 612 MB model download back into the package — nothing enforces the seam, and dev already did it once (PR #283's TestApp_NoOAuthBuildsNoIssuer and TestApp_SignedApprovalWired) between PR #284's branch and its merge

PR #284 made internal/app's wiring tests boot through Options.Embedder (a store.BatchEmbedder test seam; testenv.DeterministicEmbedder in practice). The seam is opt-in per call site: app.New with a nil Options.Embedder takes the production path (newProductionEmbedder: embeddings.Lookup, ORT init, model download into <Home>/models). So a new test that copies the old `New(ctx, cfg, Options{APIOnly: true})` shape compiles, passes on a developer machine that has ORT and the model, and fails in CI exactly the way race.yml failed from PR #243 to #278 — or, once ./internal/app/ is in tests.yml, fails every PR.

It happened immediately: while #284 was open, PR #283 added TestApp_NoOAuthBuildsNoIssuer and TestApp_SignedApprovalWired to oauth_wiring_test.go without the seam. The merge of origin/dev into #284 only caught them because the worker grepped every `New(` call in the package; git reported no conflict for those lines.

Consequence: when adding or reviewing a test in internal/app that calls New and expects the boot to SUCCEED, pass Embedder: &testenv.DeterministicEmbedder{}. Tests that expect New to FAIL before the embedder is built (TestNew_EmbedderRequired, whose unknown model id fails at embeddings.Lookup; TestBoot_RequireWithoutLocalListenerFails, which fails at checkLocalListener) do not need it and must not have it, because the seam would skip the very failure they assert.

What would close the gap: a test-side guard in internal/app (e.g. a test that greps the package's own _test.go files for `New(` calls lacking `Embedder:` and allow-lists the two failure tests), or a helper `newTestApp(t, cfg)` that every success-path test goes through so the bare New call stops appearing in tests. Neither exists as of #284.

This is a different fact from the closers-leak gotcha: that one is about the production embedder's lifetime on a FAILED boot; this one is about tests re-acquiring the production embedder on a SUCCESSFUL boot.
