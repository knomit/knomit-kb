---
type: observation
domain: [web, ui, library, materialize, decisions]
confidence: 0.95
sources: 1
entities: [LibraryHeader, ModeFlyout, COLLAPSE_MS, library-factset, library-header-sep, data-count, web/src/LibraryHeader.tsx, web/src/ModeFlyout.tsx]
motifs: [design-lost-in-handoff]
refs: ['src://7b4887ce51d9/web/src/ModeFlyout.tsx@45cbfd326ba61d19c1b554aa341df1e8f1f58699:11614600810e6ed38f214026c0640bf1f03697c1', 'src://7b4887ce51d9/web/src/LibraryHeader.tsx@45cbfd326ba61d19c1b554aa341df1e8f1f58699:b203b49e0e845e577f5d3c602653d0e83f9c64da', 'src://7b4887ce51d9/web/src/LibraryHeader.flyout.test.tsx@45cbfd326ba61d19c1b554aa341df1e8f1f58699:9c2f3f59ab681c817ae2cf8b749595a09a9bb662', 'src://7b4887ce51d9/web/src/Library.factset.test.tsx@45cbfd326ba61d19c1b554aa341df1e8f1f58699:6c0e032d91f5677e48771fdbfbc9afae654fc849', 'src://7b4887ce51d9/web/src/Library.keyboard.test.tsx@45cbfd326ba61d19c1b554aa341df1e8f1f58699:9c6dbe674768617b49cb09aab7a36ba8986d17e5', 'kb://3ec012f5b4d2/kb/decisions/ui/motif/pivot-is-a-mode/3b93b986.md', 'kb://3ec012f5b4d2/kb/architecture/web/factset-view/60d4ace8.md', 'https://github.com/knomit/knomit/pull/734']
---
# The results-list header (user design, book round; restored in PR #734): the modes collapse into a Photoshop-style flyout opening LEFT, Materialize stands ALONE at the right after a vertical bar and takes the header accent on hover, disabled for an empty list, and the header shows NO total

ORIGIN: the user designed this header in the 'book' design round. The decision is recorded in the plan book-materialize-design.md §5, the 'Entry point' bullet, and in the prototype notes, item 1, 'Placement = option (d), as a Photoshop-style flyout'. In those notes the user picked option (d), and the master chose the left direction per the prototype's recommendation, uncontested. The shipped FactSet UI (#710/#726) dropped it, because the plan its UI worker was briefed from never carried it over. The user then rejected the shipped look: a count followed by three side-by-side mode buttons and Materialize. This fact exists so the design cannot be lost a second time. Plans are not refs, so the content is restated here.

THE DESIGN (as merged in #734 at 45cbfd32):
1. MODES = ONE TOOL GROUP (web/src/ModeFlyout.tsx).
   - At rest only the CURRENT mode's icon shows, with a 4px corner triangle at the bottom right.
   - Hover or keyboard focus on it opens the other modes INLINE TO THE LEFT. They are icon-only, the same size and colour, in an opaque strip drawn over the location line (absolute, right: 100%, the header background).
   - One click picks a mode.
   - It collapses COLLAPSE_MS = 250 ms after the pointer leaves. Re-entering cancels, so it doesn't flicker; the strip is a DOM child of the group, so moving between icons never leaves. It also collapses on Esc (focus returns to the current icon) and when focus leaves the group.
   - A click on the current icon keeps the group open for a mouse and toggles it for touch or pen. The pointerType and the open state are read at pointerdown, before the tap's focus opens it.
   - Left/Right move between enabled modes in on-screen order (the strip left to right, then the current icon). The group keeps Left/Right from the Library's window-level list keys ONLY while open, always keeps Enter and Space (the Library's Enter preventDefaults, which would cancel the button), and never keeps Up/Down, so the list's arrows still work when focus sits on a mode icon after a pick. A keyboard pick (click detail 0) returns focus to the current icon; a mouse pick does not.
   - Safari blurs a pressed button with a null relatedTarget before its click, so the group ignores blur while a pointerdown inside it is in progress (cleared on pointerup, pointercancel or click). Otherwise the strip would unmount and the click would be lost.
   - Only the current icon is tabbable. It carries aria-haspopup and aria-expanded. Every icon keeps its aria-label ('Sort by X', or the exits 'Clear search' / 'Leave this motif') and its tooltip.
2. MATERIALIZE ALONE AT THE RIGHT: the newspaper glyph is the header's ONLY action and its last child. A 1px vertical bar (#3a3a3a; #222 was invisible on the #0f0f0f row in a browser) separates it from the mode group. On hover its colour becomes the header accent #7c9, the lit mode's colour; no new colour was introduced. It is disabled for an EMPTY list (count 0) and in history. Its aria-label is always 'Materialize results'. Its tooltip is 'Materialize results', except in history, where it is 'Not available in history' (a PLACEHOLDER). It is absent in the path tree, and so is the bar.
3. NO TOTAL in this header (user, 2026-11-02: 'not in the design'). The count prop still exists: it decides 'empty', and it rides on the header's data-count attribute for the count tests. The prototype notes did list a count in the header; the user's 2026-11-02 ruling supersedes that.

KEPT FROM BEFORE: the mode set, icons, labels, disabled rules and derived-mode exits are unchanged (decision kb/decisions/ui/motif/pivot-is-a-mode/3b93b986.md). While search relevance or a motif pivot is current, a click on the current icon still EXITS, because every other mode is disabled then and the flyout has nothing to pick. In every other state the current icon is only the trigger and never re-dispatches its own sort.

CONSEQUENCE: a plan or brief that touches the results-list header must carry this design verbatim. Do not put the modes back side by side, do not add a count beside them, and do not add a second action next to Materialize without a user ruling.

WHAT THIS DOES NOT MEAN: it does not remove the count from Library. Library still computes it (Recent = server total, Relevance/Path = rows) and passes it down; only its display is gone. It does not change the single-fact view's Materialize button in the fact header. It also does not cover the FactSet view's own bar, the ribbon, the sticky path or the link stubs; those have their own rulings.
