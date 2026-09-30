---
type: observation
domain: [repos, triggers, testing, race, ci, performance]
confidence: 0.85
sources: 1
entities: [TestDispatch_WriteLatencyIndependentOfTriggers, TestDispatch_WriteLatencyIndependentOfTriggers_BusyIf, within, withinMedian, withinRace, raceEnabled, internal/repos/triggers_perf_test.go, internal/repos/race_on_test.go, .github/workflows/race.yml]
motifs: [ratio-gate-under-noise, signal-inside-noise-band]
refs: ['src://7b4887ce51d9/internal/repos/triggers_perf_test.go@21b369a468a62b7d2da6736d3b07c809c9b8988b:7e9b30a676bc78c67b7ce769638b3dd60d9adb9e', 'src://7b4887ce51d9/internal/repos/race_on_test.go@21b369a468a62b7d2da6736d3b07c809c9b8988b:b8ed0e272039431adfa7d34542a79acb72d2f389', 'src://7b4887ce51d9/internal/repos/triggers.go@21b369a468a62b7d2da6736d3b07c809c9b8988b:890f76c84c3840c491f141aac8f39602b1589630', 'src://7b4887ce51d9/internal/repos/builder.go@21b369a468a62b7d2da6736d3b07c809c9b8988b:447d2d2ef03d8049e7c57975127d8a73102cd70a#L670-L684', 'kb://3ec012f5b4d2/kb/decisions/fleet/f07-triggers/async-never-in-write-path/faa2faba.md', 'https://github.com/knomit/knomit/pull/354', 'https://github.com/knomit/knomit/actions/runs/36457474102', 'https://github.com/knomit/knomit/actions/runs/36614313171']
---
# Under `go test -race` on the shared ubuntu runner, the trigger-dispatch write-latency tests' loaded arm measured 1.25–1.50× the base purely from CPU contention (busy `if`s spin a core each to the 100 ms interrupt while other packages test concurrently). The flush-per-run sabotage of the main test is only ~1.66×, so under -race that test LOGS the ratio and does not gate; _BusyIf gates at max(3×, +250ms)

Race workflow evidence as loaded vs base median. _BusyIf: 16.2/12.9 ms (run 36354285281), 25.3/19.8 (36457474102), 17.5/11.7 (36508768596), 12.9/10.3 (36601821120), 10.3/8.2 (36614313171). Main test: 62.6/41.7 = 1.503× (36457474102), against its bound of max(1.5×, +3ms), and 31.9/24.1 = 1.32× (36354285281), against the then-current 1.25× bound (that log line predates the '(report only)' format). The 1.25–1.50× range is drawn from both tests. The old bounds were max(1.25×, +2ms) for _BusyIf and max(1.5×, +3ms) for the main test.

Local checks on an idle 8-core arm64 machine with -race: ratio 0.92–1.07, all pass. With `-cpu 2` plus 6 CPU burners, 1 of 3 runs failed at 1.34×. So the excess is contention, not synchronous dispatch. Dispatch is still asynchronous; the writer simply loses CPU to the spinning `if`s.

What changed (branch fix/race-workflow-flakes): a `raceEnabled` const in internal/repos (race_on_test.go / race_off_test.go, mirroring internal/fact).
- The main test t.Logf's the ratio and does NOT gate when raceEnabled. No bound can separate its flush-per-run sabotage (~1.66×) from race noise (up to 1.50×) there.
- _BusyIf gates with withinRace = max(3×, +250ms) under -race.
- Both tests keep RUNNING under -race, for race coverage. The non-race gates are unchanged, and the non-race run in tests.yml is still the latency gate.

Sabotage still caught under -race: make the writer wait for the dispatcher run it kicked. triggerKick blocks on the kick, then polls completedHead. _BusyIf then measured 1.03 s vs 14 ms (73×) and failed. The reviewer ran an independent proxy sabotage (timedWrites calling waitTriggerHeadFor after each write) and got 1.063 s vs 24.5 ms (43×). Same conclusion; the magnitude differs because that proxy includes the flush grace, which inflates the base too.

A naive sabotage does NOT work as a check. Calling d.safeRun(d.runCtx) directly from triggerKick runs the dispatcher inside ri.onCommit, and onCommit runs under the writer's branch lock (store notifyCommit; the comment on the ri.onCommit closure in builder.go). The test then stalled and failed after 180 s on 'the dispatcher never completed and flushed a run', without ever measuring latency. The mechanism was not traced. There are two candidates, and neither is verified: (1) the synchronous run needs the store or branch lock that the writer already holds inside notifyCommit; (2) the run happens CONCURRENTLY with the dispatcher's own goroutine, which the single-actor design (completedHead updated in run's defer) does not allow. Either way, running the dispatcher inside triggerKick is not a valid sabotage for a latency test.

Not meant: this is not 'perf gates are meaningless under -race'. A gate whose sabotage signal is far outside the noise band (_BusyIf, ~70×) stays useful under -race with a loose bound.
