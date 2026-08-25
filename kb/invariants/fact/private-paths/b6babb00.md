---
kind: pragmatic
type: policy
domain: [fact, store, okf, mcp, web, synthesize, indexing]
confidence: 0.95
sources: 1
entities: [fact.IsPrivatePath, repoHandler.isFactPath, internal/fact/private.go, internal/store/factpath.go, internal/store/search_index.go, internal/store/verify.go, internal/okf/source/facts.go, internal/okf/source/history.go, okfChangeFromFile, okfDeletionFromFile, looksLikeFact, internal/web/handlers_fact_create.go, internal/web/handlers_fact_write.go, internal/web/handlers_topics.go, internal/web/handlers_lenses_topics.go, internal/mcp/learn.go, internal/mcp/update.go, validateOutputPath, internal/synthesize/decision.go, internal/synthesize/discovery.go, store.validatePath, store.WriteRootFile, ListDir, .domains/ontology.yaml, .github, kb/.drafts]
motifs: [parallel-implementations-diverge]
refs: ['src://7b4887ce51d9/internal/fact/private.go@542903478691569def2d397977201ea5f0ab5fbf:96594a5f7497e024e3a2531b5ed9de42bbab4c9c', 'src://7b4887ce51d9/internal/store/factpath.go@542903478691569def2d397977201ea5f0ab5fbf:fdd3415d6dd43bd6e4311ae504e9964939cfdb0a', 'src://7b4887ce51d9/internal/okf/source/facts.go@542903478691569def2d397977201ea5f0ab5fbf:bc46189391f678cad47cba0e96f619476ad91f00', 'src://7b4887ce51d9/internal/okf/source/history.go@542903478691569def2d397977201ea5f0ab5fbf:b9f5d1851d2470f1a75dc48bbbf68363c036bc89', 'src://7b4887ce51d9/internal/web/handlers_fact_create.go@542903478691569def2d397977201ea5f0ab5fbf:955fe4de9487cdd2582d69dbcead87e7bac7955e', 'src://7b4887ce51d9/internal/web/handlers_fact_write.go@542903478691569def2d397977201ea5f0ab5fbf:0b9813e2fd90a3ea1a5e27d087868d3f87d81f9f', 'src://7b4887ce51d9/internal/web/handlers_topics.go@542903478691569def2d397977201ea5f0ab5fbf:1c5b76ef48cd5a826359344751eae31d201e164c', 'src://7b4887ce51d9/internal/web/handlers_lenses_topics.go@542903478691569def2d397977201ea5f0ab5fbf:4b451f9bf979e8eef934e8e4daed367396e45b57', 'src://7b4887ce51d9/internal/mcp/learn.go@542903478691569def2d397977201ea5f0ab5fbf:139b7d9c61f997de53075a6d4cf659bd2d7bb2ed', 'src://7b4887ce51d9/internal/mcp/update.go@542903478691569def2d397977201ea5f0ab5fbf:9c7047ff4d22b1aae17f3bccc843f727484105bb', 'src://7b4887ce51d9/internal/synthesize/decision.go@542903478691569def2d397977201ea5f0ab5fbf:9b0edebfd92b4306099fba418ee01b50b5a2ab8b', 'src://7b4887ce51d9/internal/store/fact_write.go@542903478691569def2d397977201ea5f0ab5fbf:3f68b610abeb0897e656a03ac5ce68bd0d9bb80a', 'src://7b4887ce51d9/spec/mbekg.md@542903478691569def2d397977201ea5f0ab5fbf:85fedb62c7bad87d2a0d6fb0e75364dfde16c009', 'kb://3ec012f5b4d2/kb/decisions/fact/kb-repo-layout/private-dot-paths/b3425b43.md', 'kb://3ec012f5b4d2/kb/invariants/store/indexing/ontology-root-scope/1b35d56b.md', 'kb://3ec012f5b4d2/kb/decisions/repos/kb-repo-layout/ontology-dot-domains-fallback/318f619a.md', 'kb://3ec012f5b4d2/kb/decisions/repos/kb-repo-layout/readme-clean-break/b534400d.md', 'kb://3ec012f5b4d2/kb/decisions/repos/kb-repo-layout/license-read-only/2b569af1.md', 'https://github.com/knomit/knomit/pull/68']
---
# Private (dot-prefixed) path segments are excluded from fact DISCOVERY at seven walkers and refused at five write sites — never from ACCESS, and deliberately never via store.validatePath

`fact.IsPrivatePath(path)` is true when ANY `/`-separated segment of the path begins with `.` — file OR directory, at any depth, including under the ontology root. A private path is excluded from fact DISCOVERY and REFUSED by knomit's own write paths. `spec/mbekg.md` §3.8 states the same rule as a MUST for readers.

