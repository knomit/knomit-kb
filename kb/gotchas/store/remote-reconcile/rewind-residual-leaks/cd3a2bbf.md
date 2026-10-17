---
type: observation
domain: [store, sync, git, scrub, gap]
confidence: 0.85
sources: 1
entities: [replayOntoUpstream, strictlyAhead, readAgentBase, refs/knomit/agent-base, seedMissingWatermark, unpushedCommitsDropping]
motifs: [noop-keeps-bad-state, stale-watermark]
refs: ['src://7b4887ce51d9/internal/store/remote_reconcile_rebase.go@ba40165b1234cabca66cc59a44e24e7dace9c48a:f5fcc2cfa30de1aafe7e1c2660f7c545051ebfc8', 'src://7b4887ce51d9/internal/store/remote_reconcile_empty_walk_test.go@ba40165b1234cabca66cc59a44e24e7dace9c48a:9a847a9b2a1b38683b33e6ea884c251905be7d22', 'kb://3ec012f5b4d2/kb/gotchas/store/remote-reconcile/merges-only-pushed-as-is/75e0108b.md', 'kb://3ec012f5b4d2/kb/invariants/store/remote-reconcile/replay-stop/5482c3ab.md', 'kb://3ec012f5b4d2/kb/decisions/store/disjoint-replay-with-dead-refs/37c555b5.md', 'https://github.com/knomit/knomit/issues/541', 'https://github.com/knomit/knomit/pull/542', 'https://github.com/knomit/knomit/pull/539']
---
# KNOWN GAPS in the rewind's scrub protection after #539: a disjoint upstream keeps the branch (tracked as issue #541), a repo with no watermark that is reset to an ancestor still pushes the removed commit (being fixed on branch fix/watermark-fallback, PR #542), and an own write above the watermark that edits a removed file brings the file back (accepted)

READ THIS BEFORE SAYING 'nothing removed upstream returns after a rewind'. At merge ba40165b (#539 + #540) the rewind path rebuilds a branch on shared history whose replay walk is empty (incident: kb/incidents/store/remote-reconcile/noop-rewind-leak/), but three shapes still keep or return removed content:

(a) DISJOINT UPSTREAM. A history rewritten down to its ROOT to scrub something shares no commit with the local branch. With nothing of the agent's own past the watermark, replayOntoUpstream keeps the no-op ('nothing to replay onto a disjoint upstream; the branch stays as it is') and the force-push sends the scrubbed history back. It cannot simply be rebuilt: attaching a fresh local repo to a remote of the same knowledge base is the same shape (its watermark is the InitRepo seed, never on any origin), and taking the remote's main there changes the repository's identity (its root commit); tests TestSwap_CopyFailureRestoresStoreAndOrigin and TestHandleCommit_Disjoint_FailedSwapRestartsThePreviousSyncMode pin that identity. Closing it needs a signal that the watermark was ever on the origin. STATUS: the user ruled (2026-10-17) that this is tracked as enhancement issue #541 (stop, show the delta, let the user choose what to keep; delta export), not a silent rebuild. Also pre-existing: a local repo WITH writes that is attached to a disjoint remote already loses its seed root, so its ID changes; the identity stability those two tests pin holds only for a repo with no writes.

(b) NO WATERMARK. A repo with NO watermark ref (refs/knomit/agent-base/<branch>; made before it existed and not synced since) uses the merge base as its stop point, and strictlyAhead without a watermark is the upstream-ancestor test alone. If the origin resets main to an ancestor of the branch, the branch reads as strictly ahead, keeps the removed commit and pushes it. STATUS: a GAP being FIXED on branch fix/watermark-fallback (PR #542): sync() captures refs/remotes/origin/<main> before the fetch moves it and seeds the missing watermark there (where the origin was at the last fetch); a truly first sync seeds nothing and the merge-base fallback stays. Until #542 merges, this shape leaks. Do not read #539 as covering it.

(c) AN OWN WRITE ABOVE THE WATERMARK THAT EDITS A REMOVED FILE. The scrubbed commit stays unreachable, but the file comes back with the edited body (the local side wins), and its path and content are pushed. Pre-existing (the same on dev before #539). STATUS: accepted by the user (2026-10-17): 'this is up to however manages consensus (main)'.

NOT A GAP BUT A COST: the rewind drops CONSUMED own commits the new upstream lacks (kb/decisions/store/remote-reconcile/consumed-own-work-dropped/); the user accepted that too.

MISREADINGS. 'The disjoint no-op is intended behaviour': it is a known residual, kept only because closing it breaks attach identity. 'Issue #541 is the fix for the no-watermark case': no, #541 is the root-deep rewritten-origin case; the no-watermark fix is PR #542.
