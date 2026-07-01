---
kind: pragmatic
type: policy
domain: [repos, testing, remote, ci]
confidence: 0.95
sources: 0
entities: [Storyboard, config.LocalOriginRoot, validateLocalOrigin, internal/testenv/storyboard.go, initDefaultGit, internal/storytests, KNOMIT_LOCAL_ORIGIN_ROOT, BareRemoteHTTP]
refs: ['src://knomit/internal/testenv/storyboard.go@0e01c71', 'src://knomit/internal/repos/origin.go@0e01c71', kb/invariants/repos/remote/local-origin-gate/426d64a7.md]
---
# Storyboard file:// remote storytests were silently dead behind the local-origin gate (LocalOriginRoot unset); CI never ran them

The internal/testenv Storyboard built its config as `config.Config{Home: homeSub}` WITHOUT setting cfg.LocalOriginRoot (it bypasses config.Defaults()). Because the production local-origin security gate (validateLocalOrigin in internal/repos/origin.go, enforced at initDefaultGit/recoverFromOrigin/runReconcileLoop) blocks ALL local-path (file://) origins unless LocalOriginRoot is set, EVERY file:// remote fixture failed at Connect with "origin blocked by local-origin policy: set local_origin_root (or KNOMIT_LOCAL_ORIGIN_ROOT)". This made the file:// remote storytests — internal/storytests E-series (multi-agent push/sync) and G-series (reconcile DAG) — RED since the gate landed (commit 545cd95 / PR #87). The process env KNOMIT_LOCAL_ORIGIN_ROOT does NOT help — the Storyboard hand-builds config and does not read env. A secondary symptom "verify: list branches: sql: database is closed" at teardown is a pure cascade from the failed Connect re-boot, not an independent bug.

CI NUANCE (corrected): .github/workflows/ci.yml line 63 DOES invoke `go test ./internal/storytests/` (identical on master — not in the branch diff), so CI runs the storytests PACKAGE. But the file:// remote tests inside it were gate-blocked/RED, so they provided no real coverage — invoked but not effectively gating. (Earlier note "CI does not run them at all" was imprecise: the package is invoked; the remote subset was failing.)

Fix: set cfg.LocalOriginRoot = sb.homeDir in Storyboard.Repo() (internal/testenv/storyboard.go) — bare remotes live at <homeDir>/remotes/<name>, strictly under the sandbox root, so this authorizes exactly the fixtures and nothing outside (validateLocalOrigin returns early for non-local origins, so http:// remotes are unaffected). After the fix the full storytests suite is green, and because ci.yml already invokes the package, CI now exercises these tests for real. The NEW build-tagged contract cells (`go test -tags contract ./internal/storytests/`) are a SEPARATE suite NOT yet in CI — a dedicated CI step must be added once Phase-2 fixes turn them green. Discovered while building the adversarial origin-sync HTTP fault harness (branch origin-sync-harness); the HTTP keystone passed precisely because http:// origins bypass the gate.
