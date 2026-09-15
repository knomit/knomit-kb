---
type: observation
domain: [web, ui, manage, logging, desktop, observability]
confidence: 0.9
sources: 1
entities: [GET /api/v1/logs/events, logging.Tap, crashdump.RingWriter, handlers_logs.go, ManageLogs.tsx, LogView.tsx, logStore.ts, tools/desktop/logstream.go, tools/desktop/internal/logtail, configInjectingHandler, isServerPage, Server.ReadOnly, goob.Observable, 'PR #199']
motifs: [notification-not-payload, single-mount-for-api]
refs: ['kb://3ec012f5b4d2/kb/decisions/web/manage/rail-only-for-entity-pages/e717ac81.md', 'kb://3ec012f5b4d2/kb/decisions/web/sessions/live-refresh-sse/bc1e2815.md', 'src://7b4887ce51d9/tools/desktop/logstream.go@d3a005be3c7e2a55448fb792f8dfb9869281742e', 'src://7b4887ce51d9/tools/desktop/internal/logtail/logtail.go@d3a005be3c7e2a55448fb792f8dfb9869281742e', 'src://7b4887ce51d9/internal/platform/logging/logging.go@d3a005be3c7e2a55448fb792f8dfb9869281742e', 'src://7b4887ce51d9/internal/platform/crashdump/ring.go@d3a005be3c7e2a55448fb792f8dfb9869281742e', 'src://7b4887ce51d9/tools/desktop/app.go@d3a005be3c7e2a55448fb792f8dfb9869281742e']
---
# The Manage Logs tab streams the server log over SSE at GET /api/v1/logs/events from an in-process bounded tap (ring replay + live lines, hand-rolled bounded fan-out with counted drops, NOT goob), on the same API mount the desktop uses for everything else — not Wails IPC, not a WebSocket, not a file tail, and refused on the read-only demo

**Context (2026-09-15):** the maintainer wants the desktop app's Logs window moved into the web UI as a Manage tab. For now it is the log tail, named Logs (an Audit trail is a later, different thing). The maintainer's one constraint: 'in the desktop app we should use the exact mounting point we currently have for the rest of the API, because the logs stream IS part of the API.'

**Options considered:**
1. Keep Wails IPC (desktop:log event) and add a web-only path beside it. Rejected: violates the constraint; two transports for one stream.
2. WebSocket under /api/v1. Rejected: logs are one-way; SSE is the house pattern (branch events, session events) with EventSource reconnect for free, and a plain GET works through every mount the UI is served from (tools/desktop/logstream.go records that the wails:// scheme handler cannot carry a WebSocket upgrade).
3. Tail the log FILE server-side, as the desktop tailer does. Rejected: the tailer's reasons (history from before the window opened, a previous run's failed startup) do not apply to a web tab, which can only show a RUNNING server; and a bare `knomit serve` may have no file sink.
4. CHOSEN: an in-process tap in internal/platform/logging (io.Writer in the zerolog writer chain, model crashdump.RingWriter) that retains a bounded ring of recent lines and fans each new line out to subscribers; GET /api/v1/logs/events replays the ring (`line` events), sends `ready`, then streams; 403 'Read-only instance' on the demo and the tab hidden under readOnly; wired at both logger construction sites (cmd/serve.go and tools/desktop/logging.go) so the desktop's in-process server serves it through the API base that /config.js injects.

**Fan-out is hand-rolled and bounded, not goob (PR #199):** goob.Observable's per-subscriber pipe buffers into an unbounded slice. For task and session events the rate is low enough not to matter; for a log it means a stalled browser grows the server heap by every line the process writes. The tap therefore uses its own per-subscriber bounded queue; when a subscriber's queue is full the oldest lines are dropped, the drop count is kept per subscriber and emitted to that subscriber as a `dropped` event so the viewer never silently pretends continuity. The SSE handler also sets a per-write deadline and returns on any write error (same rule as the sessions stream), so a client that stops draining ends its subscription rather than wedging the handler.

**Rationale:** meets the constraint literally (the stream is an /api/v1 resource reached via apiUrl()), reuses the SSE handler pattern, and keeps the format the viewer parses (`<RFC3339> <LVL> <message>`, the file format) as a pinned contract.

**Consequences / non-scope:** the desktop Logs window, logstream.go and logtail become redundant and are retired in a separate PR; the maintainer accepted that the log becomes readable by any local process that can reach the API port, the concern the IPC design had avoided. This decision does NOT make the tab an audit trail: it shows zerolog lines, nothing structured, and 'Audit' waits for a real who-did-what record. Do not read 'the house pattern is goob' into this fact: goob is fine for low-rate change pings, and is the wrong buffer for anything a client can fall behind on.
