---
type: observation
domain: [repos, index, sse, testing, flaky]
confidence: 0.9
sources: 1
entities: [newMachine, Machine.lastPub, Machine.publish, Machine.computeStatus, indexProgressed, indexDone, publishIndex, TestRepoIndexEvents_StartupHealEmitsIndexingThenReady]
motifs: [seed-equals-first-state, absence-mistaken-for-result]
refs: ['kb://3ec012f5b4d2/kb/incidents/web/index-chip-stale-after-heal/1368e90f.md', 'kb://3ec012f5b4d2/kb/invariants/repos/index-state/publish-chokepoint/4110015c.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/lifecycle/mirror-indexing-first-emit-is-conditional/42cfd80d.md', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@e84e937c423445837a579b4bb38b27ffeedce26a:b97a280534f742aaeee7c9384fca4632be8c48c6', 'src://7b4887ce51d9/internal/corpus/repos/index_events_test.go@e84e937c423445837a579b4bb38b27ffeedce26a:ea44a4200b99efadb1a91e63551ca98d3f14ab38', 'src://7b4887ce51d9/internal/edge/web/handlers_repo_events_test.go@e84e937c423445837a579b4bb38b27ffeedce26a:68b17a1bd53956acfee97a08629a4eb75c1881ab', 'src://7b4887ce51d9/internal/corpus/repos/stages.go@e84e937c423445837a579b4bb38b27ffeedce26a:49c0c786dbce5e240e5b6f044ca55703ed50d6dc', 'https://github.com/knomit/knomit/issues/570', 'https://github.com/knomit/knomit/pull/640']
---
# FIXED (PR #640, issue #570): a mounted repo now publishes one unthrottled entry `indexing` event at its first publish, because newMachine seeds lastPub with the zero snapshot instead of the computed initial status

FIXED by PR #640 (issue #570, https://github.com/knomit/knomit/issues/570); merge commit e84e937c.

ORIGINAL DEFECT. newMachine (internal/corpus/repos/machine.go) seeded lastPub with the computed initial status, which already derives to Index.State = indexing (no verdict yet). publish() emits an index event only when state, reason or job CHANGED against lastPub, or progress MOVED while indexing, so the first publish compared indexing with indexing and announced nothing. The only `indexing` frame on a mount was then a throttled progress frame, present only if indexProgressed was handled before indexDone. Under load the stream carried just `ready`: the flake in TestRepoIndexEvents_StartupHealEmitsIndexingThenReady (internal/edge/web/handlers_repo_events_test.go), 6 of 1000 runs under 40 CPU burners with -race before, 0 of 1000 after.

FIX. newMachine seeds `m.lastPub = snapshot{}`, so the first publish (at Populate enter) is a change against the zero value and emits exactly one unthrottled `{state: indexing, done: 0, total: 0}` through publishIndex (the kb branch SSE via BroadcastIndex and /repo-events via RepoEventPublisher). Later Open/Index enters stay `indexing`, so there is no second entry event.

REGRESSION ANCHOR (deterministic, no load needed): TestRepoIndexEvents_MountAnnouncesEntryIntoIndexing (internal/corpus/repos/index_events_test.go) parks the index job at its hook, so no progress or terminal frame can exist, and requires the first IndexEvent for the repo to be `indexing`. Reviewer sabotage (seed reverted to the computed status) fails it 20/20; 200/200 green for it and the heal test with 10 busy-loop burners.

CONSEQUENCES / MISREADINGS.
- Every machine's first publish now emits an index event, including a mount that ends `unavailable` before an index life exists (the state derives to `indexing` while the verdict is nil). For a failed first mount the GET /repos row is Unavailable with no index_state, so the stream does NOT agree with REST there: the client patches index_state 'indexing' into any in-memory row of that name, nothing renders it (the chip is hidden on unavailable or unmountable rows), and the next refetch clears it. Indexing-then-unavailable with no terminal event is not new: a SwapStore whose Open fails already did it.
- Machine.Watch / watchIndex / streamLifecycle subscribers are unaffected: sendLocked runs on every publish regardless of `changed`, and a new Watch's first value is the computed snapshot, not lastPub.
- publishRebuildTask (the jobs SSE) is unaffected at mount: jobFull/job are empty until a Rebuild sets m.jobID.
- The invariant in kb/invariants/repos/index-state/publish-chokepoint/4110015c.md still holds: the event is built and sent only by Machine.publish, after a real transition; seeding the comparison baseline is not a publish. The fix makes "entry events are never throttled" literally true for a mount.
- General shape: a "last published" cell seeded with the first derived value can never announce that value.

DO NOT now treat the leading `indexing` in a test as a progress frame; it is the entry frame. A test that wants to exercise the throttled progress path must key on Total == 10 (or the job's real total), as TestRepoIndexEvents_TerminalSurvivesTheProgressThrottle does.
