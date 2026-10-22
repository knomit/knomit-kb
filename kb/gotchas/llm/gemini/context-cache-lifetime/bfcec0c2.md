---
type: observation
domain: [llm, gemini, caching]
confidence: 0.9
sources: 1
entities: [GeminiAdapter, cacheEntry, liveCacheName, dropCache, isCacheMissingErr, geminiCacheTTL, geminiCacheSkew, CachedContent, GenerateContentStream, internal/engine/llm/gemini.go]
motifs: [handle-outlives-target]
refs: ['src://7b4887ce51d9/internal/engine/llm/gemini.go@0610c20083fcef9e84f42ab0311a14708bcb6193:1ddc2bc4df78775047916090574f3aa34d0c14e2', 'src://7b4887ce51d9/internal/engine/llm/gemini_cache_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:3bcd533e8ef757d8d547cbec5da0c142ff62eb1c']
---
# A memoized Gemini cached-content name is a time-bounded handle, not a stable id — and the inline retry is gated on nothing having streamed yet

`GeminiAdapter` memoizes cached-content handles keyed by sha256 of the system prompt. The handle EXPIRES: cached content is created with a 10-minute TTL (`geminiCacheTTL`), and naming content the server has already deleted does not merely lose the cache hit — it FAILS the request. The original code memoized the name forever, so every request reusing that system prompt broke once the TTL elapsed, and the fallback covered only cache *creation* errors.

Two defences, both needed:
1. `cacheMap` stores `cacheEntry{name, expires}`, preferring the server's reported `cc.ExpireTime` over the locally requested TTL. `liveCacheName` treats an entry within `geminiCacheSkew` (30s) of expiry as a MISS and evicts it, so a request cannot race the server-side deletion between the check and the call.
2. Bookkeeping can still be wrong (eviction, shorter effective TTL, restarted backend), so a stream error matching `isCacheMissingErr` drops the entry and retries with the system prompt inline.

THE RETRY IS CONDITIONAL and the conditions are the point:
- only when `cfg.CachedContent != ""` (the request actually used a cache),
- only when `accumulated == ""` — nothing was emitted to `onChunk` yet. Re-streaming after chunks reached the caller would DUPLICATE output in the caller's buffer. This is why `stream` returns its partial accumulation alongside the error rather than discarding it.

`isCacheMissingErr` is deliberately NARROW: the message must mention cached content AND a not-found/expired/permission-denied condition. A broader match would swallow rate-limit and auth failures behind a silent retry, converting a visible error into a mysterious one. A false positive costs one extra inline call; a false negative resurrects the original bug.
