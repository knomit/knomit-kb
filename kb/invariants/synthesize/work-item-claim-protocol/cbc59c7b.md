---
type: principle
domain: [synthesize, store, concurrency, idempotency]
confidence: 0.95
sources: 0
entities: [AnswerPipelineWorkItem, NextPipelineWorkItem, Pipeline.ContinueSessionForItem, Strategy.Decode, Strategy.Apply, normalizeFactPath, pipeline_work_items]
motifs: [check-then-act-race]
refs: ['kb://3ec012f5b4d2/kb/architecture/store/pipeline-session-phase-cas/30657f2b.md', 'kb://3ec012f5b4d2/kb/decisions/synthesize/one-engine-two-drivers/19c69b3f.md', 'src://7b4887ce51d9/internal/store/pipeline_index.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:15299d59a34b2d62be8d508a72093018818a3779#L402-L450', 'src://7b4887ce51d9/internal/store/pipeline_index.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:15299d59a34b2d62be8d508a72093018818a3779#L536-L546', 'src://7b4887ce51d9/internal/synthesize/pipeline.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:00021a0c2205aa617bfd034a7f4f6cbcb5b0df4d#L628-L655', 'src://knomit/internal/synthesize/review_claim_test.go@d960e325']
---
# Pipeline work items are claimed by CAS, and the response is validated BEFORE the claim — never apply before claiming

The synthesis engine processes a work item in exactly this order, and the order is load-bearing:

1. PEEK the item (NextPipelineWorkItem).
2. DECODE + VALIDATE the response against that item (parse JSON, validate paths/schema). A failure here returns WITHOUT touching the item, so the item stays unclaimed and fully retryable. This is the common failure class — malformed LLM JSON, unknown paths — and it MUST stay retryable.
3. CLAIM-AND-MARK in one CAS: `UPDATE pipeline_work_items SET response = ?, applying = 1 WHERE id = ? AND response IS NULL` (store.AnswerPipelineWorkItem, which returns `n == 1`). rows == 0 means another caller already answered it: a BENIGN NO-OP, not an error, matching AdvancePipelineSessionPhase's convention.
4. APPLY the mutations, only on a won claim. The claimed item HOLDS THE SESSION'S PHASE (`applying = 1`) from the claim until its apply returns: `AdvancePipelineSessionPhase` refuses to advance while any item of the session has `response IS NULL OR applying = 1`, so nobody moves the session past follow-up items the apply is still enqueueing.
5. FINISH the item (`FinishPipelineWorkItem` sets `applying = 0`), explicitly after apply and also from a deferred call on every other exit, panics included, because an item left applying would hold the phase for good. A second caller who finds an applying item gets 'item N is being applied by another caller; retry shortly, or start with takeover:true' (`ApplyingPipelineWorkItem`).

WHY: distill and merge mint a FRESH UUID filename per apply (normalizeFactPath), so applying twice writes duplicate synthesized facts rather than converging. Before this protocol, review applied then marked (duplicates on resubmit or on a crash between the two) while hypothesize marked then applied, and the peek was a bare SELECT paired with a blind response UPDATE, so two racing continues both peeked and both applied.

ACCEPTED FAILURE MODE, stated so nobody 'fixes' it: a hard failure DURING apply loses that one item's decisions — the item is already consumed and is not retried. This is deliberate. The corpus is left un-maintained, never invalid, and per-fact apply failures already degrade to warnings. The alternative failure mode is duplicate synthesis facts, which is corpus corruption. Do NOT add a claim-lease with retry to 'fix' this: retry is only safe if apply is idempotent, and it is not, so lease+retry reintroduces exactly the duplication this protocol exists to prevent.

NOT WHAT THIS MEANS: the CAS is not a lock and takes no lease — there is no reclaim path, no expiry, and a consumed item never returns to the queue.

NOTE ON NAMES: the claim primitive is `AnswerPipelineWorkItem`. The pre-protocol blind setter (`SetPipelineWorkItemResponse`) no longer exists — it is named above only as the thing this protocol replaced. `pipeline_work_items` lives in the ephemeral session DB (pi.sessionDB), not the main store DB.

Updated 2026-10-12 (drift sweep against HEAD 2104fd6d): the claim SQL above lacked `applying = 1` and the body omitted the phase hold. At HEAD AnswerPipelineWorkItem is `UPDATE pipeline_work_items SET response = ?, applying = 1 WHERE id = ? AND response IS NULL` (internal/store/pipeline_index.go:536-540); AdvancePipelineSessionPhase additionally requires `advancing = 0` and NOT EXISTS an item with `response IS NULL OR applying = 1` (lines 402-410); FinishPipelineWorkItem clears the flag (lines 432-437); the engine finishes explicitly and via defer (internal/synthesize/pipeline.go:633-651). The CAS is still not a lease: `applying` blocks phase advance and concurrent continues but never returns a consumed item to the queue, so the 'do NOT add lease+retry' warning stands. Misreading to avoid: 'applying = 1' does not make a failed apply retryable; the item is consumed either way.
