---
type: observation
domain: [web, ui, chat]
confidence: 0.9
sources: 1
entities: [ChatToolGroup, ChatPanel, chat-activity-stop, chat-stop, CHAT_STOP]
refs: ['src://7b4887ce51d9/web/src/ChatToolGroup.tsx@8f0d71ca3aeb242ed373badbc47f58953b36d312:8650d67fd397806acb03f2eb2521d0abe4e96233', 'src://7b4887ce51d9/web/src/ChatPanel.tsx@8f0d71ca3aeb242ed373badbc47f58953b36d312:71d4c563c85ebbe598f67243895fd73f454b54ba', 'src://7b4887ce51d9/web/src/ChatTurns.test.tsx@8f0d71ca3aeb242ed373badbc47f58953b36d312:307d6a4872d45dbd13d3066481dd8033814f941d', 'kb://3ec012f5b4d2/kb/decisions/web/chat/side-panel/c12285c0.md']
---
# A chat has ONE Stop: the composer's (and Esc). The running activity line (ChatToolGroup) has no Stop pill and no onStop prop, because the pill called the same chat-level stop and only duplicated it

Chat review, 2026-11-09 (branch fix/chat-stop-approval-chips). ChatToolGroup used to draw a red Stop pill (data-testid chat-activity-stop) beside a running tool group's collapsed activity line. It called the same onStop the composer's Stop calls (ChatPanel passed its own onStop through), so it was a second copy of one action. The user asked for it removed; the pill, its stopPill style and ChatToolGroup's onStop prop are gone, and ChatPanel no longer passes onStop to the group.

What stays: the composer's Stop button (data-testid chat-stop) and Esc while a turn runs. While the turn runs, the activity line is still drawn amber (data-live) — only the pill went.

Consequence: do not add a per-group or per-call Stop back as a 'convenience' — it was removed on purpose. What this does NOT mean: stopping is not less available; the composer's Stop is always on screen while a turn runs.

Test: ChatTurns.test.tsx 'while running, the collapsed line is the same summary drawn amber, with no Stop of its own; the composer's Stop stops' asserts the live line has only its toggle button and the only Stop-named button is chat-stop.
