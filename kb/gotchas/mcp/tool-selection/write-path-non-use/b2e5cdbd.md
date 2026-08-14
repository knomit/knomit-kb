---
type: observation
domain: [mcp, claude-code, tools, integrations]
confidence: 0.9
sources: 1
entities: [toolRegistrations, ToolSearch, knomit_learn, knomit_query, UserPromptSubmit, stream-json, internal/mcp/server.go]
refs: ['src://7b4887ce51d9/internal/mcp/server.go@996f2d5019a0ad54fb0ea9b6591e979adbfa62c2:e6f3fd47d1f39602e2cd091b5a6c10e747d73258', 'src://7b4887ce51d9/internal/mcp/instructions.go@996f2d5019a0ad54fb0ea9b6591e979adbfa62c2:25629d1f6ff3bd71012aadbb0e6c2f9ef9e02953']
---
# knomit's failure mode under Claude Code is NON-USE of the write path (~20-37% call rate), not misrouting between servers — misrouting never occurred in 630 measured calls

Measured 2026-08-14 by driving real Claude Code headlessly (`claude -p --mcp-config --strict-mcp-config --output-format stream-json`) against two knomit-shaped MCP servers serving knomit's REAL tool schemas dumped from `toolRegistrations()`. 342 raw-API calls plus 288 Claude Code sessions, across four tool-description conditions and two server-naming regimes.

**Misrouting did not occur. Not once.** Every time knomit was called, the correct server was chosen — 29/29, 34/34, 30/30, 38/38 across conditions. This holds with no tool-description headline at all, and with opaque server keys (`knomit-1`/`knomit-2`) that carry no subject.

**The failure is that knomit is never called.** Split by request intent:

| condition | READ-intent call rate | WRITE-intent call rate |
|---|---|---|
| no headline | 83% | 20% |
| headline with exclusion clauses | 100% | 10-23% |
| headline, keyword-dense | 100% | 37% |

READ intent = the request asks a question ("what do we know about X", "look up Y", "why does Z"). WRITE intent = the request asks to store something ("remember that X", "record that Y", "save this").

**Non-use is deterministic per request shape**, not sampling noise: every prompt scored 3/3 or 0/3, never split. Requests with no subject vocabulary at all ("Remember that for next time", "Record what we figured out just now") failed 100% in every condition — twelve consecutive failures with no variation.

**Consequence to act on:** for a memory system this is the worst place to have a hole. A misrouted fact is in the wrong repo but recoverable; a fact never written does not exist, and neither the user nor the model finds out, because "Remember that X" gets an identical acknowledgement either way. Effort spent disambiguating between servers is effort spent on a failure that does not happen. The lever that DOES reach the write path is a `UserPromptSubmit` hook — knomit has no such hook today (only SessionStart, two PostToolUse, PreCompact).

**Mechanism:** Claude Code defers MCP tool schemas and retrieves them via ToolSearch (observed directly in the stream-json event sequence). So a tool description is the retrieval corpus that determines whether knomit is FOUND, not merely text read when choosing between servers. A read request pulls knomit in because the model goes looking for an information source; a write request pulls nothing in, because there is no question to answer.

**This does NOT mean** tool descriptions are unimportant — they took the read path from 83% to 100%. It means they are a retrieval lever, not a routing lever, and they do not reach the write path at all.

**Statistical caveat:** effect sizes are directional, not precise. The identical condition scored 32% then 38% overall non-use on two runs; run-to-run variance at n=72 is substantial. Trust the directions, not the percentages.

Full evidence, harness and raw data: `.claude/plans/2026-08-14-routing-and-integration-findings.md` and `.claude/plans/2026-08-14-routing-eval/` (local-only; `.gitignore:56` ignores `.claude/*`).
