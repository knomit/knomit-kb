---
type: observation
domain: [web, sse, desktop]
confidence: 0.85
sources: 1
entities: [internal/web/sse.go, sseStream.Write, handlers_repo_events.go, handlers_events.go, handlers_logs.go, Last-Event-ID, ready, WailsEventSource, serveSSEStream]
refs: ['src://7b4887ce51d9/internal/web/sse.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:85d6814115a1c34e047af6e2bfe5251ca61b1e06', 'src://7b4887ce51d9/internal/web/handlers_repo_events.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:862e39a6b90673ead64e0b50a508e37bebe79829', 'src://7b4887ce51d9/tools/desktop/uistreams.go@2104fd6d5e0fe0d6a1bdfbad4c75a973a539a62d:6f96882cdac7b69b290197bc2db73c0918b4e235', 'kb://3ec012f5b4d2/kb/architecture/web/sse/fleet-index-stream/8e878292.md', 'kb://3ec012f5b4d2/kb/gotchas/web/sse/unbounded-write/1ec178a3.md']
---
# knomit's SSE handlers emit no id: or retry: field and nothing reads Last-Event-ID — reconnect continuity is the ready frame written on every connect plus a client refetch, so events published while a stream is down are lost

At 2104fd6d: every SSE handler under internal/web writes only `event:` and `data:` lines through sseStream.Write, plus the `: keepalive` comment; a grep of internal/ finds no `id:` or `retry:` output and no use of Last-Event-ID. Continuity after a drop is designed differently: each connect writes `event: ready` after subscribing (repo-events, session events, logs), and the client refetches on a later ready (kb 8e878292), so nothing needs a replayable event id.

The desktop adapter forwards Last-Event-ID anyway when it has an id (uistreams.go sets the header; the page sends last_event_id), so it is ready for a server that adds ids; today that path carries nothing and resume works only because of the ready-and-refetch design. Verified live on macOS: after a forced upstream drop the resumed ready reached the page and the UI re-read /repos about 100 ms later.

CONSEQUENCE: do not build a feature on missed-event replay ("the stream resumes where it left off"); a new SSE endpoint that wants resumability must add id: AND Last-Event-ID handling server-side, and it works for the browser build and the desktop adapter alike. WHAT THIS DOES NOT MEAN: reconnects are not lossless, they are self-healing by refetch; and a stream that is merely quiet is not dropped (keepalive comments every 30 s).
