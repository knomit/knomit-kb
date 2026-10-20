---
type: observation
domain: [store, sessions, refactor]
confidence: 0.9
sources: 1
entities: [internal/store/internal/sessions, sessions.Open, sessions.PipelineIndex, sessions.ToolIndex, sessions.ReapIdle, sessions_conn.go, PipelineIndex, ToolSessionIndex]
refs: ['kb://3ec012f5b4d2/kb/architecture/store/tool-sessions-unified-schema/be09558e.md', 'kb://3ec012f5b4d2/kb/architecture/store/pipeline-session-phase-cas/30657f2b.md', 'kb://3ec012f5b4d2/kb/invariants/store/transactions/212e7e44.md']
---
# store/internal/sessions owns the ephemeral session DB (open, schema, reap), the pipeline index and the tool index; the Service wiring, published interfaces and public-name re-exports stay in package store

Step 06b of the internal/ hierarchy stack moved pipeline_index.go, tool_index.go and session_schema.sql out of package store into internal/store/internal/sessions, with the session-DB open (was openSessionDB), SessionDBPathFor/SessionDBSuffix/IsSessionDBFile (now PathFor/DBSuffix/IsDBFile) and the SQL body of Service.ReapIdleSessions (now sessions.ReapIdle).

What the child exports: Open, PathFor, DBSuffix, IsDBFile, NewPipelineIndex(mainDB, sessionDB), NewToolIndex(sessionDB), ReapIdle, the row types (PipelineSession, PipelineWorkItem, PipelineSessionStats, ToolSession, QueueItem) and ErrPipelineSlotChanged. Package store re-exports the row types as aliases and the sentinel/consts/funcs in sessions_conn.go, so no caller outside store changed.

What STAYED in store: Service fields pi/ti/sessionDB/sessionDBPath and their construction order in Open, Service.Pipeline()/ToolSession(), Service.ReapIdleSessions (a nil-check plus a call), Service.Close (closes the session DB and removes its files), and interfaces.go (PipelineIndex/ToolSessionIndex are the published contract; interface_guards.go checks the child types satisfy them). Tests that go through a Service (session_db_test.go: relocation out of the main DB, reap through Service) stayed too; the six pipeline_* tests that touch only the session DB moved with a Service-free helper.

Boundary invariant kept: PipelineIndex holds BOTH databases. Watermark methods use the MAIN db through sqlite.Conn (so a ctx-carried tx participates); session and work-item methods use the session DB directly and must never route through Conn/BeginTxIfNeeded, because a ctx-carried *sql.Tx is bound to the main DB. The child imports only internal/sqlite (sessions -> sqlite, recorded in sameLayerEdges in test/archtest/layers_test.go).

WHAT THIS DOES NOT MEAN: the child is not the owner of the main-DB pipeline_watermarks table schema (that is migration 000001 in migrate/repo); it only reads and writes it.
