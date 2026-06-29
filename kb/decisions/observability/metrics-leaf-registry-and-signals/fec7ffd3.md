---
type: observation
domain: []
confidence: 0.95
sources: 0
entities: [internal/metrics, metrics.Default, metricsMiddleware, knomit_http_requests_total, knomit_cypher_retry_total, knomit_embed_inference_seconds, store.withCypherRetry, Embedder.embedBatch]
refs: ['src://knomit/internal/metrics/metrics.go@5987e9b', 'src://knomit/internal/web/observability.go@5987e9b', 'src://knomit/internal/store/cypher_retry.go@5987e9b', 'src://knomit/internal/embeddings/inference_metrics.go@5987e9b', 'https://github.com/p8a/knomit/pull/107']
---
# Metrics live in a leaf internal/metrics with a process-global Default; cypher-retry is the SQLite contention proxy (not sqlite_busy)

**Architecture choice:** Metrics primitives (Counter, Gauge, CounterVec, Histogram + Prometheus-text rendering) live in `internal/metrics`, a STDLIB-ONLY leaf package with a process-global `metrics.Default()` registry. Subsystems (store, web, embeddings) record into Default directly; `runtimeobs` `/metrics` only renders it. This decouples instrumentation from the diagnostics server — numbers are collected (atomic adds) whether or not the port is enabled, and `store`/`web` never import `runtimeobs` (which pulls in `net/http/pprof`). No `prometheus/client_golang`; histograms are hand-rolled fixed-bucket cumulative `_bucket`/`_sum`/`_count`.

**Signal choices (what to measure, and why these seams):**
- `knomit_http_requests_total` is labelled by the chi ROUTE PATTERN (`chi.RouteContext(r).RoutePattern()`), never the concrete path — otherwise every distinct fact/repo path becomes a new series and cardinality explodes. Implemented as `metricsMiddleware` in internal/web, which also logs slow requests at WARN past `[log].slow_request_ms`.
- `knomit_cypher_retry_total` (incremented in `store.withCypherRetry`) is used as the read-side SQLite CONTENTION proxy because true SQLITE_BUSY is absorbed by `_busy_timeout` at the driver and rarely surfaces as a Go-level error — there is no clean `sqlite_busy` event to count. The GraphQLite cypher() transient-collision retry IS a real, measurable contention signal with a clean seam, so it was instrumented instead.
- `knomit_embed_inference_seconds` histogram wraps each ONNX `Embedder.embedBatch` `sess.Run` (via `observeEmbedInference`) — the highest-value latency signal since embedding runs on essentially every fact write. The helper is unit-tested without needing the ONNX runtime.

**Testability additions:** `Counter.Value()` and `Histogram.Count()` readers exist so instrumentation sites can be asserted in unit tests via delta against `metrics.Default()`.
