---
type: principle
domain: [synthesize, store, concurrency, idempotency]
confidence: 0.95
sources: 0
entities: [AnswerPipelineWorkItem, NextPipelineWorkItem, Pipeline.ContinueSessionForItem, Strategy.Decode, Strategy.Apply, normalizeFactPath, pipeline_work_items]
refs: ['src://knomit/internal/store/pipeline_index.go@d960e325', 'src://knomit/internal/synthesize/pipeline.go@d960e325', 'src://knomit/internal/synthesize/review_claim_test.go@d960e325', kb/architecture/store/pipeline-session-phase-cas/30657f2b.md, kb/decisions/synthesize/one-engine-two-drivers/19c69b3f.md]
---
# Pipeline work items are claimed by CAS, and the response is validated BEFORE the claim — never apply before claiming

The synthesis engine processes a work item in exactly this order, and the order is load-bearing:

1. PEEK the item (NextPipelineWorkItem).
2. DECODE + VALIDATE the response against that item (parse JSON, validate paths/schema). A failure here returns WITHOUT touching the item, so the item stays unclaimed and fully retryable. This is the common failure class — malformed LLM JSON, unknown paths — and it MUST stay retryable.
3. CLAIM-AND-MARK in one CAS: `UPDATE pipeline_work_items SET response = ? WHERE id = ? AND response IS NULL` (store.AnswerPipelineWorkItem, which returns `n == 1`). rows == 0 means another caller already answered it: a BENIGN NO-OP, not an error, matching AdvancePipelineSessionPhase's convention.
4. APPLY the mutations, only on a won claim.

WHY: distill and merge mint a FRESH UUID filename per apply (normalizeFactPath), so applying twice writes duplicate synthesized facts rather than converging. Before this protocol, review applied then marked (duplicates on resubmit or on a crash between the two) while hypothesize marked then applied, and the peek was a bare SELECT paired with a blind response UPDATE, so two racing continues both peeked and both applied.

ACCEPTED FAILURE MODE, stated so nobody 'fixes' it: a hard failure DURING apply loses that one item's decisions — the item is already consumed and is not retried. This is deliberate. The corpus is left un-maintained, never invalid, and per-fact apply failures already degrade to warnings. The alternative failure mode is duplicate synthesis facts, which is corpus corruption. Do NOT add a claim-lease with retry to 'fix' this: retry is only safe if apply is idempotent, and it is not, so lease+retry reintroduces exactly the duplication this protocol exists to prevent.

NOT WHAT THIS MEANS: the CAS is not a lock and takes no lease — there is no reclaim path, no expiry, and a consumed item never returns to the queue.

NOTE ON NAMES: the claim primitive is `AnswerPipelineWorkItem`. The pre-protocol blind setter (`SetPipelineWorkItemResponse`) no longer exists — it is named above only as the thing this protocol replaced. `pipeline_work_items` lives in the ephemeral session DB (pi.sessionDB), not the main store DB.
