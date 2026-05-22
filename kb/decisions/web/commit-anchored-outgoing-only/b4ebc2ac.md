---
type: principle
domain: [web, graph, temporal]
confidence: 0.85
sources: 0
entities: [factView, _links.incoming, _links.outgoing, _links.live, _links.snapshot, TestRefTemporal_StateAtDefinitionTime]
refs: ['src://knomit/.claude/plans/2026-04-11-rest-api-hateoas-design.md@0938d83']
---
# Commit-anchored views are OUTGOING-ONLY (no _links.incoming)

HEAD-anchored fact views expose both /incoming and /outgoing. Commit-anchored fact views (GET /branches/{b}/commits/{sha}/facts/{path}) expose ONLY /outgoing — NO _links.incoming, no incoming sub-resource. Rationale: 'who pointed at me at commit C?' requires either a per-commit visibility table (doesn't exist in store) or a tree walk (expensive). The temporal invariant (TestRefTemporal_StateAtDefinitionTime) is entirely about FollowRef which is OUTGOING. Time-travel exploration walks outgoing chains only. Every anchor-carrying link in a commit-anchored response preserves the /commits/{sha}/ segment — the client cannot accidentally drift to HEAD by following links. _links.live points back at the HEAD view (HEAD has no live link — it IS live). _links.snapshot is the immutable citation link from HEAD: commit-anchored URI at the commit where this fact was last changed.
