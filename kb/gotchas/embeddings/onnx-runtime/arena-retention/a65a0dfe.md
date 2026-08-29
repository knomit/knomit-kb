---
type: observation
domain: [embeddings, onnx, memory, performance]
confidence: 0.95
sources: 1
entities: [onnxruntime_go, session.Run, VmHWM, arena allocator, embeddinggemma]
motifs: [retained-high-water-mark]
refs: ['src://7b4887ce51d9/internal/embeddings/batching.go@31c420243f188c9e824022c996393f6f27a7e97e:40c76d0627e87e01c9cf432cbb40bb7cfb751c93', 'https://github.com/knomit/knomit/pull/175']
---
# The ONNX Runtime arena retains its high-water RSS for the process lifetime — the worst single batch sets STEADY-STATE memory, not a transient spike

ONNX Runtime's arena allocator does not return memory to the OS after a `session.Run` completes. Whatever the largest batch needed stays resident until the process exits.

Observed in production 2026-08-29: after knomit finished indexing three repos, RSS was 4203 MiB against a peak (VmHWM) of 4219 MiB — essentially identical. The memory did not come back.

TWO CONSEQUENCES, both easy to get wrong:

1. Sizing must budget for RETAINED peak, not transient peak. A batch budget does not set a spike height, it sets the server's resting footprint. knomit rests at ~4.2 GiB indefinitely after an index, not just during one.

2. Any memory measurement must run ONE PROCESS PER CONFIGURATION. A single process measuring several batch sizes in sequence reports only a monotonic high-water mark, so every configuration after the first is contaminated by the largest one before it.

WHAT THIS DOES NOT MEAN: this is not a leak and not a bug to fix — it is arena allocator behaviour working as designed. Do not go looking for the leak.
