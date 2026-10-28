---
type: observation
domain: [repos, lifecycle, testing]
confidence: 0.85
sources: 1
entities: [Machine.Send, Machine.handleExternal, sendPastFastPath, MachineEvent.standingRefusal, ErrIndexing, TestWalkthrough_SubscriptionDetachDuringIndexing, TestMachine_TransitionTable]
motifs: [duplicated-check-one-tested, test-seam-bypass]
refs: ['kb://3ec012f5b4d2/kb/decisions/repos/lifecycle/indexing-exclusive-with-attach-detach-swap/cc2212e3.md', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@e60be957a54c485f321b85391310d10a7c8fb685:952a6e80e41aa3d94ccdd50ac1f0472ca6cff218#L254-L279', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@e60be957a54c485f321b85391310d10a7c8fb685:952a6e80e41aa3d94ccdd50ac1f0472ca6cff218#L347-L379', 'src://7b4887ce51d9/internal/corpus/repos/machine_test.go@e60be957a54c485f321b85391310d10a7c8fb685:f2687b3eb4af5633d9713f14a2f720915192e6aa#L150-L165']
---
# Every refusal Machine.Send answers early is answered AGAIN in handleExternal, and a test through Send can never reach the second site: use sendPastFastPath (machine_test.go) to hand an event to the driver past Send's snapshot checks

Machine.Send answers the cheap refusals (closed, unavailable, not-open, MachineEvent.standingRefusal, ErrIndexing) from the status snapshot before queueing; handleExternal re-checks the same conditions on the driver goroutine before the guard runs. Whenever the snapshot and the driver agree (the normal case in a test that holds the index at the index-job hook), Send's copy answers first, so a test that only calls Send (or Manager.Send) leaves the handleExternal copy untested — reorder or delete it and nothing fails.

CONSEQUENCE: to cover the second site, use sendPastFastPath(t, ri, e) in internal/corpus/repos/machine_test.go, which sets e.reply and sends &e straight onto ri.machine.external (with timeouts), exactly as Send's tail does. TestWalkthrough_SubscriptionDetachDuringIndexing and the 'ready, index running' subtest of TestMachine_TransitionTable assert both sites with it; swapping the order of standingRefusal and the index gate at either site alone fails a distinct assertion (checked by sabotage when added).

WHAT THIS DOES NOT MEAN: that every handleExternal copy is a live, load-bearing path. It differs by gate. For the index gate and the closed / unavailable / not-open gates the handleExternal copy is authoritative: the snapshot can be stale between a transition and its publish (an index that starts between Send's snapshot and the driver taking the event is a real production path, and sendPastFastPath models it). For standingRefusal it is NOT: it reads state fixed for the instance's whole life (r.subscribed), so Send has always already answered it and the handleExternal copy is defensive only, guarding a future edit to Send. sendPastFastPath therefore models a real production path for the ErrIndexing recheck, but not for the standing refusal; covering the standingRefusal copy tests that guard, not a reachable race. Send's copy, for every gate, is the early answer that lets a caller be refused while the driver is busy (e.g. inside a clone).
