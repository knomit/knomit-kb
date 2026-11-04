---
kind: pragmatic
type: policy
domain: []
confidence: 0.9
sources: 1
entities: [toolEffects, ClassifyCall, annotateFromEffects, CallEffect, TestToolEffects_EveryRegisteredToolHasRows, TestToolAnnotations_ListedFromTheTable, knomit_review, knomit_hypothesize, knomit_experiment, knomit_bind]
refs: ['src://7b4887ce51d9/internal/edge/mcp/tool_effects.go@b182acbac096bb3aaa19f29efe5125f828d27e46:18e81a622b9067c7a4755f4431231ee2b726679a', 'src://7b4887ce51d9/internal/edge/mcp/tool_effects_test.go@b182acbac096bb3aaa19f29efe5125f828d27e46:e2e27749251d2a8316bc33c5a71f5b6c0f0c0da4', 'src://7b4887ce51d9/internal/edge/mcp/server.go@b182acbac096bb3aaa19f29efe5125f828d27e46:b57e2de142e7e48578b8693778020c3cd32f2d2c', 'src://7b4887ce51d9/internal/edge/mcp/endpoint_tools.go@b182acbac096bb3aaa19f29efe5125f828d27e46:6df3cf5b47a39628b4f85303b216b2a93ab3635f']
---
# The (tool, action) read-only versus side-effecting classification lives in ONE table, toolEffects in internal/edge/mcp/tool_effects.go; MCP annotations are derived from it and a new tool or action without a row fails a test

The (tool, action) read-only versus side-effecting classification lives in ONE table, `toolEffects` in internal/edge/mcp/tool_effects.go. MCP readOnlyHint/destructiveHint are derived from it by annotateFromEffects at registration (the server.go AddTool loop, and for the rebuilt scoped knomit_bind in endpoint_tools.go). Any approval gate must call ClassifyCall, never keep its own list.

Derivation: a tool's readOnlyHint is true only when EVERY row of the tool is read-only; its destructiveHint is true when ANY row is destructive. So knomit_hypothesize is {readOnly:false, destructive:false} (its answer row is a non-destructive write) while knomit_review and knomit_experiment are destructive.

Completeness is enforced by TestToolEffects_EveryRegisteredToolHasRows: every registered tool has a row, a registration with write:true has a non-read row and write:false has only read rows, and every value of an `action` enum has a row. A new tool or action without a row fails that test.

ClassifyCall mirrors handler dispatch, not caller labels. knomit_review: takeover (takeover:true), else start (no session_id), else page (page>0 and no response), else current (no response), else answer. knomit_hypothesize: start (no session_id), current (current:true), else answer. knomit_experiment uses its `action` argument. A review call carrying both page and response is therefore an answer (a write). knomit_review/takeover has its own destructive row, separate from answer. knomit_bind and knomit_repos are read-only rows (bind mints a handle, writes no knowledge). knomit_experiment: list is read; open, sync, commit are writes; rollback is destructive.

Consequence: an unknown tool or unknown action returns known=false; callers must treat it as side-effecting and never grantable.

WHAT THIS DOES NOT MEAN: the table does not enforce anything. Enforcement is still gateBinding/gatePermission driven by the registration's `write` flag; the table classifies and annotates, and the test keeps the two in agreement.
