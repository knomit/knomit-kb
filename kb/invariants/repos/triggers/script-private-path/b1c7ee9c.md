---
kind: pragmatic
type: policy
domain: [repos, mcp, triggers, script, private-paths]
confidence: 0.9
sources: 1
entities: [factPath, IsPrivatePath, NormalizePath, IsWritablePrivatePath, TestScript_PrivatePathRefused, TestScriptTools_SameCodePathAsMCP, internal/repos/trigger_script.go]
motifs: [caller-side-denial]
refs: ['src://7b4887ce51d9/internal/repos/trigger_script.go@49bc0a6a5fc8493d538be563cf9baa204071f160:e65c3fdb14a763353b138fdccdc20cbb22ed2b15', 'src://7b4887ce51d9/internal/mcp/script_tools_test.go@49bc0a6a5fc8493d538be563cf9baa204071f160:17d7462edaa7f4e21e527762e2bd39c157ab1670', 'src://7b4887ce51d9/internal/repos/triggers_script_test.go@49bc0a6a5fc8493d538be563cf9baa204071f160:08810bc5fb48efa863f3b71f6c2e4cee75869733', 'https://github.com/knomit/knomit/pull/339']
---
# A trigger script may NOT write under `.knomit/`, and the refusal is the HOST's, not the handler's: the MCP learn/update/retract tools deliberately ACCEPT `.knomit/<area>/…` job state for sessions, so "same code path as the MCP tools" alone would let a script write there — the host refuses a `path` key on any learn fact and any private path on update/retract (checked after fact.NormalizePath, the handler's own order) BEFORE the tool runs

Ruling R7: `.knomit/` is never trigger-visible and a script may not write under it. In internal/repos/trigger_script.go: `learn` returns an error for any fact object carrying `path` (a script names topic and category only); `update`/`retract` go through `factPath`, which normalises the argument with `fact.NormalizePath(ontologyRoot, file)` and refuses `fact.IsPrivatePath` — so `.knomit/x` and `.knomit/x.md` are refused alike. The error is thrown to the script as "a script may not write under .knomit/", the fire completes (`ran` if the script catches it), nothing is committed and no tool is called. TestScript_PrivatePathRefused proves the refusal; TestScriptTools_SameCodePathAsMCP proves the same calls through the real tool set SUCCEED, which is what shows the check lives in the host.

Misreading to avoid: this is the ONE place a script is denied what a session may do; everything else (validations, dedup, the refs gate, CAS, the write branch) is the unchanged handler's.

F08 PR A (M2) adds a fourth door with the SAME refusal: `knomit.learn(facts, {retract: [paths]})` (the F04 move from a script; `knomit.learn([], {retract})` is a batch retraction). The host runs every retract path through privateRefused (factPath's rule: NormalizePath, then IsPrivatePath) and throws before the tool runs; a non-list `retract` or a non-string entry throws too. The learn tool itself would accept `.knomit/<area>/` retract paths for a session, so again the refusal is the host's. Pinned by TestScript_LearnRetractPrivateRefused; forwarding by TestScript_LearnRetractForwarded and, through the real learn handler (one commit, +move@ author, trailers), TestScript_MoveThroughRealTools in internal/web. Merged in #348 (merge commit 53d102e9): src://7b4887ce51d9/internal/repos/trigger_script.go@53d102e998b25671eb2922a9ba25b064652ec6fd:94380721535d2fc448497ee570290528720cdc95, src://7b4887ce51d9/internal/web/script_e2e_test.go@53d102e998b25671eb2922a9ba25b064652ec6fd:20468a2ae0cdef26d0056f5e0b9384af489bb404.
