---
type: observation
domain: [repos, lifecycle, testing]
confidence: 0.9
sources: 1
entities: [Manager.Set, Manager.ForEach, Manager.Close, handleHALRepos, internal/repos/manager.go, LensRegistry.Close, LensRegistry.List]
motifs: [absent-vs-nil-conflated, signature-invites-misuse]
refs: ['src://7b4887ce51d9/internal/repos/manager.go@897e17edb6a2799ca5f6872ccc89fa3ff5e9e3ba:8916b69d8a5ba9cabbe5952bb7aea1e15bc713c1', 'kb://3ec012f5b4d2/kb/decisions/mcp/session-binding/catalog-tool/314b18e6.md']
---
# Manager.Set(name, nil) STORES a nil value under the name instead of deleting the key — ForEach then hands nil to every callback and Manager.Close nil-panics; latent because no production caller passes nil

Found 2026-09-16 by the worker session while writing a concurrency regression test for knomit_catalog (PR #210).

WHAT HAPPENS: Manager.Set(name, ri) (internal/repos/manager.go, ~line 495) does `m.repos[name] = ri` unconditionally after clearing byUID. With ri == nil the map now holds a nil *RepoInstance under name. ForEach iterates m.repos and does not filter nils, so every ForEach consumer that dereferences the instance panics: Manager.Close's pass-1 loop (`ri.mu.RLock()`, ~line 594) and handleHALRepos (`instances[name].IndexStatus()`) both do. The catalog tool's snapshot loop skips nil defensively, but that is one consumer of several.

WHY IT IS LATENT, NOT LIVE: no production code path calls Set with nil today — removal goes through the delete/archive lifecycle, which deletes the key. The hazard is a TEST or a future caller using Set(name, nil) as a shorthand for "remove", which reads as reasonable from the signature and is exactly what the worker did first.

CONSEQUENCE FOR A CONSUMER: do not use Set(name, nil) to remove a repo from a Manager in tests; use two real instances, or the lifecycle removal path. If this is ever fixed, the two candidate fixes are different decisions — Set deletes the key on nil (Set becomes remove-capable), or ForEach filters nils (every caller stays unaware) — and one should be chosen deliberately rather than patched at a call site.

RELATED, same PR: LensRegistry.Close() is a no-op when the registry does not own its handle, and the Manager-owned registry shares the control.db handle, so no test in another package can induce a LensRegistry.List() query failure; the not-started path is the only reachable branch of that error handling from internal/mcp.
