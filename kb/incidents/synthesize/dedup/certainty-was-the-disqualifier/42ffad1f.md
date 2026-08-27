---
type: observation
domain: [synthesize, dedup, restatement]
confidence: 0.95
sources: 1
entities: [filterByBlendedCosine, dedupCluster, mergeFacts, refreshRestatementShortlist, clusterCoMembership, restatement_pairs, EmbedderThresholds]
motifs: [certainty-disqualifies-candidate, proxy-inverts-predicate]
refs: ['src://7b4887ce51d9/internal/synthesize/restatement.go@8c094a6b9679f3e336824fa7b2e8004da0417ae0:03b5de13c1a04811c0d4c24ac891ff1ac5743946', 'src://7b4887ce51d9/internal/synthesize/dedup.go@8c094a6b9679f3e336824fa7b2e8004da0417ae0:651f9573022a551a3c5cd9075330cef2b01821c9', 'https://github.com/knomit/knomit/issues/127', 'https://github.com/knomit/knomit/pull/153', 'kb://3ec012f5b4d2/kb/gotchas/synthesize/prune-scope/c40d6748.md']
---
# The restatement shortlist deleted its own best candidates: a pair at or above the dedup floor was dropped as "mergeFacts handles it", but mergeFacts only merges WITHIN a cluster

`filterByBlendedCosine` dropped every standing candidate pair whose stored vectors sat at or above the model's calibrated dedup floor, on the stated rationale that "mergeFacts already merges them mechanically, so spending a judge slot on one is pure waste."

That rationale is false for exactly the population the shortlist exists to serve. `dedupCluster`'s mechanical merge only ever pairs facts in the SAME cluster — every search hit is gated on cluster membership before it can become a mergePair — and the shortlist exists precisely because restatements whose halves cluster APART are judged by nothing. So a cross-cluster pair above the floor was deleted as already-handled and then handled by nothing. BEING A CERTAIN DUPLICATE WAS THE DISQUALIFIER.

MEASURED on the live core corpus before the fix (2,415 facts, 14,768 standing pairs; vectors read from the sqlite-vec chunk tables). All six confirmed duplicate pairs from the operator drain sat at blended cosine 0.83–0.97 against a floor of 0.82, and all six were ABSENT from the cache — while roughly two dozen unrelated partners per fact WERE cached, topping out at 0.77. Both cisco twins were cached against an unrelated `cisco/b3dac019` at 0.772/0.773 while their own twin at 0.969 was missing.

CONSEQUENCE, and it is the part that misleads: the reported "restatement candidates emitted: 0 / shortlist throttle: defunded" is a CONSEQUENCE of this filter, not an independent throttle defect. The shortlist was left judging only the sub-floor leftovers, which are genuinely not duplicates, so the judge correctly kept them all and the corpus defunded itself on that evidence. Do not diagnose a defunded shortlist as a throttle problem without first asking what the candidate generator removed.

FIX: the filter is gone. The exclusion it was approximating — "prune already sees this pair" — is implemented once and correctly at SELECTION time as a cluster co-membership check, which is exact and session-aware; the cosine version was a proxy for it and the proxy was inverted. An above-floor pair is also the pair a mechanical merge would handle WORST, since mergeFacts picks a winner and discards the loser's body while the judge must preserve both.

WHAT THIS DOES NOT MEAN: it is not "cosine filtering is wrong". The floor is still the right predicate for deciding whether a pair needs a JUDGE rather than a mechanical merge — it was being applied where the mechanical merge could not reach.
