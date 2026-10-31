---
type: observation
domain: [store, graph, edges]
confidence: 0.9
sources: 1
entities: [IncomingAtCommit, blobHoldings, graphDerivedFromNeighbourRows, branch_facts, path_changes, OutgoingAtCommit, readNode, superseded]
motifs: [proxy-not-the-thing]
refs: ['src://7b4887ce51d9/internal/engine/store/index/search_graph_query.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:3f4cfbcf325cc37d9a07c12701d0e59a5eac73c6', 'src://7b4887ce51d9/internal/engine/store/index/ref_edges.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:8c40e7c412d6592fdaac909882e218f18d276ca3', 'src://7b4887ce51d9/internal/engine/store/index/search_query.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:7e6c12094a16441b4b4346e912cb921cfc6289b8', 'src://7b4887ce51d9/internal/edge/mcp/explain.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:a1c8089a81c6797d8f2e3b65b94f667630b677e1', 'src://7b4887ce51d9/internal/engine/store/blob_edges_test.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:de26f644ebbcace3126346c34eba44acc2c275d1', 'src://7b4887ce51d9/internal/edge/mcp/explain_superseded_merge_test.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:c4c38f16422c54b27c4feecae3a3f7656782190e', 'src://7b4887ce51d9/internal/edge/web/api/kb/handlers_fact_sub.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:d2eeca72516a6d62ea7f071557415f6b62f9c8dc', 'src://7b4887ce51d9/internal/engine/store/index/blast_radius.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:d9f445238d5b77165366f13653f1e26d0dc04e50', 'src://7b4887ce51d9/internal/engine/store/index/highlights.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:7336ac6c9370a4f134b4260f79039a2bd10f0dd2', 'src://7b4887ce51d9/internal/corpus/materialize/reader.go@b208d5ed7105be5b42fabb9cc769cc0f37eebf42:dd9368678a0830ae38d9a344a9967fc5ab01a911', 'kb://3ec012f5b4d2/kb/decisions/store/derived-from-time-aware-edges/a5222e48.md', 'kb://3ec012f5b4d2/kb/decisions/store/temporal/retracted-fact-semantics/85c1bb26.md', 'kb://3ec012f5b4d2/kb/decisions/store/graph/ref-edges-flat-table/aeb15f93.md']
---
# Incoming refs are visible on a branch iff the CITING BLOB is held there (now or in its history), not iff the edge's source_commit is on it; a blob's refs stay resolved once at its earliest introduction even across unrelated branches (option 1), so a branch may show another branch's resolution, flagged as changed since cited

USER RULING (2026-10-30, PR #693 round 4, B4). Scenario: the same bytes X (citing R) are written on brA, then on brC, and the branches never merge. The blob's single claim is pinned at cA on brA. The old incoming filter (source_commit in the reading branch's branch_commits) dropped the edge on brC, so R's incoming there was empty (dev listed it).

INCOMING (IncomingAtCommit, index/search_graph_query.go):
- An edge is visible on branch B iff the citing blob is held on B.
- The source is listed as one row per node of the citing blob whose path holds it on B, now (branch_facts JOIN facts on blob) or in B's history (path_changes rows on B's branch_commits). History keeps retracted citers, which render deleted=true per kb/decisions/store/temporal/retracted-fact-semantics, and a rename's old path. A path only another branch has is not listed.
- blobHoldings answers it for all candidate rows set-based, one statement per chunk (see COST).
- Each row carries a commit OF B at which that path holds the blob: the edge's own source_commit when the row is the edge's own node and that commit is on B (unchanged behaviour; since PR #694 read from ref_edges as the citing blob's claim node and its refs_commit, which are the edge's node and source_commit); else B's recorded commit (branch_facts); else the latest commit of B's history. A hop to (path, commit) on B therefore lands.
- Order (since PR #694, ref_edges): the citing version's source commit in commitEarlier's total order (committed_at, same-second ancestors, hash), then the citing blob, then the path (sortIncomingCandidates); chronological, and unchanged by a rebuild. Before PR #694 it was edge id, then node id, which a rebuild renumbered.
- Since PR #694 the candidates come from ref_edges (idx_ref_edges_target; kb/decisions/store/graph/ref-edges-flat-table), not the DERIVED_FROM edges; the visibility rule above is unchanged.

OPTION 1 (accepted): no per-branch edge sets and no schema change. A blob's refs stay resolved ONCE, at its earliest introduction. On brC, X's outgoing shows R as resolved on brA (R_A at its commit), even when that commit is not on brC:
- OutgoingAtCommit filters target_commit only against the global commit_log;
- the REST gate FactLiveAtCommit and ReadFact AtCommit ignore the branch;
- materialize pins come from OutgoingAtCommit and are read AtCommit.
When brC's own R differs, MCP explain flags the node superseded, which is "changed since cited". Since PR #693 round 5 this is a BLOB comparison: the cited blob vs the blob brC holds for that path now (FactQuery.CitedVersionState). The old commit comparison falsely flagged a target that a merge carried unchanged.

OTHER READERS (checked):
- blast radius widens by blob and counts dependents live on the branch through branch_facts paths;
- highlights keys outdegree by blob over the branch's live nodes;
- verify (reference-blobs) is branch-independent.
None filters by source_commit reachability.

OPERATIONAL CONSEQUENCE: never decide a citer's visibility on a branch by where its blob's edges were resolved. Decide it by whether the branch holds the citing blob.

WHAT THIS DOES NOT MEAN: it does not re-resolve refs per branch. A branch can legitimately show a target version that only another branch ever had; that is the accepted cost of option 1, surfaced as changed since cited, not a bug to 'fix' by re-pinning.

TEMPORAL GRAPH (user ruling 2026-10-30, PR #693 round 5, F2): incoming keeps EVERY reference across all time, and the branch/commit view disambiguates. A citer whose newer version drops the ref is still listed, with each old version that cited the target at its own commit, not marked deleted (F is live). This is intended, not a bug; do not 'fix' it by hiding replaced versions. Two rules each keep those rows: the edge's own node (the citing blob's claim node) with its source_commit (the claim's refs_commit) on the branch, and the path's history on the branch. TestBlobEdges_IncomingKeepsEveryVersionThatCited pins it.

COST: blobHoldings binds two variables per citing (path, blob) node, so it runs in chunks of blobHoldingsChunk (4000) under SQLite's 32766-variable limit, with the same result.

CHANGED SINCE CITED IS RELATIVE TO THE READING BRANCH (user ruling, PR #693; verified at 866eb303): the comparison is always against what the READING BRANCH holds now (branch_facts via FactQuery.CitedVersionState), never against the commit being viewed. Example: F cites R; R is unchanged on main but changed on an agent branch. Read on main, F's reference to R shows NOT changed; read on the agent branch, it shows changed. This is the option 1 consequence above seen from the reader's side. Time-travelling to an old commit does not change the comparison.

MISREADING: "changed is relative to the viewed commit." It is not.
