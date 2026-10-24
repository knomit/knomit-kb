---
type: observation
domain: [web, architecture, refactoring, routing]
confidence: 0.92
sources: 1
entities: [router.go, manage.Mount, kb.Mount, TestRouter_RouteTableMatchesGolden, chi, api/manage, api/kb]
motifs: [shared-tree-single-builder]
refs: ['src://7b4887ce51d9/internal/edge/web/router.go@a8bf8d7a0b0058e878b7231a0cfb744b7711dfaf:e7c0dc038e66898b5aba402bf495cfeef64449cf', 'src://7b4887ce51d9/internal/edge/web/api/manage/mount.go@a8bf8d7a0b0058e878b7231a0cfb744b7711dfaf:7b398acaff192e090c5dfee34c41ab0cb67e0a7b', 'src://7b4887ce51d9/internal/edge/web/route_table_golden_test.go@a8bf8d7a0b0058e878b7231a0cfb744b7711dfaf:6e5c561aa1ecec5b5b45354d137e663b01c2df83', 'kb://3ec012f5b4d2/kb/decisions/web/api-packages-export-only-mount/2128cc76.md', 'https://github.com/knomit/knomit/pull/628']
---
# DECISION (user ruling 2026-10-24): router.go builds the whole shared route tree and its middleware; each api Mount takes the subrouters it registers into as positional parameters; a golden route table proves before equals after

RULING (the user, 2026-10-24, web split step 12e/12f; built in PR #628, verified at a8bf8d7a). Problem: the manage routes and the KB routes share the /repos/{repo}, /branches/{branch} and /lenses/{lens} chi subtrees, and chi panics on a pattern mounted twice, so two api packages cannot each own a subtree.

DECIDED. router.go keeps building the whole route tree and every middleware. Each api package's Mount takes the subrouters it registers into as positional parameters (manage.Mount(top, repoOuter, repo, branch, lens, rm, ...); kb.Mount(repo, branch, lens, ...) in 12f). The fleet gate moved to httpx (httpx.FleetRepoRefused, beside BranchMiddleware). Mount stays the only export and returns nothing. A route-table golden test, TestRouter_RouteTableMatchesGolden, pins that the table before the move equals the table after.

REJECTED. (1) manage owns the skeleton and takes callbacks for the KB routes (invites the callee to own middleware it cannot see the other half of). (2) manage registers top-level routes only (leaves the /repos/{repo} lifecycle routes behind and splits one resource across two places). (3) merging 12e into 12f (one very large, unreviewable move).

CONSEQUENCE. A middleware on a shared subtree is added in router.go before Mount, never inside an api package; an api package that needs a gate receives it as a parameter (operator for manage). See kb/architecture/web/api-manage for the built shape.

DOES NOT MEAN: that Mount may take any number of parameters (the 2128cc76 ruling still asks the PR to flag about 5; the subrouters are the user-approved exception for manage).
