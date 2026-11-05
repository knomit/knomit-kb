---
kind: pragmatic
type: policy
domain: []
confidence: 0.9
sources: 1
entities: [toolEffects, ClassifyCall, annotateFromEffects, CallEffect, ReviewModes, HypothesizeModes, reviewMode, hypothesizeMode, TestToolEffects_EveryRegisteredToolHasRows, TestModes_ListsMatchDispatch, TestToolEffects_ModeClassifications, TestToolAnnotations_ListedFromTheTable, knomit_review, knomit_hypothesize, knomit_experiment, knomit_bind]
refs: ['kb://3ec012f5b4d2/kb/decisions/mcp/review/start-resumes-live-session/e4f26b91.md', 'src://7b4887ce51d9/internal/edge/mcp/tool_effects.go@fed6e47c4ebce1fcfbbe89d3047ea0e007440918:b86b4f0d22de809366eb2c1b067d303ca7f344b2', 'src://7b4887ce51d9/internal/edge/mcp/tool_effects_test.go@fed6e47c4ebce1fcfbbe89d3047ea0e007440918:3829a3b54ceb7043e354d012c6de8ebf61855641', 'src://7b4887ce51d9/internal/edge/mcp/review.go@fed6e47c4ebce1fcfbbe89d3047ea0e007440918:62a3f41255df8cf62514032c1ffb94f07ffe494e', 'src://7b4887ce51d9/internal/edge/mcp/hypothesize.go@fed6e47c4ebce1fcfbbe89d3047ea0e007440918:fecf147f174568d546d208a3473ed556f122cb46', 'src://7b4887ce51d9/internal/edge/mcp/server.go@fed6e47c4ebce1fcfbbe89d3047ea0e007440918:b57e2de142e7e48578b8693778020c3cd32f2d2c', 'src://7b4887ce51d9/internal/edge/mcp/endpoint_tools.go@fed6e47c4ebce1fcfbbe89d3047ea0e007440918:6df3cf5b47a39628b4f85303b216b2a93ab3635f']
---
# The (tool, action) read-only versus side-effecting classification lives in ONE table, toolEffects in internal/edge/mcp/tool_effects.go; MCP annotations are derived from it and a new tool or action without a row fails a test

The (tool, action) read-only versus side-effecting classification lives in ONE table, `toolEffects` in internal/edge/mcp/tool_effects.go. MCP readOnlyHint/destructiveHint are derived from it by annotateFromEffects at registration (the server.go AddTool loop, and for the rebuilt scoped knomit_bind in endpoint_tools.go). Any approval gate must call ClassifyCall, never keep its own list.

Derivation: a tool's readOnlyHint is true only when EVERY row of the tool is read-only; its destructiveHint is true when ANY row is destructive. So knomit_hypothesize is {readOnly:false, destructive:false} (its start and answer rows are non-destructive writes) while knomit_review and knomit_experiment are destructive.

Session starts are side-effecting. A knomit_review start opens or resumes a pipeline session and can DISPLACE a session idle past the resume window even without takeover, so it needs approval (write, not destructive; takeover and answer are the destructive rows). A knomit_hypothesize start creates a session and silently displaces an active one on its branch, so it is a write too. knomit_review page and current are reads: they only read, or advance, the CALLER'S OWN session. Hypothesize current is likewise a read.

Completeness is enumerated for EVERY multi-mode tool, not only knomit_experiment. TestToolEffects_EveryRegisteredToolHasRows requires a row for every registered tool, for every value of knomit_experiment's `action` enum, and for every entry of ReviewModes (start, takeover, page, current, answer) and HypothesizeModes (start, current, answer), and rejects a row for a mode the handler does not have. The handlers switch on reviewMode / hypothesizeMode, the very functions the classifier calls, and TestModes_ListsMatchDispatch fails if either function can return a mode missing from its list; TestToolEffects_ModeClassifications pins WHICH classification each mode has. A handler that gains a mode, or a new tool or action, without a row fails a test. A registration with write:true has a non-read row and write:false has only read rows.

ClassifyCall IS handler dispatch, not caller labels. knomit_review (reviewMode): takeover (takeover:true), else start (no session_id), else page (page>0 and no response), else current (no response), else answer. knomit_hypothesize (hypothesizeMode): start (no session_id), current (current:true), else answer. knomit_experiment uses its `action` argument. A review call carrying both page and response is therefore an answer (a write). knomit_review/takeover has its own destructive row, separate from answer. knomit_bind and knomit_repos are read-only rows (bind mints a handle, writes no knowledge). knomit_experiment: list is read; open, sync, commit are writes; rollback is destructive.

Consequence: an unknown tool or unknown action returns known=false; callers must treat it as side-effecting and never grantable.

COMPLETENESS LIMIT (do not drop): ReviewModes and HypothesizeModes are maintained BY HAND next to the dispatch switch. TestModes_ListsMatchDispatch only probes the session_id, response, page, takeover and current arguments, so a new mode gated on a NEW argument would escape the completeness test. Anyone adding such a mode must add it to the list AND extend the probe grid in that test.

WHAT THIS DOES NOT MEAN: the table does not enforce anything. Enforcement is still gateBinding/gatePermission driven by the registration's `write` flag; the table classifies and annotates, and the test keeps the two in agreement.

Replay column (added with the adaptive-tools stubs): every row also states Replay (ReplayYes/ReplayNo, plus ReplayNotWith argument names), and Replayable(tool, args) reads it. Replay is NOT ReadOnly: bind, review page/current and hypothesize current are read-only rows that are not replayable. See kb/invariants/mcp/tool-effects/d21549b8.md. TestToolEffects_EveryRowStatesReplay fails a row that does not state it.
