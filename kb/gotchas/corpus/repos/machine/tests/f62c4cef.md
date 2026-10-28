---
type: observation
domain: [repos, lifecycle, testing]
confidence: 0.85
sources: 1
entities: [Machine.Send, Machine.handleExternal, sendPastFastPath, MachineEvent.standingRefusal, ErrIndexing, TestWalkthrough_SubscriptionDetachDuringIndexing, TestMachine_TransitionTable]
motifs: [duplicated-check-one-tested, test-seam-bypass]
refs: ['kb://3ec012f5b4d2/kb/decisions/repos/lifecycle/indexing-exclusive-with-attach-detach-swap/cc2212e3.md', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@13e90f4c1222aab9853e6302389a4e0930e685b1:952a6e80e41aa3d94ccdd50ac1f0472ca6cff218#L254-L279', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@13e90f4c1222aab9853e6302389a4e0930e685b1:952a6e80e41aa3d94ccdd50ac1f0472ca6cff218#L345-L379', 'src://7b4887ce51d9/internal/corpus/repos/machine_test.go@13e90f4c1222aab9853e6302389a4e0930e685b1:f2687b3eb4af5633d9713f14a2f720915192e6aa']
---
# Every refusal Machine.Send answers early is answered AGAIN in handleExternal, and a test through Send can never reach the second site: use sendPastFastPath (machine_test.go) to hand an event to the driver past Send's snapshot checks

Machine.Send answers the cheap refusals (closed, unavailable, not-open, MachineEvent.standingRefusal, ErrIndexing) from the status snapshot before queueing; handleExternal re-checks the same conditions on the driver goroutine before the guard runs. Whenever the snapshot and the driver agree (the normal case in a test that holds the index at the index-job hook), Send's copy answers first, so a test that only calls Send (or Manager.Send) leaves the handleExternal copy untested — reorder or delete it and nothing fails.

CONSEQUENCE: to cover the second site, use sendPastFastPath(t, ri, e) in internal/corpus/repos/machine_test.go, which sets e.reply and sends &e straight onto ri.machine.external (with timeouts), exactly as Send's tail does. TestWalkthrough_SubscriptionDetachDuringIndexing and the 'ready, index running' subtest of TestMachine_TransitionTable assert both sites with it; swapping the order of standingRefusal and the index gate at either site alone fails a distinct assertion (checked by sabotage when added).

WHAT THIS DOES NOT MEAN: that the handleExternal copy is redundant. It is the authoritative check — the snapshot can be stale between a transition and its publish — and Send's is the early answer that lets a caller be refused while the driver is busy (e.g. inside a clone).
