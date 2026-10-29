---
type: observation
domain: [web, ui, keyboard]
confidence: 0.9
sources: 1
entities: [App.tsx, CLEAR_FILTERS, MaterializeView, keydown]
refs: ['src://7b4887ce51d9/web/src/MaterializeView.tsx@46431604ff228f951b053ed5dea0a84564d0b038:ccc3ea7f114f3b8069b27f28dc389b8ae7dce725']
---
# Esc in the materialize Focus mode must be taken in the capture phase with stopImmediatePropagation, or the shell's Escape (CLEAR_FILTERS) also runs and closes the open fact

App.tsx registers a window keydown handler that on Escape (while the Browse facet is shown and the target is live) dispatches CLEAR_FILTERS; clearing the path filter closes the open fact and with it the materialized view. A plain bubble-phase Esc listener in the view ran AFTER it, so leaving Focus with Esc also dismissed the view (found in a real browser, invisible in jsdom). The view's listener is therefore registered on window in the CAPTURE phase while focus is on and calls stopImmediatePropagation, and is removed when focus ends; a vitest registers a shell-like bubble listener first and asserts it never sees the Esc while focus is on. Esc with focus off is untouched. Related dev trap: a long-running vite dev server in a git worktree can serve STALE source after edits (the file watcher missed them); curl the module to confirm before trusting a browser run.

Field rule (final): the capture listener always consumes the Esc while Focus is on, then decides what Esc means. If document.activeElement is an input, textarea, select or contenteditable element it only blurs it and KEEPS Focus (so Esc in Max depth / Max facts leaves the field); anywhere else it exits Focus. As merged, EVERY input counts as a field, the Cross-KB and Unpin checkbox switches included, so Esc on a focused switch blurs it rather than exiting Focus (a second Esc then exits). Blurring a Max depth / Max facts field applies the typed value (Esc does not cancel the edit). Focus is ON by default when Materialize opens, so this is the Esc a reader meets first.
