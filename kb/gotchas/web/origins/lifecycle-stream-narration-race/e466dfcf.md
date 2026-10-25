---
type: observation
domain: [web, origins, lifecycle, testing, flaky]
confidence: 0.8
sources: 1
entities: [streamLifecycle, Machine.Watch, emitPhase, TestHandleCommit_Disjoint_HappyPathSavesOrigin, internal/edge/web/api/origins/handlers.go]
motifs: [lossy-channel-narration, fallback-guard-too-narrow]
refs: ['src://7b4887ce51d9/internal/edge/web/api/origins/handlers.go@ee850fd6cef7c936fc81af311371e34a6971350a:f079275bc79f1c9a652e76e61422e216c7fecaec', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@ee850fd6cef7c936fc81af311371e34a6971350a:59276d39a47bacc416d96d534697ba1708559f14', 'https://github.com/knomit/knomit/pull/632']
---
# The origins commit stream can say swapping and done but never configuring: streamLifecycle exits on done without draining the lossy cap-1 Machine.Watch channel, and its configuring fallback fires only when no phase was sent (pre-existing, unfixed)

Seen during PR #632 (merged as ee850fd6; re-read there); the race is PRE-EXISTING and unfixed. internal/edge/web/api/origins/handlers.go streamLifecycle narrates a store swap from Machine.Watch snapshots. Watch is a lossy cap-1 broadcaster: it delivers the latest value and drops the ones in between. emitPhase narrates 'configuring' only when it OBSERVES an open or identify snapshot, and the fallback after the Send completes (`if phase == "" { configuring }`) fires only when nothing was narrated. Failure shape: the watcher reads a populate snapshot (so it sends 'swapping'), the machine then runs Open, Identify and onward faster than the handler reads its channel, and the done result wins the select; the loop exits without draining Watch, and the stream carries swapping and done with no configuring. Observable as TestHandleCommit_Disjoint_HappyPathSavesOrigin failing with 'the stream must narrate phase configuring'.

EVIDENCE. 1 failure in one full go1.26.9 run at the PR head; 0 of 20 in isolation there; reproduced about 1 in 900 on dev 37dc65d5 (before 14c touched anything) under load, so the dispatcher's start in Serve is not the cause (at most its self-kick adds CPU load). PROPOSED FIX, not made: drain the Watch channel after done, then emit configuring when the Send succeeded and phase != "configuring" (the existing fallback only covers phase == ""). CONSEQUENCE: treat an occasional failure of that test under load as this race, not as a repos or trigger regression, until it is fixed.
