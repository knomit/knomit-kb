---
type: observation
domain: [web, ui, navigation, temporal]
confidence: 0.85
sources: 1
entities: [returnToNow, APPLY_NAV, SET_AS_OF, NAV_BACK, navStack, web/src/state.ts, knomit#507]
motifs: [navigation-step-not-recorded]
refs: ['kb://3ec012f5b4d2/kb/architecture/ui/navigation/navstack-is-also-the-trail/1dd158ac.md', 'kb://3ec012f5b4d2/kb/decisions/ui/temporal/history-exit-and-live-hops/e25d3f7f.md', 'src://7b4887ce51d9/web/src/state.ts@0a75075b40c929d44df7e67509cc4da8762f4c77:2ade0042e1bb1de233493a2f87dc03fc0f1b454e#L632-L641', 'https://github.com/knomit/knomit/issues/507', 'https://github.com/knomit/knomit/pull/506']
---
# OPEN (#507): returning to live from a timeline pushes via APPLY_NAV, so Back re-enters the timeline and closing the fact takes an extra press — #506 left it and adds a redundant adjacent same-fact entry

After PR #506, open fact A, enter its timeline (pushes A live), then return to live (h / Escape / the return-live control): returnToNow dispatches APPLY_NAV, which pushes the timeline state A history. navStack is now [.., A live, A history] with A live current. Back from there goes live -> timeline -> A live -> closed, one press more than the pre-#506 live -> timeline -> closed. Re-entering the timeline from a return-to-live is pre-existing behaviour (returnToNow was always an APPLY_NAV push); #506 left it unchanged and the stack now carries an adjacent same-fact entry (selectTrail collapses it for the trail only). Reported by the PR review as fix-later and filed as https://github.com/knomit/knomit/issues/507 (open at 2026-10-13).

CONSEQUENCE: do not write an e2e or unit expectation that Back after return-to-live closes the fact in one press, and do not 'fix' it inside SET_AS_OF (return-to-live is not a SET_AS_OF entry; the SET_AS_OF entry-push invariant covers that). The intended fix is that return-to-live replaces or skips the push. Re-check state.ts and #507 before relying on this; retract or mark FIXED when #507 lands.
