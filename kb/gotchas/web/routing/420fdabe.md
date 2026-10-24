---
type: observation
domain: [web, routing, testing]
confidence: 0.85
sources: 1
entities: [router.go, manage.Mount, chi, TestRouter_RouteTableMatchesGolden]
motifs: [order-dependence-assumed]
refs: ['src://7b4887ce51d9/internal/edge/web/router.go@a8bf8d7a0b0058e878b7231a0cfb744b7711dfaf:e7c0dc038e66898b5aba402bf495cfeef64449cf', 'kb://3ec012f5b4d2/kb/gotchas/web/routing/chi-static-vs-param/8e750d1e.md', 'https://github.com/knomit/knomit/pull/628']
---
# Route registration order is not fully preserved by the 12e web split: the /repos/{repo}, /branches/{branch} and /lenses/{lens} routers are built up front and manage's top-level routes register after them, which is safe only because chi matches by tree structure

Verified at a8bf8d7a (PR #628; internal/edge/web/router.go). The three shared subrouters are created before manage.Mount, so manage's top-level routes (fleet, archived, repos, repo-events and the rest) now register AFTER the /repos/{repo}, /branches/{branch} and /lenses/{lens} routers exist, and the /lenses router exists before the unscoped /mcp group. Before the move they registered in a different order. chi resolves by tree structure (static over param over wildcard), not by registration order, so no request resolves differently; the golden route table and the full-router tests confirm it.

CONSEQUENCE. Do not reason about a new route's precedence from where it sits in router.go; reason from the pattern, and keep to the no-static-sibling-of-a-user-named-param rule documented above /repo-creates in api/manage/mount.go. DOES NOT MEAN: that order never matters (middleware order on one router does, and Use must precede the first route on it).
