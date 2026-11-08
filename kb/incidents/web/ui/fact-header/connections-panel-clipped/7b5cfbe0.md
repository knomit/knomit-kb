---
type: observation
domain: [web, ui, fact-header, layout, incidents]
confidence: 0.95
sources: 1
entities: [ConnectionsPanel, ConnectionsCell, FactEdgesGroup, FactView, edgesGroup, web/src/ConnectionsPanel.tsx, web/src/ConnectionsPanel.test.tsx, web/src/FactChrome.tsx]
motifs: [anchor-outlives-its-layout, test-pins-the-bug]
refs: ['src://7b4887ce51d9/web/src/ConnectionsPanel.tsx@70826ce63e0332b8e01918eb049772babeeb25db:8002c88a47333bb20d90c7366b55323c606250f7#L83-L95', 'src://7b4887ce51d9/web/src/ConnectionsPanel.test.tsx@70826ce63e0332b8e01918eb049772babeeb25db:b61cc7c090c72f97304b1d69bdd92a792c1fc504#L85-L92', 'src://7b4887ce51d9/web/src/MotifPanel.test.tsx@70826ce63e0332b8e01918eb049772babeeb25db:03dfbec7da8110c05eb99c854818fda062f7c5e9#L45-L52', 'src://7b4887ce51d9/web/src/FactChrome.tsx@70826ce63e0332b8e01918eb049772babeeb25db:64ef9fa62e0faa19d791f03de9ee9e5ce0c3e0c0#L64-L160', 'src://7b4887ce51d9/web/src/FactView.tsx@70826ce63e0332b8e01918eb049772babeeb25db:5c7171ed9d5d17f9e5d41bf914f4a9d0fe3c4ba5#L632-L645', 'kb://3ec012f5b4d2/kb/decisions/ui/motif/edges-row-header/28608045.md', 'https://github.com/knomit/knomit/pull/323', 'https://github.com/knomit/knomit/pull/177']
---
# The 'cites'/'cited by' panel was clipped at the fact pane's left edge because its right:0 anchor outlived the move of its cells from the header's right end to the start of the edges row

**Symptom (user report with screenshot, 2026-09-27):** in the fact view, clicking 'cites 15' opened a panel whose left ~40px was hidden, apparently under the list rail. The first characters of every row and the '3 retracted' count were cut off. The user reported it for every dropdown in the row.

**Cause:** ConnectionsPanel was `position:absolute; right:0; width:360` on the edgesGroup span. The anchor came from dbd9ab1c (2026-08-03), when the connection cells sat pinned right beside the version chip and a right-anchored panel opened leftward INTO the pane. PRs #177/#180 moved the cells to the START of the reinstated edges row and kept the anchor, so the panel hung leftward past the pane's left edge. The fact view's overflow:hidden ancestors clipped everything outside the pane. The cut fell exactly on the pane edge and the popover had no left border, which is how clipping (not stacking) was identified. MotifPanel was already `left:0` and was never left-clipped. 'same motif 0' is inert, so the user's 'all dropdowns' was really one panel shared by two cells.

**Why no test caught it:** a test named 'hangs from the menu, right-aligned' ASSERTED `right: '0px'`. It pinned the bug instead of the intent, and it stayed green through the layout move because jsdom has no layout.

**Fix (PR #323):** `right:0` → `left:0`. That test now asserts left '0px' with right unset, and MotifPanel has the twin. Verified in the browser on the same fact at 1504×798: panel x=554–914, rail edge 526.

**Lesson for a consumer:** when a trigger moves within a layout, re-check the anchor of whatever hangs from it. A style assertion in a jsdom test pins a value, not the reason for it.

**Where the fix lives at 70826ce6.** The refs point at the FIXED code, not the bug. ConnectionsPanel is still `position:absolute; top:100%; left:0` (ConnectionsPanel.tsx). The 'hangs from the menu, left-aligned' tests in ConnectionsPanel.test.tsx and MotifPanel.test.tsx still assert left '0px' with right unset. The edgesGroup span the panel hangs from is now rendered by FactEdgesGroup in FactChrome.tsx, not RightPanel.tsx, because the fact view moved into FactView.tsx. The overflow:hidden ancestors that clipped it are FactView.tsx's `fact-view` root and its `[data-panel-bounds]` wrapper. The same FactView is mounted in the chat's side panel, so a regressed anchor would clip there too.
