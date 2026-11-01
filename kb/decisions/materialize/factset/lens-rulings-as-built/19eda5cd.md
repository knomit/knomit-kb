---
type: observation
domain: [materialize, lens]
confidence: 0.85
sources: 1
entities: [ReasonLensRepo, Omission.RepoName, planner.lensRepos, FactSetLensTitle, FactSetSearch.Repos, checkRepoNarrowing, FactSetEnv.Anchors, FactSetNode.RepoName, knomit_materialize]
refs: ['kb://3ec012f5b4d2/kb/architecture/materialize/factset/lens-facade/0415bb0c.md', 'kb://3ec012f5b4d2/kb/decisions/lens/qualified-path-repo-identity/10a3bcc0.md', 'src://7b4887ce51d9/internal/corpus/materialize/factset_lens.go@21988d3f0f17a561f61fdab1eb5fe31d7ec2f54e:57037226daffbf54826f327d7efce7e4e39568d7', 'src://7b4887ce51d9/internal/corpus/materialize/factset_plan.go@21988d3f0f17a561f61fdab1eb5fe31d7ec2f54e:5571a414dbd654796195b4df2b1956c1d0e28df4', 'src://7b4887ce51d9/internal/corpus/materialize/factset_search.go@21988d3f0f17a561f61fdab1eb5fe31d7ec2f54e:d6f77abb86b5a7db923d86121c376c5a8d179eb0', 'src://7b4887ce51d9/internal/edge/mcp/materialize_factset.go@21988d3f0f17a561f61fdab1eb5fe31d7ec2f54e:b5003b620fbf2e7fc945e5139e822d80faadf5ba', 'src://7b4887ce51d9/internal/edge/mcp/materialize_factset_lens_contract_test.go@21988d3f0f17a561f61fdab1eb5fe31d7ec2f54e:44641a909cb4ef418b03c64a6691135a4ea0e217']
---
# Lens FactSet rulings Lens 1-7 as built in F3: the not-followed leaf is {path kb://<member>/..., reason lens_repo, repo_name}; cross_kb=true is ErrInvalid; no blob dedupe; env.anchors; 'All in lens <lens>'; since_fork = write member only; search.repo narrowing; repo_name on unit roots

The user's lens rulings (2026-11-01, factset-lens-design section 8) as implemented on feat/lens-factset-facade:

- Lens 1: a ref from a sub into ANOTHER lens member is never followed and nothing of it is read. classify (factset_plan.go) checks planner.lensRepos (every member id12 -> repo name, frozen in the v2 row) BEFORE the cross_kb gate and emits not_included {"path":"kb://<member id12>/<rel>" (or the bare path when the member is the lens write repo, the canonical wire form), "reason":"lens_repo", "repo_name":"<member's repo name at creation>"}. The agent export renders it "<repo_name> `<rel>` (another repo of this lens; not followed)". A ref to a repo OUTSIDE the lens is the plain cross_kb_off omission, as on a repo FactSet. The leaf does not count toward the N.k reservation (subs run CrossKB off).
- Lens 2: cross_kb=true on a lens FactSet is ErrInvalid "cross_kb is not accepted on a lens factset" (MCP tool error; REST 400 in F4), before any read.
- Lens 3: no blob dedupe; the same path is write-first through federate.WriteFirstWinners; the same blob at two paths stays two roots (TestLensFactSet_NoBlobDedupeAcrossForks).
- Lens 4: lens pages carry env.anchors {member id12: anchor}; env.anchor stays the write member's. Keyed by id12, not name (10a3bcc0: ids on the wire).
- Lens 5: the no-filter title is "All in lens <Binding.Name()>" (FactSetLensTitle); filtered titles follow the repo rules.
- Lens 6: since_fork reads ONLY the write member's leg, with the edge's PathsIn; other members' legs return nothing (NoResults makes every leg empty). repo= is FactSetSearch.Repos (JSON "repos", MCP search key "repo"), passed as FusedQuery.Repos, so the FactSet equals the narrowed lens list; an unknown name is the lens list's own UnknownRepoError text as ErrInvalid. On the repo path a one-member lens accepts only its write repo's name; a repository's own FactSet refuses any repo. The search line spells it "repo <names>". A one-member lens stays on the repo path.
- Lens 7: every unit root of a lens page carries "repo_name" (frozen); node.repo (id12) was already on every node.

CONSEQUENCE: these visible strings are new and were not ruled word for word: the reason token lens_repo, the export text "another repo of this lens; not followed", the cross_kb error, the repo-path refusal "repo narrows a lens factset to some of its repos; this binding is one repository", the search-line label "repo", and the MCP key description for search.repo. Treat them as the implementer's until the user confirms.

WHAT THIS DOES NOT MEAN: MCP does not accept since_fork in factset mode (it never did); the engine behaviour is ready for the REST route (F4).

FOR F4 (REST lens routes; review L5, no engine change): an unknown search.repo / repo= name reaches the edge as ErrInvalid wrapping federate.UnknownRepoError (text: no mount named "X" in lens "Y"). The REST lens FactSet route should map it to 422 "Unknown repo", as the lens list does (planLensRead's StageRepos), not to the generic 400 every other ErrInvalid gets; check errors.As for *federate.UnknownRepoError before the ErrInvalid branch.
