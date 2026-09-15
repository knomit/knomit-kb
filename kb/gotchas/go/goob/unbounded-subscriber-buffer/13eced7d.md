---
kind: pragmatic
type: heuristic
domain: [go, events, sse, memory, repos, web]
confidence: 0.95
sources: 2
entities: [goob.Observable, goob.NewPipe, Observable.Publish, Observable.Subscribe, TaskHub, internal/repos/hub.go, logging.Tap, Subscription.deliver, subscriberBuffer, github.com/ysmood/goob]
motifs: [nonblocking-hides-unbounded-buffer, safety-depends-on-rate]
refs: ['kb://3ec012f5b4d2/kb/architecture/repos/taskhub-four-event-types/205b9763.md', 'kb://3ec012f5b4d2/kb/decisions/web/manage/logs-tab-sse-ring/ddceb81b.md', 'src://7b4887ce51d9/internal/platform/logging/tap.go@19b63d81310c3a2502db559bd47f0abf4285038d:ab1243f4c55f4b92492187c60a71c54f3c5e045f', 'src://7b4887ce51d9/internal/repos/hub.go@19b63d81310c3a2502db559bd47f0abf4285038d:c9f20676b9ee0ad813871a3483cd21786d28dc6d', 'https://github.com/knomit/knomit/pull/199']
---
# goob.Observable never blocks Publish because each subscriber's pipe buffers into an UNBOUNDED slice — so it is safe only for LOW-RATE events, and a high-rate publisher with one stalled subscriber grows the heap with no ceiling and no signal

goob v0.4.0's `NewPipe` documents itself as "Events uses an internal buffer so it won't block Write", and it delivers that by appending every event to a plain `buf []Event` slice with NO cap and NO drop path. `Observable.Publish` walks its subscribers under the hub lock calling each one's `write`, and `write` only appends — so Publish is genuinely non-blocking, and the cost of a subscriber that has stopped draining is paid entirely in retained memory.

The pump goroutine forwards `buf` into an unbuffered channel. If the consumer stops reading — an SSE handler wedged on a socket write to a suspended laptop or a proxy that stopped draining — the pump blocks on the send and `buf` grows by one entry per published event for as long as the stall lasts. Nothing bounds it, nothing counts it, and nothing logs it.

WHAT TO DO: goob is the right tool when the EVENT RATE IS BOUNDED BY SOMETHING OTHER THAN REQUEST TRAFFIC. TaskHub (internal/repos/hub.go) qualifies — its TaskEvent/StatusEvent/SyncEvent/PushEvent fire on synth runs, HEAD changes and reconcile ticks, so a stalled browser retains a handful of small structs. Before adding a new goob hub, ask what publishes and how often. If the answer is "once per HTTP/MCP request" or "once per log line", goob is the wrong choice and the fan-out must be bounded with drops counted and reported.

logging.Tap is the worked example of the bounded alternative: a per-subscriber `chan string` of fixed capacity (subscriberBuffer = 1024), a non-blocking `select` with a `default:` that increments an atomic `dropped` counter, and an endpoint that emits the delta as a `dropped` event so a viewer never renders a gap as continuity. That shape costs a stalled subscriber a fixed ceiling instead of the process.

WHAT THIS DOES NOT MEAN: this is not "goob is unsafe, replace the existing hubs". Both current goob users are low-rate by construction and are fine; ripping them out would trade a real non-issue for churn. The property to carry forward is that goob's non-blocking guarantee is purchased with unbounded memory, so the safety of any given use is a fact about ITS PUBLISH RATE, not about goob.

Also note the guarantee is genuinely useful where it applies: because Publish cannot block, a hub in a write path (a store method, a logging chain) cannot be stalled by a slow reader. The hazard is only ever the retention, never latency.
