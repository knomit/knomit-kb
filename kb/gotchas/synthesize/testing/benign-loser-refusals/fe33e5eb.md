---
type: observation
domain: [synthesize, testing, concurrency]
confidence: 0.9
sources: 1
entities: [benignLoserRefusal, requireOnlyBenignErrors, TestReviewer_ConcurrentContinuations_ApplyOnce, TestHypothesizer_ConcurrentDiscoverSubmission_WritesOnce, TestReviewer_ConcurrentContinuations_EnqueueReflectOnce, TestReviewer_ReflectGap_SecondCallerDoesNotComplete, errAdvancing, handlePhase, completeSession, AdvancePipelineSessionPhase, FinishPipelineSessionAdvance, knomit#305]
motifs: [allowlist-drift, narrow-race-window]
refs: ['src://7b4887ce51d9/internal/corpus/synthesize/review_claim_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:6d1e273074a28df92d6515f5aceb3333f68afe59', 'src://7b4887ce51d9/internal/corpus/synthesize/hypothesize_engine_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:19833ba659d231ededb78ae0bd8f181a6ef57074', 'src://7b4887ce51d9/internal/corpus/synthesize/pipeline.go@0610c20083fcef9e84f42ab0311a14708bcb6193:a539c47c8a7376d5fb3fbf59d30742e2f1645e33', 'src://7b4887ce51d9/internal/engine/store/sessions/pipeline_index.go@0610c20083fcef9e84f42ab0311a14708bcb6193:1b2462d11d53a7c85f449f7fd63dcb5f2a6fbcc6', 'src://7b4887ce51d9/internal/corpus/synthesize/review_phase_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:277aa5764c3c906104f16a2800048db98f4cb2d3', 'kb://3ec012f5b4d2/kb/decisions/synthesize/testing/benign-loser-refusals/1ea63aa5.md', 'https://github.com/knomit/knomit/issues/305', 'https://github.com/knomit/knomit/pull/309']
---
# A test in internal/corpus/synthesize where callers race to CLAIM the same work item and the proof is the fact count must tolerate loser refusals through the ONE shared benignLoserRefusal pattern, not a private list — a private list goes flaky when the engine gains a refusal; but the reflect-gap tests, whose proof is that no caller completes, must NOT use it

The apply-once tests in internal/corpus/synthesize (TestReviewer_ConcurrentContinuations_ApplyOnce, which fires four callers, and TestHypothesizer_ConcurrentDiscoverSubmission_WritesOnce, two) run two or more ContinueSession calls on the same work item and assert the fact count. The LOSERS' errors are not nil and must not be: each is refused with one of three messages, each meaning "someone else handled the item":

- `is completed, not active` — arrived after the winner completed the session;
- `is being applied by another caller` — raised in handlePhase (pipeline.go) when the loser lost the phase CAS while the winner's item is still applying;
- `is moving to its next phase for another caller` — errAdvancing, raised in handlePhase and in completeSession when the loser re-reads the session and finds `advancing` set. The winner's phase CAS (store AdvancePipelineSessionPhase) sets it and FinishPipelineSessionAdvance clears it after the phase hook returns, so a loser sees it only in that window.

Before knomit#305 the tolerated list was a regex inside requireOnlyBenignErrors that named only the first two, and TestHypothesizer_ConcurrentDiscoverSubmission_WritesOnce discarded its errors entirely (so it could not flake, but could not catch a real error either). The advancing guard (#291) added the third refusal without anyone updating the list, and the reviewer test flaked.

Consequence, SCOPED: in a test where callers race to CLAIM the SAME work item and the apply-once proof is the fact count, tolerate losers only through `benignLoserRefusal` (review_claim_test.go) via requireOnlyBenignErrors, and when you add a refusal such a loser can legitimately hit, add it there — once. Do not give such a test its own regex, and do not "fix" a flake by discarding errors or matching any error: an unexpected refusal must still fail.

NOT for the reflect-gap tests in review_phase_test.go — TestReviewer_ConcurrentContinuations_EnqueueReflectOnce (its per-caller check matches only "retry shortly") and TestReviewer_ReflectGap_SecondCallerDoesNotComplete. Their guarantee is that no caller completes the session in the gap; benignLoserRefusal accepts "is completed, not active", which is exactly the regression they exist to catch. Leave them on "retry shortly".

Not a production defect: the engine refused correctly; only the test's allowlist was stale. Collecting errors needs one slot per goroutine, read only after the WaitGroup returns, to stay race-clean.
