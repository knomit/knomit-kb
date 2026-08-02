---
type: observation
domain: []
confidence: 0.95
sources: 0
entities: [internal/metrics, metrics.Default, metricsMiddleware, knomit_http_requests_total, knomit_embed_inference_seconds, Embedder.embedBatch, observeEmbedInference]
refs: ['src://knomit/internal/metrics/metrics.go@d960e325', 'src://knomit/internal/web/observability.go@d960e325', 'src://knomit/internal/embeddings/inference_metrics.go@d960e325', 'https://github.com/knomit/knomit/pull/107', 'https://github.com/knomit/knomit/pull/28']
---
# Metrics live in a leaf internal/metrics with a process-global Default; there is currently NO SQLite contention signal

**Architecture choice:** Metrics primitives (Counter, Gauge, CounterVec, Histogram + Prometheus-text rendering) live in `internal/metrics`, a STDLIB-ONLY leaf package with a process-global `metrics.Default()` registry. Subsystems (web, embeddings) record into Default directly; `runtimeobs` `/metrics` only renders it. This decouples instrumentation from the diagnostics server — numbers are collected (atomic adds) whether or not the port is enabled, and `store`/`web` never import `runtimeobs` (which pulls in `net/http/pprof`). No `prometheus/client_golang`; histograms are hand-rolled fixed-bucket cumulative `_bucket`/`_sum`/`_count`.

**Signal choices (what to measure, and why these seams):**
- `knomit_http_requests_total` is a CounterVec labelled `route`, `method`, `status`, where `route` is the chi ROUTE PATTERN (`chi.RouteContext(r).RoutePattern()`), never the concrete path — otherwise every distinct fact/repo path becomes a new series and cardinality explodes. Implemented as `metricsMiddleware` in internal/web, which also logs slow requests at WARN past `[log].slow_request_ms`.
- `knomit_embed_inference_seconds` histogram wraps each ONNX `Embedder.embedBatch` `sess.Run` (via `observeEmbedInference`) — the highest-value latency signal since embedding runs on essentially every fact write. The helper is unit-tested without needing the ONNX runtime.

**THERE IS CURRENTLY NO SQLITE CONTENTION METRIC.** `internal/store` records nothing into the registry at HEAD. The original design used `knomit_cypher_retry_total` (incremented in `store.withCypherRetry`) as the read-side contention PROXY, because true SQLITE_BUSY is absorbed by `_busy_timeout` at the driver and rarely surfaces as a Go-level error — there was no clean `sqlite_busy` event to count, whereas the GraphQLite `cypher()` transient-collision retry was a real, measurable signal with a clean seam. The GraphQLite removal (PR #28) deleted `internal/store/cypher_retry.go`, `withCypherRetry`, and the counter along with it, and NOTHING replaced them.

WHAT THIS DOES NOT MEAN:
- Do NOT look for `knomit_cypher_retry_total` in `/metrics` output or reason about contention from it — it does not exist. The underlying gap it worked around is unchanged: `_busy_timeout` still swallows SQLITE_BUSY, so contention remains unobservable until someone instruments a new seam.
- The absence is a known gap, not a decision that contention stopped mattering. Writers still serialize on `_txlock=immediate` (see kb/invariants/store/sqlite/write-serialization/9caa7895.md).

**Testability additions:** `Counter.Value()` and `Histogram.Count()` readers exist so instrumentation sites can be asserted in unit tests via delta against `metrics.Default()`.
