---
type: observation
domain: [chat, embeddings, testing]
confidence: 0.9
sources: 1
entities: [toolVectors, fakeEmb, rankEmbedder, WarmToolVectors, TestChat_OpenWarmsToolVectorsOffTheRequestPath, TestWarmToolVectors_EmbedsOnceAndRankEmbedsNone]
motifs: [process-global-test-state]
refs: ['kb://3ec012f5b4d2/kb/gotchas/web/chat/adaptive-tools/b8030ee3.md', 'src://7b4887ce51d9/internal/edge/mcp/chat_rank.go@3305c0ce138fce4156c7c5bfd69bde24007bf19d:e1057ad305b024097be7fb995845d407782de39d#L14-L30', 'src://7b4887ce51d9/internal/edge/mcp/chat_rank_test.go@3305c0ce138fce4156c7c5bfd69bde24007bf19d:a20675732ee894276a775622b1cfa30df512741d#L42-L66', 'src://7b4887ce51d9/internal/edge/web/api/chat/adaptive_test.go@3305c0ce138fce4156c7c5bfd69bde24007bf19d:94eacb715050aadcf868214e4a83f8f8591ea1e9#L85-L92']
---
# A fake embedder in a test must have an id unique per instance, not t.Name(): the tool-vector cache is process-global per embedder id, so under -count>1 a rerun finds every vector cached and its fake counts zero embeds

mcp's toolVectors cache (chat_rank.go) lives for the process and is keyed by sha256(embedder ID, tool name, description). Test fakes that return t.Name() as ID() share one cache entry set across every -count repetition of the same test, because t.Name() is identical on each run. The second and later runs then see WarmToolVectors and Rank embed nothing, and any assertion on the fake's EmbedDocument count fails: TestChat_OpenWarmsToolVectorsOffTheRequestPath failed on runs 2 and 3 of -count=3 ("every tool is embedded once" never satisfied), and the mcp ranker tests (EmbedsEachToolOncePerToolset, WarmToolVectors_EmbedsOnceAndRankEmbedsNone, ErrorPropagates) failed the same way.

Fix (branch fix/flaky-chat-turn-end): fakeEmb (chat package) and rankEmbedder (mcp package) take the id t.Name() + "#" + a process-wide atomic counter. Operational rule: any new fake model.Embedder whose calls a test counts needs a per-instance id; a test that wants two fakes to SHARE cached vectors must give them the same id explicitly.

Does NOT mean the product cache is wrong: a real embedder id names a model, and vectors for the same model, name and description are the same across chats; the bug was only the fake claiming one identity across independent runs.
