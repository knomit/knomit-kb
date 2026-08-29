---
type: observation
domain: [embeddings, onnx, performance, measurement]
confidence: 0.85
sources: 1
entities: [session.Run, batchConcurrency, ORT intra-op parallelism, KNOMIT_BENCH_BUDGET, concurrency_bench_test.go]
motifs: [uncontrolled-variable-misread]
refs: ['src://7b4887ce51d9/internal/embeddings/embedder.go@f6f82e4ac338b5e73aa5e5a07c8c093ce98176f6:ab36c4e9cf45bc1902d89c42a04c7052fa9a646b', 'kb://3ec012f5b4d2/kb/invariants/embeddings/concurrency/shared-embedder/7288aed4.md', 'kb://3ec012f5b4d2/kb/meta/measurement/embedding-memory-harness/6f13be80.md', 'https://github.com/knomit/knomit/pull/176']
---
# Whether concurrent ONNX runs buy anything depends on batch WIDTH — a wide batch already saturates the cores, so serializing it is nearly free; a narrow one leaves headroom worth ~36%

ONNX Runtime parallelizes a single `session.Run` across cores intra-op. Whether a SECOND concurrent run buys wall-clock therefore depends on how much of the machine the first one already uses, which is a function of batch WIDTH.

Measured on an 8-core host, 2 workers, identical total work, ONE build, varying only the shape (`.claude/plans/2026-08-29-embed-memory-results-4.txt`):

	2 batches of 4x2048 ( 910 MiB)  57.71s concurrent vs 78.54s serialized  ratio 1.36
	1 batch  of 8x2048 (1820 MiB)  65.42s concurrent vs 65.25s serialized  ratio 1.00

CONSEQUENCE: the cost of bounding batch concurrency is not a constant. It is ~0% for hosts whose derived budget produces wide batches and ~36% for hosts whose smaller budget produces narrow ones — so a memory-constrained host pays MORE for the same guarantee. Do not quote a single percentage for 'the cost of serializing'.

HOW THIS WAS GOT WRONG THE FIRST TIME, because the failure is more reusable than the number. An earlier benchmark appeared to show the same effect and did not: it HARDCODED its budget, so its 'rows=4' and 'rows=8' configurations both produced identical 4x2048 batches and the only variable was worker count. The width story was asserted from it in three code comments, a commit message and a PR body, and had to be withdrawn as unmeasured before being restated from the clean pair above. A benchmark that does not print the SHAPE it ran cannot be read safely later; the harness now emits rows, budget, batch count and rows-per-batch for exactly this reason.

WHAT THIS DOES NOT MEAN: the ratios are not stable constants. The same configuration measured 1.02 on an earlier build and 1.36 on a clean one, so treat magnitudes as indicative and re-measure on one build before relying on them. `results-3.txt` in that directory is marked unreliable for this reason: its two rows came from different builds and its serialized arms differ by 21% on identical work.
