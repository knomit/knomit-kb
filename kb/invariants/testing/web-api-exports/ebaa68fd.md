---
type: principle
domain: [web, testing, archtest, architecture]
confidence: 0.9
sources: 1
entities: [TestWebAPI_ExportsOnlyMount, TestWebAPI_ExportDetector_Discriminates, webAPIExportAllowlist, webFamilyLayout, exportedNames, apiExportProblems, webLayoutProblems, internal/edge/web/api]
motifs: [shrinking-allowlist]
refs: ['src://7b4887ce51d9/test/archtest/web_split_test.go@7dccaa090575d14aa615b452d8b9e3f5a9a92ec0:9ba7286c30a7ce6a4cecfd553a38d54ad6740e7b', 'https://github.com/knomit/knomit/pull/621', 'kb://3ec012f5b4d2/kb/conventions/go/package-boundaries/web-no-interfaces/d36fa222.md', 'kb://3ec012f5b4d2/kb/architecture/web/httpx/ec668e7b.md']
---
# Every package under internal/edge/web/api exports exactly func Mount (positional parameters, no exported Deps type), counting members reached through an alias or an embedded type; TestWebAPI_ExportsOnlyMount enforces it and limits the web family to web, hal, httpx and the planned api packages kb, manage, jobs, origins

Guard M (12a of the web split, PR #621, merged on dev as 7dccaa09; re-read there): TestWebAPI_ExportsOnlyMount in test/archtest/web_split_test.go.

RULE. Every package go list returns under internal/edge/web/api exports exactly one name, the package-level func Mount, and Mount takes positional parameters (there is no exported Deps type). Every per-package list in webAPIExportAllowlist is EMPTY; a future exported name needs a reviewed entry with a non-empty reason. The rulings behind it are in kb/decisions/web/api-packages-export-only-mount (written with this fact).

WHAT COUNTS AS AN EXPORT (exportedNames): package-level funcs, types, vars and consts; and every exported method or field reachable through an exported type T: T's own members; members reached through an alias of a local type (`type JobRegistry = jobRegistry` reaches jobRegistry's exported methods and fields); and members promoted from every local type T embeds, transitively, whether the embedded type is exported or not (pointer and generic embeds included; a field T declares itself shadows a promoted one). Exported methods of an unexported type that no exported type aliases or embeds are NOT counted, so ServeHTTP on an unexported handler is fine. Members of a type from another package that an exported type embeds are not walked; only local types are.

POLICY (apiExportProblems): Mount must be a package-level func (a var or const named Mount fails). Any other exported name fails unless listed with a reason. A listed name that is not exported fails as stale.

LAYOUT (webLayoutProblems): an api/* package must be a webAPIExportAllowlist key (so a bare `api` package and `api/zz` both fail). Any other web-family package must be a webFamilyLayout key: the parent, hal or httpx (httpx was allowed before it existed). A resource package outside api/, for example internal/edge/web/jobs, fails. Guard W (web-no-interfaces) does not cover this, because it only needs a webKnownTypes entry, which any new package can add for itself.

PENDING (PLANNED, NOT YET BUILT). The expected api packages kb, manage, jobs and origins do not exist at 7dccaa09 (created in 12c to 12f). A key whose package go list does not return allows nothing and is only logged, as is the state of no api package at all. END STATE: after 12f the pending Logf becomes an Errorf, so nothing may stay pending.

NON-VACUITY for each live api package: the file list comes from go list and the scanned count must equal go list's; the package clause must be the last element of the import path; Mount must be found. TestWebAPI_ExportDetector_Discriminates pins the detector and both policies on synthetic source (alias, embedding including transitive, pointer and generic embeds, shadowing, the ServeHTTP exemption, the layout rule).

CONSEQUENCES: a helper two api packages need goes in httpx; Mount's dependencies arrive as positional parameters; tests that touch an api package's internals live in that package.

DOES NOT MEAN: (1) a green run today proves the rule holds on real code. No api package exists yet, so only the synthetic tests and sabotage exercise it. (2) Mount's signature is policed: KNOWN GAP, not blocking, Mount may still return a type that has exported methods; a follow-up adds a 'Mount has no results' check. (3) Cross-package embedded members are walked.
