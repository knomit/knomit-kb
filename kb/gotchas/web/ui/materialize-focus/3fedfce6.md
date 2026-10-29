---
type: observation
domain: [web, ui, keyboard]
confidence: 0.9
sources: 1
entities: [App.tsx, CLEAR_FILTERS, MaterializeView, keydown]
refs: ['src://7b4887ce51d9/web/src/MaterializeView.tsx@36b925f96218a548a67fdf076c911870778ff6fa:ea619008f79d3ce2ccc0bd77bb7296f72a5ac37c']
---
# Esc in the materialize Focus mode must be taken in the capture phase with stopImmediatePropagation, or the shell's Escape (CLEAR_FILTERS) also runs and closes the open fact

App.tsx registers a window keydown handler that on Escape (while the Browse facet is shown and the target is live) dispatches CLEAR_FILTERS; clearing the path filter closes the open fact and with it the materialized view. A plain bubble-phase Esc listener in the view ran AFTER it, so leaving Focus with Esc also dismissed the view (found in a real browser, invisible in jsdom). The view's listener is therefore registered on window in the CAPTURE phase while focus is on and calls stopImmediatePropagation, and is removed when focus ends; a vitest registers a shell-like bubble listener first and asserts it never sees the Esc while focus is on. Esc with focus off is untouched. Related dev trap: a long-running vite dev server in a git worktree can serve STALE source after edits (the file watcher missed them); curl the module to confirm before trusting a browser run.
