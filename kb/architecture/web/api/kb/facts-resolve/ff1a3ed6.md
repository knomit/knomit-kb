---
type: observation
domain: [web, chat, api]
confidence: 0.85
sources: 1
entities: [handleFactsResolve, FactsResolvePath, IDsIn, isMutatingRequest, kb.Mount, auth.ReachesRepo, POST /facts/resolve]
refs: ['src://7b4887ce51d9/internal/edge/web/api/kb/handlers_facts_resolve.go@1b67176bbb84fb46f44660d36c52045a80535349:f3875a753a2c434ce8e79739f0bf7cb09bf1be36', 'src://7b4887ce51d9/internal/edge/web/readonly.go@1b67176bbb84fb46f44660d36c52045a80535349:1d04d90b0c6755b9a721d4ddcda462395c546dc5', 'src://7b4887ce51d9/internal/engine/store/index/search_query.go@1b67176bbb84fb46f44660d36c52045a80535349:d875436ddf287fcbf30af3e48a76ecdec1be24f3', 'kb://3ec012f5b4d2/kb/invariants/web/chat-write-gate/839ea93c.md']
---
# POST /api/v1/facts/resolve turns cited fact handles (8-hex short id, path with or without kb/ and .md, kb://<repo id or name>/<path>) into facts across every repo the caller REACHES; unknown, unreachable and dot-path handles are omitted, never errors

The chat's fact links resolve a turn's cited handles in ONE call: POST /api/v1/facts/resolve {handles: string[]} (max 100) -> [{handle, id, title, type, repo, branch, path, excerpt, confidence}] in the order asked, once per distinct handle. It lives in api/kb (handlers_facts_resolve.go), mounted on the API root through kb.Mount's new first argument `top`; the root HAL document links it as `facts_resolve`.

Lookup: a handle naming a repo (kb://<id12 or name>/...) is looked for only there; any other handle is looked for in every repo for which auth.ReachesRepo(ctx, name) holds, in repo NAME order, at each repo's ReadBranch, and the first match wins. The fleet repo is never searched. A short id is found with index.SearchOptions.IDsIn (f.path LIKE '%/<id>.md'; only lowercase-hex ids are put into the pattern), a path with PathsIn; both set an explicit Limit because a zero Limit is a literal LIMIT 0. Each hit is then read (readFact) for confidence and the excerpt (body minus a leading '# <title>' line, whitespace collapsed, cut at 300 runes); a hit that does not parse still resolves with its index title and type, excerpt empty and confidence 0.

Gates: isMutatingRequest treats POST to exactly httpx.APIBase+kb.FactsResolvePath as a read, so a Read-only principal (writeGate) and a ReadOnly demo server (readOnlyGate) both serve it. This does NOT exempt anything else under /facts: the match is exact, not a prefix.

Consequence: the client must leave any handle missing from the answer as plain code; the absence carries no reason (unknown vs no permission are indistinguishable by design).
