---
kind: pragmatic
type: policy
domain: [web, ui, history]
confidence: 0.8
sources: 0
entities: [navStack, ENTER_HISTORY, EXIT_HISTORY, NAV_BACK, HistoryTimeline, /api/v1/commit]
refs: ['src://knomit/.claude/plans/2026-03-15-history-mode-design.md@0938d83']
---
# navStack is 10-deep; pushed on NAVIGATE/SELECT_FACT/ENTER_HISTORY/SEARCH; oldest dropped when full

History mode = vertical git commit timeline scoped to the current ontology path, toggled with `h`. Tags (learn/*, update/*, retract/*, synthesize/*) shown as colored badges (green, blue, red, orange). navStack is capped at 10 entries. PUSH POINTS: NAVIGATE (directory change), SELECT_FACT, ENTER_HISTORY, SEARCH. Oldest entry dropped when full. Backspace/Delete dispatches NAV_BACK to pop the stack. Escape in history mode dispatches EXIT_HISTORY (takes priority over other Escape behaviors). New backend endpoint: GET /api/v1/commit?hash=<commit_hash> returns commit detail (date, message, tags, files[] with action: added|modified|deleted). GET /api/v1/fact?path=...&commit=<hash> accepts optional commit param — reads at that commit instead of HEAD. History endpoint supports cursor pagination via &after=<commit_hash> returning &next=<commit_hash>.
