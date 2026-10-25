---
kind: pragmatic
type: policy
domain: [mcp, web, query, context]
confidence: 0.9
sources: 1
entities: [knomit_query, parseContextFilter, applyContextParams, SearchOptions.Context, factCreateRequest]
motifs: [refuse-never-drop]
refs: ['src://7b4887ce51d9/internal/edge/mcp/query.go@0610c20083fcef9e84f42ab0311a14708bcb6193:ca03e74bb60c2ee60d9ea5fdde1fec35f0b8a102', 'src://7b4887ce51d9/internal/edge/web/api/kb/context_params.go@33eb030738eeefc277e8744f5f8c71e39336251f:035c92e2870b910dceff9c775246c669ac3ddb04', 'src://7b4887ce51d9/internal/edge/web/api/kb/handlers_fact_create.go@33eb030738eeefc277e8744f5f8c71e39336251f:0d77636d4ebb6e97e24af960c7ce8d87a19640a4', 'src://7b4887ce51d9/internal/edge/mcp/context_query_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:7474615cb509fa9b93b76e5101de13c29ab91d73', 'src://7b4887ce51d9/internal/edge/web/context_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:a46070ad4edad0b1d020ea9990c760313b501174', 'https://github.com/knomit/knomit/pull/413']
---
# knomit_query `context` and REST `context.<key>=<value>` are an exact AND match on canonical text, combined with every filter, frozen into the cursor and fanned out to every lens mount; a bad key is refused, never a silent empty result; REST POST create refuses `context`

Values compare as the stored canonical text: a number in shortest form, true/false, and a time in UTC with a Z. Every key must match (AND), and the filter combines with path, type, text and the rest. A context-only query is a valid query. With text it applies after the vector window. The SearchOptions are copied whole to each lens mount, and the result set is frozen with the cursor, so page 2 holds only matches.

REST: the four surfaces that take the expiry params (repo facts, repo search, lens facts, lens search). A key outside `[a-z][a-z0-9_]*`, a repeated key, or a value that could never be stored → 400. Rows and the fact view carry `context`.

REST POST create does NOT accept context and refuses it with a 400: its decoder ignores unknown keys, so the field would otherwise be dropped silently while the call returned 201. Set context with PUT or knomit_learn, which run the gates.
