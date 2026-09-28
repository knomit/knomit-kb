---
type: observation
domain: [fleet, triggers, decisions, expires]
confidence: 0.95
sources: 1
entities: [sweepDue, advance, change.episode, TestDue_MixedLearnDue, internal/repos/triggers.go]
motifs: [state-not-event, two-episodes-one-run]
refs: ['src://7b4887ce51d9/internal/repos/triggers.go@5e3036526100c6f98b5a2dbefa8944eaa5d57bf6:7dec69912ce10d6f6f4b34bebb78c82004384850', 'src://7b4887ce51d9/internal/repos/triggers_due_test.go@5e3036526100c6f98b5a2dbefa8944eaa5d57bf6:225eca9a36c5e70e61ea347864b65b207425cd59', 'kb://3ec012f5b4d2/kb/architecture/repos/triggers/due-sweep/a23473de.md', 'https://github.com/knomit/knomit/pull/337']
---
# A mixed `[learn, due]` trigger fires BOTH episodes in one run for a fact learned already past its date (two log rows, `change.episode` tells them apart; likewise `[update, due]` when the date moves to another past instant); rejected suppressing `due` in the run where the same path fired `learn`

OPTIONS (F07 PR 2 RCA D3 / Q8b, 2026-09-28). One trigger, one name, one bookmark row for its tree episodes; the due sweep applies to it as to any `due` trigger. (i) A fact LEARNED already past its date fires `learn` (the advance) AND `due` (the same run's sweep): two fires, two log rows, one run; `change.episode === 'due'` distinguishes them in `if`. (ii) Suppress `due` in the run where the same path fired `learn` — a special case that hides a true state (the fact IS overdue).

RATIONALE: the tree episode and the due state are different facts about the world; hiding one to save a log row costs a script the information it would branch on, and the suppression would need its own bookkeeping. Consequences stated with it: `[retract, due]` fires retract when the fact is deleted and any due mark stays (harmless); `[update, due]` fires update and due together when `expires` moves to ANOTHER past instant (the re-arm).

CHOICE (maintainer): recommended (i), not objected — approved as recommended. Built in #337 (TestDue_MixedLearnDue: learn+due in one run row; update+due share a run).

NON-SCOPE: no ordering between the two rows is promised beyond 'same run'. Sibling ruling: decisions/fleet/f07-triggers/due-activation-fires-once.
