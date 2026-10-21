---
type: observation
domain: [repos, index, sse, testing, flaky]
confidence: 0.9
sources: 1
entities: [newMachine, Machine.lastPub, Machine.publish, Machine.computeStatus, indexProgressed, indexDone, publishIndex, TestRepoIndexEvents_StartupHealEmitsIndexingThenReady]
motifs: [seed-equals-first-state, absence-mistaken-for-result]
refs: ['src://7b4887ce51d9/internal/repos/machine.go@301154c5bbf111c27142fc7e73ebfa9884268a17:6abc9980c7e763ad4c0a5dbf95737ea48067d273', 'src://7b4887ce51d9/internal/repos/stages.go@301154c5bbf111c27142fc7e73ebfa9884268a17:4281176c5654d70e094e29310cc48a94127133b7', 'src://7b4887ce51d9/internal/web/handlers_repo_events_test.go@301154c5bbf111c27142fc7e73ebfa9884268a17:b2cf0df82139b41d217c59e5a6a8bf5d9745f16a', 'kb://3ec012f5b4d2/kb/incidents/web/index-chip-stale-after-heal/1368e90f.md', 'kb://3ec012f5b4d2/kb/invariants/repos/index-state/publish-chokepoint/4110015c.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/lifecycle/mirror-indexing-first-emit-is-conditional/42cfd80d.md', 'https://github.com/knomit/knomit/issues/570']
---
# A mounted repo publishes NO entry `indexing` event: newMachine seeds lastPub with a status that is already `indexing`, so the only `indexing` frame is a throttled progress frame, present only if indexProgressed is handled before indexDone (issue #570)

A fresh Machine seeds lastPub with its initial status (internal/repos/machine.go, newMachine: m.lastPub = m.computeStatus()), and computeStatus reports Index.State = indexing whenever the job's verdict (r.indexVerdict) is nil, which it is at construction. publish() emits an index event only when the state, reason or job CHANGED against lastPub, or when progress MOVED while indexing. So the first publish after a mount compares indexing against indexing and emits no entry `indexing` event.

CONSEQUENCE: the only `indexing` frame on the SSE stream is a throttled progress frame (publishIndex(s, true); progress events are throttled, entry and terminal events are not), and it appears only if the driver handles indexProgressed BEFORE indexDone. indexProgressed is posted by the index job's progress callback and coalesced into at most one pending internal event (postProgress); indexDone is posted when the job returns (internal/repos/stages.go). When the job is fast, or the machine is loaded, indexDone can be handled first: the verdict is set, the stream carries only `ready`, and a later indexProgressed publishes nothing new. A test or client that expects `indexing` ... `ready` for a mount can therefore see just `ready`.

This is the load flake in TestRepoIndexEvents_StartupHealEmitsIndexingThenReady (internal/web/handlers_repo_events_test.go), tracked in issue #570 (https://github.com/knomit/knomit/issues/570). The proposed fix is to seed lastPub = Status{} so that the first publish is a change and emits the entry `indexing` event; it was not applied at 301154c5.

MISREADING: this does NOT mean the stream can get stuck on `indexing`. Terminal frames (ready, error) are always published when the verdict lands, because the state changes against lastPub and entry and terminal events are never throttled. The defect is a missing entry frame, not a missing terminal one, and the REST status is correct throughout. Do not weaken the test to accept `ready` alone as a fix, and do not add sleeps to give the progress frame time: the order is decided by the driver's inbox, not by the clock.
