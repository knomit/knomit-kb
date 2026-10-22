---
type: observation
domain: [web, testing, search, lens]
confidence: 0.9
sources: 1
entities: [cosEmbedder, distinctQueries, searchRepoServer, observeMountSearch, index.Overrides.Search, SearchDiag, min_similarity, motif_match]
motifs: [path-reached-assertion, controlled-input-real-pipeline]
refs: ['kb://3ec012f5b4d2/kb/decisions/store/pr-e/e8-test-doubles/0272840a.md', 'kb://3ec012f5b4d2/kb/conventions/testing/web/9407c9da.md', 'kb://3ec012f5b4d2/kb/gotchas/lens/search/per-mount-embedding/9c1592d5.md', 'kb://3ec012f5b4d2/kb/decisions/store/web-provider-removal/c501f55f.md', 'src://7b4887ce51d9/internal/edge/web/search_realstore_test.go@eeab486fd8fe87449b76b3b53fa57b53a9470743:1b4de0cc52390baceaafe48c593f48d1215ebb0e', 'src://7b4887ce51d9/internal/edge/web/handlers_search_hal.go@eeab486fd8fe87449b76b3b53fa57b53a9470743:be4104cadc5b4c7f267b9488ef2f1779f92b82bb', 'https://github.com/knomit/knomit/pull/605']
---
# Web search tests run the real vector step with cosEmbedder: a document's cosine with every query is the number in its cos=<c> marker (title or body), so the served score is exactly c*100 and nothing stubs a store read

internal/edge/web/search_realstore_test.go defines cosEmbedder, the embedder of the web search tests (PR 605). A query embeds to the unit vector e0; a document embeds to (c, sqrt(1-c^2), 0, ...) where c is the `cos=<c>` marker found in its title OR body (the first match in title+body wins), so its cosine with any query is exactly c and the handler serves score c*100. A document with no marker embeds to e1 (cosine 0). A malformed marker panics. The store's own KNN, cutoff, SearchDiag and sort all run; no store read is stubbed.

Uses: a test that needs 'the cutoff emptied the result' writes a fact at cos=0.54 and asks min_similarity=0.7, and the notice comes from the store's own diagnosis. The embedder counts EmbedQuery calls and can fail or answer an empty vector. With distinctQueries set, every EmbedQuery answers a DIFFERENT vector (a tiny component that grows with the call number), which is what makes 'every mount got the SAME hoisted vector' falsifiable: with a constant fake vector, per-mount embedding produces vectors equal to the hoisted one and the assertion passes with the hoist deleted (seen in sabotage). The query vector is then not unit length, so cosines move by about 1e-6.

Index.Overrides.Search is used only as an OBSERVER (observeMountSearch records the SearchOptions a mount was searched with and calls next unchanged), consistent with E8 (0272840a).

Lens motif tests: the lens widens the query through the UNION vocabulary, so spellings that share a mechanical cluster key (silent-fallback / silent-fallbacks) are one shape across mounts even at motif_match=exact. To prove a tier reaches every mount, separate exact from token-2 with spellings that share two stemmed tokens but are not one cluster (silent-fallback-path vs silent-fallback-retry).

Does NOT mean: cosEmbedder is a model. Ranking between two facts is only as meaningful as the c values the test writes.
