---
type: observation
domain: [synthesize, testing, concurrency]
confidence: 0.9
sources: 1
entities: [benignLoserRefusal, requireOnlyBenignErrors, TestReviewer_ConcurrentContinuations_ApplyOnce, TestHypothesizer_ConcurrentDiscoverSubmission_WritesOnce, errAdvancing, AdvancePipelineSessionPhase, FinishPipelineSessionAdvance, knomit#305]
motifs: [allowlist-drift, narrow-race-window]
refs: ['src://7b4887ce51d9/internal/synthesize/review_claim_test.go@5055edfb2f75e72ea665fe4f8ae7bb75d6449ce6:9bbbdaea9c058b6d694097e680d808d33524d0f8#L194-L239', 'src://7b4887ce51d9/internal/synthesize/hypothesize_engine_test.go@5055edfb2f75e72ea665fe4f8ae7bb75d6449ce6:8cc556980b6f08e64cac6ae2294ab8dc5fb5fdc7#L316-L332', 'src://7b4887ce51d9/internal/synthesize/pipeline.go@5055edfb2f75e72ea665fe4f8ae7bb75d6449ce6:451a628a325e7edb70b75feb6e515b50298dc7a3#L280-L284', 'src://7b4887ce51d9/internal/store/pipeline_index.go@5055edfb2f75e72ea665fe4f8ae7bb75d6449ce6:15299d59a34b2d62be8d508a72093018818a3779#L402-L430', 'kb://3ec012f5b4d2/kb/decisions/synthesize/testing/benign-loser-refusals/1ea63aa5.md', 'https://github.com/knomit/knomit/issues/305']
---
# A concurrent-submission test in internal/synthesize that tolerates only a LISTED set of loser refusals goes flaky whenever the engine gains a new refusal a loser can hit — the list must be the one shared benignLoserRefusal pattern, not a private copy per test

The apply-once tests in internal/synthesize run two ContinueSession calls on the same work item and assert the fact count. The LOSER's error is not nil and must not be: it is refused with one of three messages, each meaning "someone else handled the item":

- `is completed, not active` — arrived after the winner completed the session;
- `is being applied by another caller` — arrived while the winner held the item's applying claim;
- `is moving to its next phase for another caller` — errAdvancing, raised when the loser re-reads the session and finds `advancing` set. The winner's phase CAS (store AdvancePipelineSessionPhase) sets it and FinishPipelineSessionAdvance clears it after the phase hook returns, so a loser sees it only in that window.

Before knomit#305 the tolerated list was a regex inside requireOnlyBenignErrors that named only the first two, and TestHypothesizer_ConcurrentDiscoverSubmission_WritesOnce discarded its errors entirely (so it could not flake, but could not catch a real error either). The advancing window is narrow, so the reviewer test passed for a long time and then flaked.

Consequence: when you add a refusal a racing caller can legitimately hit, add it to `benignLoserRefusal` (review_claim_test.go) — the ONE definition both tests use through requireOnlyBenignErrors. Do not give a new concurrency test its own regex, and do not "fix" a flake by discarding errors or matching any error: an unexpected refusal must still fail.

Not a production defect: the engine refused correctly; only the test's allowlist was stale. Collecting errors needs one slot per goroutine, read only after the WaitGroup returns, to stay race-clean.
