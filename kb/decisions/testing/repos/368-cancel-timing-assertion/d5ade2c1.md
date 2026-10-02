---
type: observation
domain: [repos, testing, create, cancel]
confidence: 0.95
sources: 1
entities: [TestCancelCreate_DuringIndexLandsWithoutWaitingForTheIndex, TestCreate_CancelledDuringIndexSkipsSyncActivation, sawSync, Manager.CancelCreate, RepoInstance.ActivateSync, indexHealGate, knomit#368]
motifs: [blind-timing-check, baseline-measured-elsewhen]
refs: ['kb://3ec012f5b4d2/kb/meta/methodology/measure-outcome-not-the-race/eae2bb22.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/create-job/cancel-during-index/25f58520.md', 'src://7b4887ce51d9/internal/repos/create_job_cancel_test.go@5f000d185c49cedf3662b293912a33bc7c6a16a2:7aad885ea50887451a0e8c474d628243ba5f9e8f', 'https://github.com/knomit/knomit/issues/368', 'https://github.com/knomit/knomit/actions/runs/36590870983']
---
# TestCancelCreate_DuringIndexLandsWithoutWaitingForTheIndex drops `elapsed < fullCreate` and its baseline create; the regression stays caught by sawSync (e2e) and TestCreate_CancelledDuringIndexSkipsSyncActivation (deterministic) — test-only, no production gate

Context (#368): the test failed on Windows CI (run 36590870983) with 'a 17.5 s cancel is not less than a 1.88 s full create'. The relative timing check compared two durations taken about 10 s apart on a runner whose disk slowed about 23x in between: the same 202-commit commit-log phase took 0.3 s for the reference create and 7.0 s for the cancelled one. The cancel itself behaved correctly: it skipped ActivateSync and never reported step 'sync'. No sleep, retry or poll exists in CancelCreate, DeleteRepo, Archive, Purge, shutdown or closeFn.

Decisive measurement: with the original regression re-injected (Create always calls ActivateSync), `elapsed < fullCreate` still PASSED 3/3 (cancel 0.63 s against create 2.65-3.02 s). Only sawSync and the unit test went red. The check was blind to the bug it was written for, and a source of flakes.

Options:
1. Delete the timing check and the baseline create, and keep sawSync, the CreateCancelled terminal state and the no-trace checks. CHOSEN by the user. Test-only.
2. Add a second nil-in-production gate (same pattern as indexHealGate) to hold the heal mid-index under the branch lock, and prove the cancel lands while the index is provably unfinished. REJECTED: it needs a production hook. The prototype went 10/10 green and 3/3 red against the regression. The unit test already covers the mechanism deterministically.
3. Loosen to a multiple of a baseline. REJECTED: still a machine-speed assertion, and shown blind to the regression.

Why sawSync is enough end to end: under the regression the job parks on step 'sync' behind the heal's branch lock for the whole remaining index, so a 5 ms poll cannot miss it.

The 120 s deadline remains as a hang detector only.
