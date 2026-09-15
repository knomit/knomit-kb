---
type: synthesis
domain: [embeddings, store, memory, concurrency, performance]
confidence: 0.9
sources: 1
evidence_weight: 0.8245614035087719
entities: [session.Run, packByTokenBudget, max_batch_tokens, minRowCharge, batchConcurrency, EmbedQuery, EmbedDocument, rebuildEmbeddings]
motifs: [wrong-quantity-bounded]
refs: ['kb://3ec012f5b4d2/kb/invariants/embeddings/inference-memory/padded-token-budget/f84543d8.md', 'kb://3ec012f5b4d2/kb/incidents/embeddings/inference-oom/2026-08-29/9112062a.md', 'kb://3ec012f5b4d2/kb/invariants/embeddings/concurrency/shared-embedder/7288aed4.md', 'kb://3ec012f5b4d2/kb/invariants/embeddings/cancellation-checkpoint/548a4acc.md']
---
# The unit every embedding guarantee is stated in is ONE session.Run, and a run is priced by the padded token budget — memory, uninterruptible latency and the concurrency cap are all per-run, so no document count anywhere bounds anything

Every bound the embedding path offers is stated per inference run, and a run's cost is set by the padded token budget it is packed to rather than by how many documents a caller hands over, so a caller-side document count carries no guarantee and every consumer that sizes memory, timeouts or overlap must reason in the budget.

The three bounds and how each is per-run:
- MEMORY: peak RSS tracks rows × padded sequence length, and a batch is priced at its LONGEST member — one long document among short ones silently multiplies the cost of everything batched with it. `packByTokenBudget` sorts rows by charged width descending so `rows × width ≤ budget` holds by construction. The OOM this replaced was two equal constants in different packages (rebuild's document chunk and the embedder's document batch) that made every rebuild chunk exactly one maximal run; that steady state, not a rare pile-up, was the root cause, and the rebuild chunk constant was deliberately left unchanged because it no longer carries the safety story.
- LATENCY: `sess.Run` exposes no termination hook, so ctx is a CHECKPOINT observed between runs — cancelling bounds latency to one run, it does not abort one. Since batches pack to `embeddings.max_batch_tokens`, that bound scales with the setting and the documents in flight; `minRowCharge = 64` keeps it finite for callers that embed an entire request or vocabulary. The contract is UNIFORM, not improved: short-string cancellation latency grew relative to the old fixed chunk. Never size a client timeout against document count; size it against the configured token budget.
- CONCURRENCY: one shared Embedder plus per-branch locking means overlapping runs' peaks ADD, and the ONNX arena keeps the high-water mark for the process lifetime, so overlap is a permanent floor rather than a spike. Batch inference is capped at ONE concurrent run wherever a memory ceiling was detected or unreadable, or `max_batch_tokens` was raised above the default; an UNKNOWN ceiling is UNBOUNDED by deliberate choice, with a boot warning.

WHAT THIS DOES NOT MEAN: the concurrency cap is not a per-process memory bound — `EmbedQuery` and `EmbedDocument` bypass it by design so interactive search never queues behind a rebuild, each up to one MaxTokens row, unbounded in count, retained by the same arena; the cap is not free (~36% at the narrower shapes a ~10 GiB host derives); and none of this bounds unknown-ceiling hosts. 'Long documents use more memory' is true but useless — the trap is the longest member pricing the whole batch.
