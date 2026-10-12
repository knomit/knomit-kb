---
type: observation
domain: [web, ui, shell, boot]
confidence: 0.9
sources: 1
entities: [BootScreen, boot.phase, browse-opening, state.branch, SET_CONTEXT]
refs: ['src://7b4887ce51d9/web/src/App.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:31841ac63f4195f598cd8bb45f911e3f44b3e36c', 'src://7b4887ce51d9/web/src/BrowseFacet.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:5f198c939e55febe02ab8d3c1b891305a923c9ea', 'src://7b4887ce51d9/web/src/boot.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:4acebfedc25d837a207f67963f31022c8626dfb2', 'src://7b4887ce51d9/web/src/App.facets.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:c5ba0220f77e1535173a7e1fde7f15b0e1c549b9', 'kb://3ec012f5b4d2/kb/decisions/web/boot/two-hops-and-progress/66045611.md']
---
# The BootScreen covers only the first boot; a later context switch keeps the shell and Browse shows Opening <name> with no panels mounted until the branch lands

App's gate is `!state.branch && boot.phase !== 'done' && !noRepos`. The boot reducer is monotonic and `done` is terminal (Retry exists only on the BootScreen, before done), so after the first branch the shell (top bar, rail, footer) stays mounted through `SET_CONTEXT`, which clears the branch. BrowseFacet returns `browse-opening` while `branch === ''` and mounts no panel, because every panel reads under the branch and would 404. Activity and Settings need no branch.

Consequence: a new body that reads under the branch must gate itself on `state.branch`.

WHAT THIS DOES NOT MEAN: a failed status bootstrap after a switch no longer shows the BootScreen's retry; the placeholder stays until the bootstrap succeeds.
