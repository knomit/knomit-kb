---
type: observation
domain: [fleet, roadmap, decisions]
confidence: 0.95
sources: 1
entities: [F11, F12, F13]
refs: ['https://github.com/knomit/knomit/pull/340']
---
# F12 (librarian) and F13 (leader lease) are postponed; a human does both jobs, and F11 was next (user roadmap ruling, 2026-09-28)

CHOICE (user, verbatim, 2026-09-28): "about the overall plan, I want to postpone for now F12 and F13 - I will use a human for both. Next up would be F11".

CONSEQUENCES applied: nothing in F11 depends on F12 or F13; the F11 design's "merging is the librarian's" became "the host never merges on push — a human merges", and how that human merges is its own decision (kb/decisions/fleet/f11/merge-path-ui-follow-up). F11 was built next (PR #340).

NON-SCOPE: postponed, not dropped. F12/F13 proposals stay in .claude/future/fleet/proposals/ for a later ruling; nothing here authorises building an automated merger or leader election.
