---
type: observation
domain: [mcp, hypothesize, synthesize, sessions]
confidence: 0.9
sources: 1
entities: [knomit_hypothesize, ContinueSessionForItem, hypothesizeStrategy, acknowledgedResponse, knomit_review, session_id]
motifs: [failure-presents-as-success]
refs: ['src://7b4887ce51d9/internal/edge/mcp/hypothesize.go@da40f11df65370c9bada04ab46c5a12386347a7c:60c937b9645c5dc7f7b0fe1111a0fbe3c8d96234', 'src://7b4887ce51d9/internal/corpus/synthesize/hypothesize_strategy.go@da40f11df65370c9bada04ab46c5a12386347a7c:0e585c2782867dedaeaa06fb904d1ffbd6f02229', 'src://7b4887ce51d9/internal/corpus/synthesize/pipeline.go@da40f11df65370c9bada04ab46c5a12386347a7c:22dc19ad5c55751af814cd1464dffe506e96bbb9', 'src://7b4887ce51d9/internal/edge/mcp/review.go@da40f11df65370c9bada04ab46c5a12386347a7c:3ec150c62f2ad610462f5c3577501b5575ed26db', 'https://github.com/knomit/knomit/pull/657']
---
# knomit_hypothesize called with session_id but no response (and without `current: true`) ANSWERS the outstanding item as acknowledged; it does not re-read it

The handler calls ContinueSessionForItem with response ""; hypothesizeStrategy.Decode substitutes "acknowledged". On a discover item a blank response means "no proposals". The item is claimed and the next one is returned.

To see the current item again without answering it, pass `current: true` with session_id alone (current=true with any response or item_id is refused).

WHAT THIS DOES NOT MEAN: knomit_review does not behave this way. There session_id alone returns the current item, and a response requires item_id. Also, effort/domain/entities on hypothesize are start-only; they are ignored on a continue.
