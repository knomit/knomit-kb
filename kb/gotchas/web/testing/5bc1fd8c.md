---
type: observation
domain: [web, testing, go, ci]
confidence: 0.9
sources: 1
entities: [runtime.FuncForPC, mwFingerprint, GOTOOLCHAIN, go1.26.9, TestRouter_RouteTableMatchesGolden]
motifs: [measured-environment-differs, toolchain-dependent-name]
refs: ['src://7b4887ce51d9/internal/edge/web/route_table_golden_test.go@a8bf8d7a0b0058e878b7231a0cfb744b7711dfaf:6e5c561aa1ecec5b5b45354d137e663b01c2df83', 'https://github.com/knomit/knomit/pull/628']
---
# Go 1.26 prepends the inlining chain to runtime.FuncForPC names and 1.27 does not, so a middleware fingerprint must keep the LAST dotted segment of a plain func or closure and Receiver.Method-fm of a method value; cutting at the first dot passed on a laptop and failed all 189 routes in CI

Found in PR #628 (verified at a8bf8d7a: mwFingerprint in internal/edge/web/route_table_golden_test.go). Under Go 1.26 the name runtime.FuncForPC returns for a function that was inlined into its caller carries the inlining chain: `outer.func10.inner`, for example `(*Server).NewAPIRouter.(*Server).authEdge.authMiddleware`. Go 1.27 returns just the function's own name. A route-table fingerprint that cut at the FIRST dot therefore matched on a 1.27 laptop and mismatched on every one of 189 routes in CI, which runs go1.26.9.

WHAT HOLDS. mwFingerprint keeps only the last dot-separated segment of a plain func or closure, and for method values (`-fm` suffix) keeps `Receiver.Method-fm`, dropping only the package. Verify any change to it under BOTH toolchains: `GOTOOLCHAIN=go1.26.9 go test ./internal/edge/web -run TestRouter_RouteTableMatchesGolden` (CI's version) and the local one; `-gcflags=all=-l` (no inlining) is the other axis.

CONSEQUENCE. Any test that compares runtime function names must be checked under the CI toolchain, not only the laptop's. A green local run says nothing about it.

DOES NOT MEAN: that the CI version is fixed forever (it is pinned in the CI workflow; re-check there).
