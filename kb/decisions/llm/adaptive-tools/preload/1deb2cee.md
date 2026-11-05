---
kind: pragmatic
type: policy
domain: [llm, chat, embeddings, tools]
confidence: 0.85
sources: 1
entities: [ToolRanker, ScoredTool, NewEmbeddingRanker, NewLexicalRanker, ToolWindow.Preload]
motifs: [calibrated-on-wrong-population]
refs: ['kb://3ec012f5b4d2/kb/invariants/embeddings/thresholds/9f110192.md', 'kb://3ec012f5b4d2/kb/gotchas/embeddings/short-strings/1d564e96.md', 'kb://3ec012f5b4d2/kb/principles/anti-patterns/corpus-property-constants/63e6ecca.md', 'src://7b4887ce51d9/internal/edge/mcp/chat_rank.go@3e19d72cc1f5aaf2383deb20b39b4dc50adca4c6:319c6163da03e2cc1051a6bae247076bcf5f9167', 'src://7b4887ce51d9/internal/engine/llm/toolwindow.go@3e19d72cc1f5aaf2383deb20b39b4dc50adca4c6:d2ce8b5172c3fb253fdf6faeef5f9f835993fc8b']
---
# Tool preload ranks by position under a token budget and never by a similarity score: a cosine cutoff swept on fact bodies is not valid for tool descriptions, and the ranker's scores carry order only

Preload takes tools in rank order while their definition tokens fit min(P, room left); it never compares a score with a threshold. Embedder.Thresholds() were swept on fact bodies under one rendering; tool descriptions are another population (invariant 9f110192; short strings are anisotropic, 1d564e96), so borrowing a cutoff would admit or refuse tools for reasons unrelated to relevance. Consequences: (1) ScoredTool.Score is comparable only inside one Rank call; do not log it as a confidence or threshold it. (2) The one relative rule is that tools tied with the LOWEST score in a ranking are not candidates (a message naming nothing makes the lexical ranker score every tool 0, and preloads nothing); that is a comparison inside one ranking, not a cutoff. A one-tool ranking therefore preloads nothing. (3) The embedding ranker must not call Thresholds(); its test's fake panics if it does. Does NOT mean the ranker is unreliable: it means its output is an ordering.
