---
kind: pragmatic
type: policy
domain: [web, origin-sessions, invariant]
confidence: 0.85
sources: 1
entities: [handleCommit, reopenSessionClone, openSessionClone, openSessionCloneFn, stateApplied, stateError, clone.db, guardSwap, inspectIncomingStore, SwapStore, sessionManager.Use, finalize]
motifs: [handle-outlives-target, retry-hits-leftover-state]
refs: ['kb://3ec012f5b4d2/kb/invariants/web/origin-sessions/ad44382e.md', 'kb://3ec012f5b4d2/kb/invariants/repos/one-local-copy-per-knowledge-base/b27a57c6.md', 'kb://3ec012f5b4d2/kb/gotchas/web/origin-sessions/e3474a59.md', 'src://7b4887ce51d9/internal/edge/web/api/origins/handlers.go@20c800a017123c5257ce055b3d0eb41c0fb2c3c7:dc7512ffa9ccc270e945a6973ed08f3bd02083f6#L878-L900', 'src://7b4887ce51d9/internal/edge/web/api/origins/commit_retry_test.go@20c800a017123c5257ce055b3d0eb41c0fb2c3c7:646d5ec0b1aa31818bf53f08d6169c5b4e0261d9', 'https://github.com/knomit/knomit/issues/416']
---
# A SWAP commit closes the session clone only inside the run, immediately before the send, and reopens it via openSessionClone + OpenRepo when repos.IsRefused; the clone must stay closed during the send, an ATTACH never closes it, and a failed reopen puts the session in stateError

handleCommit (internal/edge/web/api/origins/handlers.go) for a swap: after the cancel_indexing step it checkpoints and closes the clone and sets sess.RemoteStore=nil, then sends SwapStore. The clone MUST stay closed for the whole send: guardSwap's inspectIncomingStore opens the SAME clone.db with its own store.Open (migrate.All and PRAGMA optimize write; sessions.Open deletes and recreates the shared session sidecar and Service.Close deletes it again), and the install copies only the main file, so a live writer or WAL corrupts or loses data. Closing the clone only AFTER the swap is accepted (the fix proposed in #416) is therefore wrong.

If the send fails with repos.IsRefused, reopenSessionClone reopens clone.db with openSessionClone (the same setup /test uses: store.Open, SetSigner, SetAgentID, SetAcceptList) then OpenRepo, reattaches it under sess.mu, and leaves the state at stateApplied, so the user can fix the cause and commit again. If the reopen fails: Warn, state stateError, the ORIGINAL refusal is still returned, and the next commit gets 409 'applied state' (create a new session). An ATTACH never touches clone.db; sessionManager.Delete -> finalize closes the store on success.

WHAT THIS DOES NOT MEAN: it does not make every failed swap retryable. A non-refused failure (a bare context error, applySwap errors after the exits) leaves the clone closed and detached, so the retry still hits 'session has no remote store'. The reopen is safe against Delete/finalize because it runs while the run still holds the session's exclusive use.
