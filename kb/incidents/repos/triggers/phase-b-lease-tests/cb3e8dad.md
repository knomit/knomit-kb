---
type: observation
domain: [repos, triggers, testing, concurrency]
confidence: 0.92
sources: 1
entities: [TestDispatch_NoStoreHeldDuringIf, TestScript_NoStoreHeldDuringScript, TestDispatch_PhaseBHoldsNoLease, Machine.exitStages, openStage.Exit, phaseB, RunOnceForTest, EvalDelay]
motifs: [unverified-sabotage-claim, test-cannot-fail]
refs: ['src://7b4887ce51d9/internal/corpus/repos/triggers_test.go@ee850fd6cef7c936fc81af311371e34a6971350a:ec5104fc4fb24465fc897989083842537899f244', 'src://7b4887ce51d9/internal/corpus/repos/triggers_script_test.go@ee850fd6cef7c936fc81af311371e34a6971350a:b5a3c11fe3aa839ca8834d6b2bd20efccc69b806', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@ee850fd6cef7c936fc81af311371e34a6971350a:59276d39a47bacc416d96d534697ba1708559f14', 'src://7b4887ce51d9/internal/corpus/repos/trigger/triggers.go@ee850fd6cef7c936fc81af311371e34a6971350a:8b055e220ddc6a6925559e47d353fbfde05e5365', 'https://github.com/knomit/knomit/pull/632']
---
# TestDispatch_NoStoreHeldDuringIf and TestScript_NoStoreHeldDuringScript cannot see a store lease held across phase B (a swap cancels the dispatcher before the leases drain); TestDispatch_PhaseBHoldsNoLease, a counting acquire wrapper, is the real guard

Found by the PR #632 reviewer (L1) and fixed in the PR; merged as ee850fd6, re-read there. Two tests claimed in their comments that holding Acquire across phase B turns them red ('Sabotage: hold Acquire across phase B'). The reviewer ran it: release() changed to defer release() after phase A, and both tests stayed GREEN. WHY: a SwapStore goes through Machine.exitStages, which cancels every life (the dispatcher's run is the Serve life) and waits for them in reverse order BEFORE openStage.Exit drains the leases; phase B aborts on ctx within one evaluation and releases either way, so the swap's latency cannot tell a held lease from a released one. The comment was wrong from the day it was written.

THE REAL GUARD. TestDispatch_PhaseBHoldsNoLease (internal/corpus/repos/triggers_test.go) parks the real dispatcher, builds a second one with trigger.New over a counting wrapper of ri.Acquire (counter incremented on acquire, decremented in release), records the outstanding count in the EvalDelay hook at each evaluation, runs one RunOnceForTest pass (phase C only buffered, so no table writes) and requires at least one evaluation and every recorded count 0. Re-run by the re-reviewer at 03c56e27: release deferred to the end of the run gives 'Should be zero, but was 1'. NoStoreHeldDuringIf's own real sabotage is dropping phase B's per-evaluation ctx check (the swap then waits behind the slow ifs: 2.36 s). TestScript_NoStoreHeldDuringScript's comment was corrected the same way.

LESSON. A sabotage note in a test comment is a claim and must be re-run against a commit before it is trusted or copied. A latency-based test observes ctx-cancel behaviour, not resource ownership; to observe ownership, count the acquires.
