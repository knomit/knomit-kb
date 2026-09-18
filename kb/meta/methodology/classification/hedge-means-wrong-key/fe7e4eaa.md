---
kind: pragmatic
type: heuristic
domain: [methodology, design, review, classification]
confidence: 0.9
sources: 1
entities: [paramClass, TestRouter_NoStaticSiblingsOfUserNamedParams]
motifs: [hedge-signals-shape-error, one-key-two-truths]
refs: ['kb://3ec012f5b4d2/kb/invariants/web/routing/no-static-children-under-repos/ea605d86.md', 'kb://3ec012f5b4d2/kb/meta/methodology/testing/mutation-must-name-its-failure/184f4255.md', 'src://7b4887ce51d9/internal/web/router_static_children_test.go@2d36336ce0e9420d6977905436c344e1aec3d5ea:75d81328c565418c56117525220db184bc81868f', 'https://github.com/knomit/knomit/pull/224']
---
# A caveat written into a classification's own reason is evidence the KEY is at the wrong granularity — one key being asked to hold two truths — not evidence the classifier was unsure; check the key before hedging the entry

When you find yourself writing "but note that", "the same doubt applies" or "revisit this" into the REASON field of a classification entry, stop and ask whether the table is KEYED WRONG. A hedge usually means one key is being asked to carry two different answers — which is a property of the table's shape, not of your confidence.

THE ANCHOR (2026-09-18, PR #224). A routing test classified URL params by their SPELLING: is `{name}` a value a user can cause to exist? The honest answer is BOTH. At `/ontologies/presets/{name}` it is one of our shipped preset names, which no user adds to; at `/repos/{repo}/branches/{branch}/domains/{name}` it is an author-written domain tag, created by anyone who writes `domain: [stats]` in a fact's frontmatter. One key, two truths.

I wrote the caveat — "under presets these are ours; under domains they are authored tags, classified not-user-named by the same survey; the same doubt applies" — and flagged it for challenge. That was the honest move AVAILABLE INSIDE THE SHAPE I HAD, and it was still the wrong move: the table could not express the answer, so I documented my discomfort instead of fixing the container. Rekeyed by PARENT PREFIX, both entries got a decision and neither needed a hedge.

IT IS NOT A HARMLESS HEDGE. The caveat classified `/domains/{name}` as unguarded, so a static route at `/domains/stats` would have passed the guard and shipped, shadowing a real domain tag. A hedge does not suspend the entry — the code still acts on whichever value you wrote, while the prose records that you were not sure. The reader in a hurry resolves it the way that lets them proceed.

THE TEST TO APPLY, in order: (1) can I state this entry's answer without qualification? (2) if not, is the qualification about MY uncertainty, or about the key covering two cases? (3) if the latter, re-key — by parent, by context, by whatever the two cases differ in — and the hedge disappears rather than being rewritten more carefully.

GENERALISES past routing to any classification a program acts on: feature flags keyed by name when the answer differs by environment, error-handling keyed by type when it differs by call site, permissions keyed by role when they differ by resource. The smell is identical — a reason field arguing with itself.

WHAT THIS DOES NOT MEAN: not every caveat is a shape error. A genuinely uncertain entry — where more information would settle it and no re-keying helps — should say so, and say what would resolve it. The distinguishing question is whether the two readings apply to DIFFERENT INSTANCES (shape error, re-key) or to the SAME instance (real uncertainty, name the evidence you lack).
