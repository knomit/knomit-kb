---
type: observation
domain: [web, ui, fact-header, layout, incidents]
confidence: 0.95
sources: 1
entities: [ConnectionsPanel, ConnectionsCell, RightPanel, edgesGroup, web/src/ConnectionsPanel.tsx, web/src/ConnectionsPanel.test.tsx]
motifs: [anchor-outlives-its-layout, test-pins-the-bug]
refs: ['src://7b4887ce51d9/web/src/ConnectionsPanel.tsx@2ed1de3515f70cd5bc58f480bec4fd226bc2bffe:d072a46af18277e3331fc9798271a8566a804dfc#L76-L90', 'src://7b4887ce51d9/web/src/ConnectionsPanel.test.tsx@2ed1de3515f70cd5bc58f480bec4fd226bc2bffe:225dc1b486c2a10bb6dfb2c3a7d1c93c21a26d86#L72-L84', 'src://7b4887ce51d9/web/src/MotifPanel.test.tsx@2ed1de3515f70cd5bc58f480bec4fd226bc2bffe:03dfbec7da8110c05eb99c854818fda062f7c5e9#L42-L52', 'kb://3ec012f5b4d2/kb/decisions/ui/motif/edges-row-header/28608045.md', 'https://github.com/knomit/knomit/pull/323', 'https://github.com/knomit/knomit/pull/177']
---
# The 'cites'/'cited by' panel was clipped at the fact pane's left edge because its right:0 anchor outlived the move of its cells from the header's right end to the start of the edges row

**Symptom (user report with screenshot, 2026-09-27):** in the fact view, clicking 'cites 15' opened a panel whose left ~40px was hidden, apparently under the list rail. The first characters of every row and the '3 retracted' count were cut off. The user reported it for every dropdown in the row.

**Cause:** ConnectionsPanel was `position:absolute; right:0; width:360` on the edgesGroup span. The anchor came from dbd9ab1c (2026-08-03), when the connection cells sat pinned right beside the version chip and a right-anchored panel opened leftward INTO the pane. PRs #177/#180 moved the cells to the START of the reinstated edges row and kept the anchor, so the panel hung leftward past the pane's left edge. The fact view's overflow:hidden ancestors clipped everything outside the pane. The cut fell exactly on the pane edge and the popover had no left border, which is how clipping (not stacking) was identified. MotifPanel was already `left:0` and was never left-clipped. 'same motif 0' is inert, so the user's 'all dropdowns' was really one panel shared by two cells.

**Why no test caught it:** a test named 'hangs from the menu, right-aligned' ASSERTED `right: '0px'`. It pinned the bug instead of the intent, and it stayed green through the layout move because jsdom has no layout.

**Fix (PR #323):** `right:0` → `left:0`. That test now asserts left '0px' with right unset, and MotifPanel has the twin. Verified in the browser on the same fact at 1504×798: panel x=554–914, rail edge 526.

**Lesson for a consumer:** when a trigger moves within a layout, re-check the anchor of whatever hangs from it. A style assertion in a jsdom test pins a value, not the reason for it.
