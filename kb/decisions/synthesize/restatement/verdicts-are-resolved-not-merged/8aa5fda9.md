---
type: observation
domain: [synthesize, prune, restatement]
confidence: 0.9
sources: 1
entities: [restatement_verdicts, MergeEntry, RetractEntry, probeAllowed]
refs: ['src://7b4887ce51d9/internal/store/migrate/repo/000018_abstraction_axis.up.sql@b57785357448cbbca9b3b43607fe33987f5522f2:91f2d8db2ac8f8c4fe810ec006da0485fa62bcc2', 'src://7b4887ce51d9/internal/synthesize/restatement.go@b57785357448cbbca9b3b43607fe33987f5522f2:431b3d5e94e95c731f4874514b657d00f8868c0c', 'https://github.com/knomit/knomit/pull/99']
---
# A restatement verdict counts as RESOLVED, not merged — a judge that retracts the redundant half has done the work the mechanism exists to buy

The shortlist is funded by a trailing rate over its own judge outcomes. The question is what counts as a success.

**Options considered.**
1. *Count merges only.* REJECTED. A judge that consolidates a restatement by RETRACTING the redundant half has done exactly the work the mechanism exists to buy. Counting only merges defunds a corpus that is succeeding by another route — and from outside, that is INDISTINGUISHABLE from "the shortlist finds nothing", so the failure is silent and self-confirming.
2. *Count any judge response.* REJECTED. A confidence update is not a resolution: both facts still stand, the restatement is still there, and nothing was bought.
3. *Count any outcome that removes the redundancy — merge OR retraction.* CHOSEN, stored as the `resolved` column.

**Consequence for anyone adding a verdict kind.** Ask whether the new outcome LEAVES BOTH FACTS STANDING. If it does, it is not a resolution however useful it is; if it does not, it must count, or the funding signal quietly punishes a working corpus.

**Why fact ids and not just paths.** Verdict rows carry content-addressed `a_fact_id`/`b_fact_id`, so "the judge kept this pair" expires STRUCTURALLY the moment either fact is edited — no hash comparison, no staleness logic. Paths ride along because every human and log line reading these rows thinks in paths.
