---
type: observation
domain: [web, ui, shell, server]
confidence: 0.9
sources: 1
entities: [railShown, railOpen, TOGGLE_RAIL, SET_RAIL, ViewState.server, knomit.rail]
motifs: [preference-vs-shown-state]
refs: ['src://7b4887ce51d9/web/src/facets.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:5a255dc90a4f1f82cbbff1c61aef97950f4a8295', 'src://7b4887ce51d9/web/src/App.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:31841ac63f4195f598cd8bb45f911e3f44b3e36c', 'src://7b4887ce51d9/web/src/TopBar.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:c36df268fc11b2f54f672d6e7b41f68b84ea6e85', 'src://7b4887ce51d9/web/src/App.rail.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:41083a2c5d5b9ed8d6a386ffa35c37177fd6ea4f', 'kb://3ec012f5b4d2/kb/decisions/web/shell/target-rail/bb551c94.md', 'kb://3ec012f5b4d2/kb/decisions/web/shell/facets/server-pages-behind-gear/911dd19a.md', 'https://github.com/knomit/knomit/pull/468']
---
# railOpen is the reader's preference, not the shown state: over the server pages the rail is hidden and the toggle is disabled, and the preference is untouched

`railShown(view, noRepos)` in web/src/facets.ts is `server === null && (railOpen || noRepos)`. `TOGGLE_RAIL` and `SET_RAIL` return the same state while `server` is set, and `TOGGLE_SERVER`/`SET_SERVER_PAGE` never touch `railOpen`, so opening and closing the server pages never changes `railOpen` or the persisted `knomit.rail`. App mounts the rail from `railShown` and passes it to TopBar, which disables the toggle (same 26x26 box at the same place, title 'Repositories and lenses are hidden over the server pages') while a server page is open. The forced-open no-repos rail is hidden over the pages too and returns on close.

Command+K over the server pages does nothing: it neither opens the rail nor raises a focus request (a request raised then would fire when the rail remounted). The status footer's creates link is likewise a no-op there. A rail pick can no longer be a route back from the server pages: the gear is the only one (Escape does not leave them).

Consequence: new code that shows or hides the rail must read `railShown`, not `view.railOpen`, and any new rail-opening action must be refused over the pages.

WHAT THIS DOES NOT MEAN: this is not 'the rail is closed over the pages' (that would persist closed); it is hidden, with the preference restored on return.