**Why the site list below is the load-bearing part of this fact.** The implementation plan for PR #68 claimed three creation sites and four discovery walkers. Both counts were wrong, and the gaps were caught only in the final whole-branch review — after seven of eight tasks had already shipped. A missed walker leaks a private stash into a live listing; a missed writer commits a fact that no reader will ever see and returns success for it. Enumerate against this list before assuming a new reader or writer is covered.

**SEVEN DISCOVERY GUARDS (skip on walk).** Verified at `54290347` (the PR #68 merge on `dev`):

1. `internal/store/factpath.go`, inside `repoHandler.isFactPath` — ONE place to write, but it covers SIX readers: `search_index.go`'s three admission points (`Sync`'s full-rebuild branch, `Sync`'s incremental added/modified branch, `rebuildFacts`) and BOTH `verify.go` call sites (`checkFactsCoherence`, `checkFactFormat`). Do not mistake "one call site" for "one reader".
2. `internal/okf/source/facts.go` — the OKF fact walk. The check MUST sit BEFORE the `ParseFact`/`looksLikeFact` step, not after: a hand-placed private draft that fails to parse would otherwise be reported as lost knowledge on EVERY export — a permanent false alarm that trains readers to ignore a real one.
3. `internal/okf/source/history.go`, `okfDeletionFromFile` — removing a private file is not the retirement of a published fact.
4. `internal/okf/source/history.go`, `okfChangeFromFile` — feeds authored time and revision history, and must agree with the fact walk about which paths those apply to.
5. `internal/web/handlers_topics.go` — the fact listing under a topic.
6. `internal/web/handlers_topics.go` — the topic tree listing.
7. `internal/web/handlers_lenses_topics.go` — the lens union tree.

**FIVE WRITE REFUSALS (reject, never accept-and-hide).**

1. `internal/web/handlers_fact_create.go` — `POST`, path built from topic/category.
2. `internal/web/handlers_fact_write.go` — `PUT .../facts/{path...}`. The path is FULLY caller-supplied, and this endpoint is also how a brand-new fact is created (`PriorRefs` returns nil for one). This is the site the plan missed; without the guard the branch had silently turned it into "committed, permanently invisible, returns 200".
3. `internal/mcp/learn.go` — `knomit_learn`, refusing to ALLOCATE a private path.
4. `internal/mcp/update.go` — `knomit_update`, refusing to write a REVISION to a private path that already exists. Same rule, other half; not a creation site, which is why a plan that counts only "creation sites" misses it. The file stays readable and retractable.
5. `internal/synthesize/decision.go`, inside `validateOutputPath` — one guard covering FOUR callers: `decision.go` merge, `decision.go` distill, `decision.go` propose, and `discovery.go`'s emergent-fact write.

Refusal is chosen over silent skip on purpose: returning 201/success for a fact that never appears in search is silent knowledge loss, and the hardest class of bug to trace back to its cause.

**TRAP 1 — the check must NOT go in `store.validatePath`.** It looks like the ideal chokepoint, because every write reaches it. It also guards the writes for `README.md` (via `store.WriteRootFile`) and `.domains/ontology.yaml`. Putting the private rule there breaks the boot-time ontology refresh. The rule belongs at the FACT-creation boundaries listed above, not at the generic path validator underneath them.

**TRAP 2 — private means excluded from DISCOVERY, never from ACCESS.** knomit reads `.domains/ontology.yaml` BY NAME and that is permitted; so is reading, updating-by-hand, or deleting an existing private file. The rule governs WALKING, not OPENING. An implementer who compresses this to "never touch dot-paths" builds a reader that cannot load its own ontology. A human may place files under `kb/.drafts/` by hand and knomit respects them as a private stash; the refusal binds only knomit's own write path.

**TRAP 3 — the ListDir-backed browsers are discovery surfaces too.** Guards 5, 6 and 7 walk the RAW GIT TREE via `ListDir`, not the fact index, so they do NOT inherit `isFactPath`'s guard and must filter independently. Each must test the CONSTRUCTED FULL PATH, not the bare entry name: testing `e.Name` alone lets a private ANCESTOR through, so browsing `.../topics/.drafts/facts` directly would list the stash with live self links.

**Relation to the ontology-root invariant.** This is the SAME location test one level finer, and EXTENDS `kb/invariants/store/indexing/ontology-root-scope/1b35d56b.md` rather than competing with it: membership is decided by where a file sits, never by whether its bytes happen to parse.
