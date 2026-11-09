---
type: observation
domain: []
confidence: 0.85
sources: 1
entities: [titleOf, chatTitleMax, ChatRailRow, rowState, chatTitleShown, TopBar]
refs: ['src://7b4887ce51d9/internal/edge/web/api/chat/chat.go@aa98431a369f919af94c4e2184440814ea9f9d1c:6583fcab90e0cb46bc07edfb200297ce58fa82c1', 'src://7b4887ce51d9/web/src/utils.ts@cbe3eb26e3d954cc035328b3f5b7cc5b6d33f394:bbdfc46566c7aa0597a008a78b749ca51ff3034f', 'kb://3ec012f5b4d2/kb/decisions/web/chat/titles/display-cap/98a825c1.md']
---
# A chat's title is its first message with whitespace collapsed, cut to chatTitleMax = 300 runes (word boundary in the last 15, then an ellipsis) as a payload bound; the UI DISPLAYS it capped at 20 characters + '…' (chatTitleShown) in the rail row and the top bar, and only the rail tooltip and the bar's title attribute show it whole

chatTitleMax was 60, which truncated the title at the source so the rail tooltip could not show the full first message. It is now 300 runes: a bound on payload (GET /chats rows, the chat_state event, the snapshot all carry the title), not a display length. Consumers (since 2026-11-09, superseding 'the UI cuts it with CSS'): the rail row and the crumb's name span both show chatTitleShown(title) — 20 characters (graphemes) cut at a word boundary, then '…' (see kb/decisions/web/chat/titles/display-cap/98a825c1.md) — with their CSS ellipsis kept only as the narrow-width fallback; the crumb's title attribute and the rail tooltip still hold the whole title, and the tooltip title wraps. Anything past 300 runes still ends in an ellipsis. Does NOT mean the title is a model-written summary: it is still the first message, per the design ruling.
