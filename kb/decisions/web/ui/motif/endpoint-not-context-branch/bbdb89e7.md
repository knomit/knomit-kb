---
type: observation
domain: [web, ui, lens, motif]
confidence: 0.95
sources: 1
entities: [web/src/motifEndpoint.ts, MotifEndpoint, motifEndpointOf, canReadMotifs, canReadMotifCluster, readMotifs, readMotifCluster, web/src/MotifsBlock.tsx, web/src/useMotifClusters.ts, web/src/FilterBar.tsx, web/src/Library.tsx, web/src/RightPanel.tsx, factHistoryAnchor, api.lensMotifs, api.lensMotifCluster]
motifs: [one-seam-many-consumers, same-question-drifting-guards]
refs: ['src://7b4887ce51d9/web/src/motifEndpoint.ts@7c9aa98e2d09138bbf1dff085fa1239af6a0519b:22810e5b637146bd4e4bc749d262e612e3f5e725', 'src://7b4887ce51d9/internal/web/handlers_lenses_motifs.go@7c9aa98e2d09138bbf1dff085fa1239af6a0519b:77a54054d5c061ffa5dd220c78399538999a151f', 'kb://3ec012f5b4d2/kb/decisions/ui/motif/vocabulary-fourth-block/b5534397.md', 'kb://3ec012f5b4d2/kb/decisions/lens/read-endpoints-not-stopgap/ddd424f9.md']
---
# The motif UI takes a MotifEndpoint, not an isLens flag — one derivation, and no component below it knows a lens exists

# One endpoint value, not a context branch per component

When the lens motif vocabulary landed, four surfaces had to stop being repo-only: `MotifsBlock`, `useMotifClusters`, the FilterBar `motif:` category, and Library's pivot heading resolution.

## Options considered

1. **`isLensContext(state)` at each call site**, branching to `api.lensMotifs` / `api.motifs` — the shape the surrounding code already used for facts and search. Rejected: four branches is four chances to drift, and three of the four had ALREADY drifted into different gating expressions for the same question (`isLens || typeof api.motifs !== 'function'` in FilterBar, a bare `!repo` in the block, `!repo || !branch` in the hook).
2. **A `lens?: string` prop added beside `repo`/`branch`** — rejected: it makes the illegal state (both set, neither set) representable.
3. **A `MotifEndpoint` discriminated union derived once from state** (chosen).

## The choice

`web/src/motifEndpoint.ts` holds the type, `motifEndpointOf(state)`, the two readers (`readMotifs`, `readMotifCluster`) and the availability predicates. Everything above it is context-blind: the block, the picker and the hook each gained a lens WITHOUT gaining a lens code path.

This works only because the SERVER made the two responses the same shape on purpose — `renderMotifCollection` is shared by both handlers. The client's single call site is downstream of the server's single renderer, not a coincidence alongside it.

## Two consequences that are easy to get wrong

- `canReadMotifs` and `canReadMotifCluster` are SEPARATE predicates. A vendored client (knomit.io `/explore` swaps `api.ts` for a static bundle) can carry one read and not the other; collapsing them lets one surface's absence silently govern the other's. Both also fold in "is the endpoint addressable yet" — App picks a repo from `/repos` on mount, so the first render has none.
- In a lens, a fact's motif counts now resolve against the LENS, not the fact's own mount. The count sits beside a pivot and the pivot lists the lens; a mount-scoped count under a lens-scoped pivot promises one number and delivers another. Edges and history still anchor on the fact's own mount via `factHistoryAnchor` — those are questions about the fact's history, which IS a property of where it lives.

## Non-scope

This does not say every lens/repo branch in the web client should become an endpoint value. It applies where the two REST surfaces return the SAME shape. Facts and search do not (a lens row carries `source`), which is why those keep their `isLens` branches.
