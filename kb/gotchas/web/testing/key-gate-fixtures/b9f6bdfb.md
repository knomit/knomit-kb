---
kind: pragmatic
type: policy
domain: [web, testing, keyboard]
confidence: 0.9
sources: 1
entities: [App.server.test.tsx, App.facets.test.tsx, CLEAR_FILTERS, Escape, Backspace]
motifs: [vacuous-test]
refs: ['src://7b4887ce51d9/web/src/App.server.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:727f4bc38fee46c09369e098e167e4144449cbff', 'src://7b4887ce51d9/web/src/App.facets.test.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:c5ba0220f77e1535173a7e1fde7f15b0e1c549b9', 'src://7b4887ce51d9/web/src/App.tsx@18598fbc4b5cfb3d078ab104efcfdf4463364421:31841ac63f4195f598cd8bb45f911e3f44b3e36c']
---
# A test that a key does nothing on a page must first set up the state that key WOULD change, and pair it with a control on the page where the key works

Escape on Browse clears the open fact (CLEAR_FILTERS) and Backspace pops the nav stack. On a page with no fact open and no history both are invisible no-ops whether or not they are gated, so an Escape test without an open fact passes with the gate deleted (found in the PR D review, finding 2).

The rule: a no-op test first puts real state in place (a committed free-text filter, an open fact, a nav-stack entry), presses the key on the gated page, returns, and asserts the state is still there; and a separate control test shows the same key DOES change that state where it is live. `App.server.test.tsx` opens a fact on Browse, presses the key on a server page, returns by the gear and asserts the fact is still open.

WHAT THIS DOES NOT MEAN: "assert the page is still shown" is not enough. The gate protects the hidden Browse state, not the visible page. Verify the test by sabotage: delete the gate (`if (shownFacet !== 'browse') return;` in App's key handler) and see exactly the key tests fail.
