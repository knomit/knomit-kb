---
kind: pragmatic
type: policy
domain: [mcp, handlers]
confidence: 0.95
sources: 0
entities: [context.WithTimeout, repos.RepoFromContext, storeIndices, mcpgo.NewToolResultError]
refs: ['src://knomit/internal/mcp/query.go@307b67d', 'src://knomit/internal/mcp/learn.go@307b67d']
---
# All MCP handlers follow the same prologue: timeout, RepoFromContext, storeIndices, accessors, validate, execute

All MCP tool handlers follow the SAME prologue template: (1) ctx, cancel := context.WithTimeout(ctx, <30s|60s>); defer cancel() — 60s for learn (dedup work), 30s for everything else; (2) ri := repos.RepoFromContext(ctx) — RepoMiddleware guarantees presence, or RepoFromContextOpt for optional; (3) s := storeIndices(ri) — extract sub-indices under single WithRead; (4) agentBranch := ri.AgentBranch(); ontologyRoot := ri.OntologyRoot(); ontology := ri.Ontology() — accessor calls, no extra lock; (5) parse + validate args; (6) execute. Handlers return (*mcpgo.CallToolResult, error). CONVENTION: return mcpgo.NewToolResultError(msg) with err == nil for user-input/validation errors — reserves the Go error channel for unexpected failures only. Anti-pattern: returning a Go error for validation failure — clients see 'internal error' instead of the actual message.
