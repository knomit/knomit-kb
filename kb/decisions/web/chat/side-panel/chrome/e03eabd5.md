---
type: observation
domain: [web, ui, chat]
confidence: 0.85
sources: 1
entities: [ChatSidePanel, ChatResultPanel, JsonView, FacetSplitter, .k-chat-result]
motifs: [duplicate-boundary-lines]
refs: ['src://7b4887ce51d9/web/src/ChatSidePanel.tsx@b2dc2edaa499ff10c68589b1c2c3963231e437cb:1b3c81e66211de2857341ad97a60e0b6a8c59b02', 'src://7b4887ce51d9/web/src/ChatResultPanel.tsx@b2dc2edaa499ff10c68589b1c2c3963231e437cb:8f7aae681b5a932c7cc8a2e891af2bc3b3a6d77c', 'src://7b4887ce51d9/web/src/App.css@b2dc2edaa499ff10c68589b1c2c3963231e437cb:7955e96a188cd1e2851aa1917722dbf66b20c9e7', 'src://7b4887ce51d9/web/src/JsonView.tsx@b2dc2edaa499ff10c68589b1c2c3963231e437cb:258d61638384ba0f1eee020445a3c5d42eddeb61', 'src://7b4887ce51d9/web/src/ChatSidePanel.test.tsx@b2dc2edaa499ff10c68589b1c2c3963231e437cb:3e891b7ad10a2f85eb4d679163102d4eadde5147']
---
# Chat side panel chrome for a tool result is minimal (user, 2026-11-09: 'clean it up'): no rule under the header, under the tab row or above the footer; one left edge line (the splitter's); 12px sides; the result box's own frame and the active tab's underline are the only lines kept

Values as merged (PR #819): the result header is 36px tall, padding 0 6px 0 12px, borderBottom none (a commit header keeps its 40px and 1px #1e1e1e rule). ChatResultPanel padding 0 12px (was 18px 20px); the tab row has no border-bottom (was 1px #1e1e1e) and marginBottom 6 (was 14); the active tab keeps inset 0 -2px 0 #6ea8fe. The footer has no border-top (was 1px #1e1e1e) and padding 4px 12px (was 8px 14px). .k-chat-result lost its border-left: the FacetSplitter beside it already draws border-left #222, and the two lines sat 2px apart. Inside JsonView: content padding 6px 0 (was 10px 0), line padding 0 8px 0 6px, fold gutter 14px + 2px; the scroller's webkit track takes the well ground (#0c0c0c), because the app-wide #111 track read as a second line next to the frame.

Consequence: adding a divider to this panel re-opens a user decision. Misreading to avoid: this is about the result entry's chrome; the commit header's rule and the fact band are not part of it, though the footer and the left edge are shared by every entry.
