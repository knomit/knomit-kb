---
type: observation
domain: [web, sse, desktop]
confidence: 0.85
sources: 1
entities: [internal/edge/web/httpx/sse.go, sseStream.Write, internal/edge/web/api/manage/handlers_repo_events.go, internal/edge/web/api/kb/handlers_events.go, internal/edge/web/handlers_logs.go, Last-Event-ID, ready, WailsEventSource, serveSSEStream]
refs: ['src://7b4887ce51d9/internal/edge/web/httpx/sse.go@ee850fd6cef7c936fc81af311371e34a6971350a:5e89e7974c537fa2a4578323e829e5d717b8dc92', 'src://7b4887ce51d9/internal/edge/web/api/manage/handlers_repo_events.go@ee850fd6cef7c936fc81af311371e34a6971350a:82588776070ecef147bd488b007bad59655137df', 'src://7b4887ce51d9/tools/desktop/uistreams.go@ee850fd6cef7c936fc81af311371e34a6971350a:6f96882cdac7b69b290197bc2db73c0918b4e235', 'kb://3ec012f5b4d2/kb/architecture/web/sse/fleet-index-stream/8e878292.md', 'kb://3ec012f5b4d2/kb/gotchas/web/sse/unbounded-write/1ec178a3.md']
---
# knomit's SSE handlers emit no id: or retry: field and nothing reads Last-Event-ID — reconnect continuity is the ready frame written on every connect plus a client refetch, so events published while a stream is down are lost

Re-read at dev ee850fd6: every SSE handler under internal/edge/web (api/kb events, api/manage repo-events, api/jobs job events, client-sessions, logs, and the origin-session stream, which writes bare `data:` lines) writes only `event:` and `data:` lines through httpx's sseStream.Write (internal/edge/web/httpx/sse.go), plus the `: keepalive` comment; a grep of internal/ finds no `id:` or `retry:` output and no server-side use of Last-Event-ID. Continuity after a drop is designed differently: the repo-events, client-session and logs streams write `event: ready` on every connect (the kb events stream writes an `event: status` head frame and a job stream writes the job's current `task` frame instead, neither is a replay), and the client refetches on a later ready (kb 8e878292), so nothing needs a replayable event id.

The desktop adapter forwards Last-Event-ID anyway when it has an id (uistreams.go sets the header; the page sends last_event_id), so it is ready for a server that adds ids; today that path carries nothing and resume works only because of the ready-and-refetch design. Verified live on macOS: after a forced upstream drop the resumed ready reached the page and the UI re-read /repos about 100 ms later.

CONSEQUENCE: do not build a feature on missed-event replay ("the stream resumes where it left off"); a new SSE endpoint that wants resumability must add id: AND Last-Event-ID handling server-side, and it works for the browser build and the desktop adapter alike. WHAT THIS DOES NOT MEAN: reconnects are not lossless, they are self-healing by refetch; and a stream that is merely quiet is not dropped (keepalive comments every 30 s).
