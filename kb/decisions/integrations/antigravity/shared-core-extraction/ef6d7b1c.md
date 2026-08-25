---
type: observation
domain: [integrations, antigravity, claude-code, bridge, refactoring]
confidence: 0.95
sources: 1
entities: [tools/bridge/knomitapi, tools/bridge/skills, knomitapi.SessionContext, skills.FS, claude/hook_session_start_test.go, ServerKey]
motifs: [parallel-implementations-diverge]
refs: ['src://7b4887ce51d9/tools/bridge/knomitapi/session.go@687fb378dcb0e064498cd24b0e0baa3786567a34:0e324542cb97fd96693ebfe1fd7a2a2f3c0614cd', 'src://7b4887ce51d9/tools/bridge/skills/skills.go@687fb378dcb0e064498cd24b0e0baa3786567a34:042e87f88792371db0e308c66331311d7a55f923']
---
# The second agent host was built by EXTRACTING a shared core from the shipped Claude Code host, not by duplicating it — with a zero-diff test as the blast-radius gate

**Options considered.** (1) Extract the host-neutral half of `tools/bridge/claude` into shared packages and make both hosts thin adapters. (2) Duplicate what the new host needs into `tools/bridge/antigravity`, leaving the shipped Claude Code integration completely untouched. (3) Extract only the pure functions (fact filters, intent matching) and duplicate the I/O.

**Rationale.** Option 1 won despite carrying the only real risk in the project — it edits a shipped, working integration. Duplication would have forked the fact-fetching, the filtering, and the session-context body builder, three things that must stay identical across hosts or the two agents see different corpora from the same knowledge base. The risk was contained by a mechanical gate rather than by care: `claude/hook_session_start_test.go` had to keep passing with ZERO diff across the whole branch, which is what proves the emitted text did not shift. Only two test edits were needed anywhere — two URL tests moved packages, and one type literal was renamed.

**The choice.** `tools/bridge/knomitapi` holds the REST client, fact filters and `SessionContext`; `tools/bridge/skills` holds the ten SKILL.md templates. Both hosts consume them.

**Non-scope.** Extraction was limited to what v1 actually consumes. `intents.go`, `transcript_scan.go`, and the post-edit search helpers deliberately stayed in `claude/` because no second consumer exists yet — they move when v2 needs them. This decision does NOT license pre-emptively hoisting anything else that merely looks shareable.

**One constraint that forced structure:** `//go:embed` patterns cannot contain `..`, so a shared template DIRECTORY is unembeddable from two sibling packages. `tools/bridge/skills` therefore had to be a Go package exporting an `embed.FS`, not a bare folder.
