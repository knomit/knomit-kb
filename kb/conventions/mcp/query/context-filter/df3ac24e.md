---
kind: pragmatic
type: policy
domain: [mcp, web, query, context]
confidence: 0.9
sources: 1
entities: [knomit_query, parseContextFilter, applyContextParams, SearchOptions.Context, factCreateRequest]
motifs: [refuse-never-drop]
refs: ['src://7b4887ce51d9/internal/mcp/query.go@4345c879623bc67951560c2c7e018c0c50964214:5b2f86cdbdbb34d6714a596dcd4402a5e310ea42', 'src://7b4887ce51d9/internal/web/context_params.go@4345c879623bc67951560c2c7e018c0c50964214:693908353aa7743c3219e018a76266a0031bcff0', 'src://7b4887ce51d9/internal/web/handlers_fact_create.go@4345c879623bc67951560c2c7e018c0c50964214:88345e3785977760130b0c7d340ed877532ab074', 'src://7b4887ce51d9/internal/mcp/context_query_test.go@4345c879623bc67951560c2c7e018c0c50964214:d84a0ea929cdba1f24b5cb06a99c9391a82a86f2', 'src://7b4887ce51d9/internal/web/context_test.go@4345c879623bc67951560c2c7e018c0c50964214:fd9b1a04c1d693f8c3837de575d86dae5e015b63', 'https://github.com/knomit/knomit/pull/413']
---
# knomit_query `context` and REST `context.<key>=<value>` are an exact AND match on canonical text, combined with every filter, frozen into the cursor and fanned out to every lens mount; a bad key is refused, never a silent empty result; REST POST create refuses `context`

Values compare as the stored canonical text: a number in shortest form, true/false, and a time in UTC with a Z. Every key must match (AND), and the filter combines with path, type, text and the rest. A context-only query is a valid query. With text it applies after the vector window. The SearchOptions are copied whole to each lens mount, and the result set is frozen with the cursor, so page 2 holds only matches.

REST: the four surfaces that take the expiry params (repo facts, repo search, lens facts, lens search). A key outside `[a-z][a-z0-9_]*`, a repeated key, or a value that could never be stored → 400. Rows and the fact view carry `context`.

REST POST create does NOT accept context and refuses it with a 400: its decoder ignores unknown keys, so the field would otherwise be dropped silently while the call returned 201. Set context with PUT or knomit_learn, which run the gates.
