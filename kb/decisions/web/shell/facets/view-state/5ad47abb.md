---
type: observation
domain: [web, ui, shell, state]
confidence: 0.9
sources: 1
entities: [useViewState, viewReducer, ViewState, SELECT_CONTEXT_TARGET, SET_FACET, lastFacet, knomit.facet, knomit.rail, applyFacetRule]
motifs: [derived-state-liability]
refs: ['src://7b4887ce51d9/web/src/facets.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:5a255dc90a4f1f82cbbff1c61aef97950f4a8295', 'src://7b4887ce51d9/web/src/repoSelection.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:b2c51d1dca7782ec7bd736622bf74bae4c045161', 'src://7b4887ce51d9/web/src/App.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:31841ac63f4195f598cd8bb45f911e3f44b3e36c', 'src://7b4887ce51d9/web/src/facets.test.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:8390a2733689794d02adb5aefc69a0e86dba65b1', 'src://7b4887ce51d9/web/src/App.facets.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:c5ba0220f77e1535173a7e1fde7f15b0e1c549b9', 'https://github.com/knomit/knomit/pull/463']
---
# The facet, rail, focus, connect lock, pin, draft and server-page state live in facets.ts's view reducer, outside AppState and NavEntry; only the facet and the rail preference persist

App holds `useViewState()` (web/src/facets.ts): `railOpen`, `facet`, `lastFacet`, `focus`, `locked`, and (added by the later edge-target and server-page PRs) `pinned`, `draft`, `draftShown`, `returnTo`, `server`. None is a way of looking at facts, so none goes in AppState, NavEntry, pushNav or NAV_BACK.

Only two persist: `knomit.rail` ('1'/'0') and `knomit.facet` ('browse'|'activity'|'settings'; anything else reads as browse), each written by an effect on change. `lastFacet`, `focus`, `locked`, `pinned`, the draft fields and `server` are never persisted, so a reload lands on the remembered context and facet.

The facet ON SCREEN is `view.facet` when `facetsFor(target)` contains it and Settings otherwise; the stored choice is left alone so the next target that has it gets it back. A rail pick dispatches `SELECT_CONTEXT_TARGET` with `facetsFor(target)` BEFORE `SET_CONTEXT` (the facet rule): restore `lastFacet` if the target has it, else keep the facet if it has it, else Settings and remember the facet in `lastFacet` (only if empty). `SET_FACET` (a tab click, Reconnect, enterBranch) clears `lastFacet`.

Consequence: a new route that changes the context must decide whether it applies the facet rule. Plain `SET_CONTEXT` (refresh fallbacks, boot) does not, and relies on the shown-facet fallback.

WHAT THIS DOES NOT MEAN: the facet is NOT per target. One facet is shared and restored by the rule.
