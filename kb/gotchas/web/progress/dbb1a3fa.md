---
type: observation
domain: [web, store, ui, progress]
confidence: 0.9
sources: 0
entities: [progressThrottle, rebuildEmbeddings, RebuildProgress, handleStartRebuild, handleApply]
refs: ['src://knomit/internal/web/progress_throttle.go', 'https://github.com/knomit/knomit/pull/82']
---
# Rebuild progress: forward every batch via a time throttle, never a batch-misaligned modulo gate

searchIndex.Rebuild reports progress once per 32-fact embeddings batch (done = 32, 64, ...). Consumers that re-gate these events with a fixed modulo (the old handlers used done%10 / done%20) SWALLOW almost everything: the first done that is also a multiple of 20 is 160 (LCM with the 32-fact batch), so the connect/rebuild UI sat frozen at '0/total' until ~160 facts were embedded — minutes on a slow machine. Convention: forward rebuild progress via web.progressThrottle (time-based — always emit the first and final update, rate-limit the middle to ~250ms), which is batch-size-agnostic and cannot drift out of alignment with the batch size. Used by handlers_jobs (standalone rebuild) and handlers_origin_session (connect).
