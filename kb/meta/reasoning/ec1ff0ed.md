---
type: methodology
domain: [methodology, review, epistemics, testing]
confidence: 0.85
sources: 1
evidence_weight: 0.726027397260274
entities: [independent review, repair recursion, fix scope, cleanup, 'PR #99', 'PR #176']
motifs: [agreement-ends-examination, fix-inherits-bug-scope]
refs: ['kb://3ec012f5b4d2/kb/meta/methodology/review/fixes-are-under-reviewed/4f7028d9.md', 'kb://3ec012f5b4d2/kb/meta/methodology/review/agreement-blind-spot/185e1fa6.md', 'kb://3ec012f5b4d2/kb/meta/methodology/repair-recursion/1b6dca60.md']
---
# The defect migrates OUTWARD exactly as scrutiny CONTRACTS — whatever marks a change as settled (agreed diagnosis, "non-blocking tidy", converged design) is what shortens its review, and that is where the bug class reappears

Three separately-derived findings describe one loop, and the loop is worse than any of them alone.

WHERE THE DEFECT GOES. A fix frequently re-creates its own bug class ONE LEVEL OUT, in the code the fix reaches into — measured across three instances in one phase (requeuing partners destroyed the partners' own pairs by the identical mechanism; not-marking-uncovered still lost pairs scanned against a partial axis; deleting a declined pair let a later rescan re-mint it).

WHERE THE SCRUTINY GOES. Simultaneously, a fix receives LESS review than the code it fixes, because by the time it exists everyone has agreed what the problem was, so review collapses into "does it close the finding?". Across six rounds on one change, EVERY defect after the first round was in a fix rather than in the original code.

WHY BOTH. The same thing causes them: agreement ends examination. Once a question is settled between the participants, neither re-derives its premises.

THE COMPOSITION, which is the part worth keeping: these are not two problems but one, and they point at the SAME region. The defect moves outward from the fixed instance at the same moment the review contracts onto it. A reviewer asking "does this close the finding" is looking exactly where the next defect is not.

THE MARKERS OF SETTLEDNESS ARE THE SEARCH INDEX. Each of the three sources names a different one, and they are interchangeable in effect: an AGREED DIAGNOSIS (this is a fix, so review it as a closure), a SAFE LABEL (a reviewer called it non-blocking tidying, so nobody re-derives its blast radius — "the label is doing the reasoning"), and a CONVERGED DESIGN (we already decided this, so the premises are not re-opened). Cleanups are the worst case precisely because they carry the strongest safety label while receiving the least scrutiny.

WHAT TO DO, from the sources jointly:

1. After a fix, NAME THE NEIGHBOURS it touches and ask the ORIGINAL failure question about each — not "does the test pass", since the test was written for the first instance and will pass.
2. Review a fix AS A NEW CHANGE; ask what its own scope EXCLUDES, not only whether the reported case now behaves.
3. The person who reported a finding should not be the ONLY verifier of its fix; their attention is on the case they named.
4. Ask whether a fix should have been a REPLACEMENT or a GENERALISATION — replacement is the shape that closes one case and opens its mirror.
5. A CLEAN ROUND IS NOT A REASON TO STOP. Yield did not decay: a later round found a regression in the fix to the round before it.
6. Get a reader independent of the DESIGN, not merely of the code, and brief them on what was DECIDED rather than only on what was built.

WHAT THIS DOES NOT MEAN. Not that the fixes or cleanups were careless — they were careful, and each addressed its finding correctly; competence at the named problem is exactly what the shortened review is calibrated to, and it is not what fails. Not "write more tests around the fix" either: this is a question about SCOPE — which other code now behaves differently, and does the reason the original was wrong apply there too — which a test cannot ask, because the neighbour was never its subject. And not that adversarial review is ineffective; its yield is simply concentrated OUTSIDE the reviewer's own prior agreements.

Each source carries instances and consequences this summary does not reproduce in full; consult them before acting on any single clause.
