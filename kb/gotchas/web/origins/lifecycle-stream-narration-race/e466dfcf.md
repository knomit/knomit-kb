---
type: observation
domain: [web, origins, lifecycle, testing, flaky]
confidence: 0.9
sources: 1
entities: [streamLifecycle, Machine.Watch, emitPhase, TestHandleCommit_Disjoint_HappyPathSavesOrigin, internal/edge/web/api/origins/handlers.go]
motifs: [lossy-channel-narration, fallback-guard-too-narrow]
refs: ['src://7b4887ce51d9/internal/edge/web/api/origins/handlers.go@fbed4a49ff6dfa5cf45f459b8fd888c13a4d77ec:305cbe0d6ea790f99f94ab57e7a97ef72058f018', 'src://7b4887ce51d9/internal/edge/web/api/origins/commit_narration_test.go@fbed4a49ff6dfa5cf45f459b8fd888c13a4d77ec:d0a8e49e8397be73740625c8a574cc44ea2af7f8', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@fbed4a49ff6dfa5cf45f459b8fd888c13a4d77ec:59276d39a47bacc416d96d534697ba1708559f14', 'https://github.com/knomit/knomit/pull/632', 'https://github.com/knomit/knomit/pull/634']
---
# FIXED in #634 (fbed4a49): the origins commit stream could say swapping and done but never configuring, because the lossy cap-1 Machine.Watch dropped open/identify; streamLifecycle now drains Watch after a successful Send and falls back to configuring whenever phase != configuring

FIXED by PR #634 (merged as fbed4a49; re-read there). Found during PR #632 as a flaky TestHandleCommit_Disjoint_HappyPathSavesOrigin ('the stream must narrate phase configuring').

CAUSE (still the shape to recognise). internal/edge/web/api/origins/handlers.go streamLifecycle narrates a store swap from Machine.Watch snapshots. Watch is a lossy cap-1 broadcaster: it delivers the latest value and drops the ones in between. emitPhase narrates 'configuring' only when it OBSERVES an open or identify snapshot. Before the fix the post-Send fallback fired only when no phase had been narrated (phase == ""), and the loop exited on the done result without draining Watch. So when the machine ran Open and Identify faster than the handler read its channel, the stream carried swapping and done with no configuring (reproduced about 1 in 900 under load; 1 in one full run; 0 of 20 in isolation).

FIX. After a successful Send (sendErr == nil; a failed Send returns before the drain and the fallback), streamLifecycle drains Machine.Watch without blocking (a select with a default; a closed channel sets watch = nil) and emitPhase's each pending value, then emits 'configuring' whenever phase != "configuring". Stream order is swapping, configuring, done even when the lossy channel dropped open and identify. phase only moves forward: swapping is emitted only when phase == "", configuring only when phase != "configuring".

DOES NOT MEAN: that the drain is what guarantees configuring. The status read after done is 'ready', which emitPhase ignores; the phase != "configuring" fallback is the guarantee, and the drain only lets late open/identify snapshots narrate in order. Nor does it mean the stream observes every machine stage: Watch is still lossy, so a consumer must not assume each intermediate phase was seen, only that configuring precedes done after a successful Send.

TEST. internal/edge/web/api/origins/commit_narration_test.go TestHandleCommit_Disjoint_NarratesConfiguringWhenTheReaderIsLate holds the machine at populate via Options.Hook and stalls the reader in its 'swapping' write; it is red without the fix (stream 'swapping, done'). Its proceed helper polls Status().Stage != "ready" with a 10s deadline, so a renamed stage degrades to a 10s stall then the assertion failure.

CONSEQUENCE. An occasional TestHandleCommit_Disjoint_HappyPathSavesOrigin failure is no longer explained by this race; investigate it as something else. KNOWN LOW (#634 review): the drain is not bounded by a count; theoretical, since the watch is cap-1 and the machine is idle once Send has returned.
