---
type: observation
domain: []
confidence: 0.85
sources: 1
entities: [titleOf, chatTitleMax, ChatRailRow, rowState]
refs: ['src://7b4887ce51d9/internal/edge/web/api/chat/chat.go@2ae851d47625101c0c9805ed56fe41918f1533d4:21318500d2fdca3c8880d0fc5240f8691ad7be06']
---
# A chat's title is its first message with whitespace collapsed, cut to chatTitleMax = 300 runes (word boundary in the last 15, then an ellipsis); the UI cuts it with CSS and only the rail tooltip shows it whole

chatTitleMax was 60, which truncated the title at the source so the rail tooltip could not show the full first message. It is now 300 runes: a bound on payload (GET /chats rows, the chat_state event, the snapshot all carry the title), not a display length. Consumers: the rail row ellipsizes with CSS (white-space nowrap), the crumb's name span ellipsizes and its title attribute holds the name, and the tooltip title wraps. Anything past 300 runes still ends in an ellipsis. Does NOT mean the title is a model-written summary: it is still the first message, per the design ruling.
