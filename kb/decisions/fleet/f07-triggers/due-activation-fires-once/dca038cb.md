---
type: observation
domain: [fleet, triggers, decisions, expires]
confidence: 0.95
sources: 1
entities: [sweepDue, trigger_due_fires, TestDue_ActivationAlreadyOverdue, TestDue_RemovedTriggerForgetsMarks, TestDue_SurvivesSwapStore, internal/repos/triggers.go]
motifs: [state-not-event, loud-failure-over-silent]
refs: ['src://7b4887ce51d9/internal/repos/triggers.go@5e3036526100c6f98b5a2dbefa8944eaa5d57bf6:7dec69912ce10d6f6f4b34bebb78c82004384850', 'src://7b4887ce51d9/internal/repos/triggers_due_test.go@5e3036526100c6f98b5a2dbefa8944eaa5d57bf6:225eca9a36c5e70e61ea347864b65b207425cd59', 'kb://3ec012f5b4d2/kb/decisions/fleet/f07-triggers/unsupported-state-no-backfill/f27fc8d8.md', 'kb://3ec012f5b4d2/kb/invariants/repos/triggers/due-marks/db770fa9.md', 'https://github.com/knomit/knomit/pull/337']
---
# Facts already overdue when a `due` trigger first becomes active fire ONCE on the first sweep — and the same after a SwapStore, a rename or a remove-and-re-add — because 'the feature was never live - we're writing it now'; rejected 'never' (seeding marks at activation), whose silence would recur on every bookmark reset

OPTIONS (F07 PR 2 RCA D2, 2026-09-28). When `on: due` becomes active (PR 2 lands on a KB that already declares one, or an operator adds it) and matching facts are already past their date: (a) fire once for each on the first sweep — one burst on that tick, narrowed by `if`, nothing hidden; (b) never — seed a mark for every currently-due matching fact without emitting (the due analogue of the learn side's W = H). The reviewer's M9 named the decisive fact: the dispatcher's ONLY signal that a trigger 'just came alive' is the missing bookmark row, which is also missing after a SwapStore (the clone holds no dispatcher tables), after a rename (a new name is a new trigger) and after a remove-and-re-add (the name-absent rule deletes the row and its marks); under (b) every one of those would silently seed the overdue set — including facts that came due during a store swap — and nothing would tell the operator; under (a) they fire once more, visibly, in the log.

RATIONALE: `due` is a LEVEL (the fact is overdue now), not an event that happened while the trigger was unsupported, so nothing is replayed and no-back-fill (the learn-side rule) is not violated; an operator adding `on: due` to a KB with overdue hypotheses is asking to hear about them; (a) is the simplest shape and the one whose failure mode is loud.

CHOICE (maintainer, verbatim): "Again, the feature was never live - we're writing it now." → (a), no seeding code; a fact fires at most once per due instant per trigger. Built in #337; the learn side of a mixed trigger still bookmarks W = H.

NON-SCOPE: this says nothing about `do: script|push|run`, which stay `unsupported` with no bookmark until their PRs.
