---
type: observation
domain: [okf, export, architecture, cli]
confidence: 0.95
sources: 1
entities: [tools/okf, internal/okfsource, internal/okf, knomit-okf, EnsureOKF, MapperVersion]
refs: ['src://7b4887ce51d9/tools/okf/main.go@4baae4ed3043cbcb37a797b625f5e8a1a071f912:a54ced3cdc4c281cbe1ca92330700639ff280d97', 'src://7b4887ce51d9/internal/okf/source/source.go@4baae4ed3043cbcb37a797b625f5e8a1a071f912:59f3ccf26464f5c77ea32edd81a0e83d537707af', 'https://github.com/knomit/knomit/pull/41']
---
# OKF export is a standalone knomit-okf CLI, not a server feature — no generated refs, no cache, no server state

The OKF export runs as a standalone CLI (`knomit-okf clone` / `knomit-okf sync`) that reads a knowledge base over plain git. The knomit server generates nothing and stores nothing for OKF.

Options considered: (a) generate bundles inside the server, publish them as refs/heads/okf/<branch>, cache on a marker, regenerate lazily on the git advertise path; (b) a standalone tool that turns any KB git URL into a publishable OKF repository.

Rationale: (a) shipped first and was withdrawn. Holding state caused the problems, nearly all downstream of it: generated refs living in refs/heads/* were enumerated by every ref walker, so `knomit verify` reported one ERROR per bundle file (~1000 on a real corpus); one snapshot commit per regeneration grew the store unboundedly; generation sat on the unauthenticated /info/refs path at ~3.5s per branch with no singleflight; the unlocked read-modify-write on the ref raced; the marker key could not capture all its inputs, needing a hand-maintained MapperVersion; and the push guard panicked in a bare goroutine, which killed the server. Six commits on okf/main in the live store all recorded the SAME source commit — they tracked mapper releases, not knowledge.

The choice: (b). Determinism does the diffing — the mapper is pure, so re-exporting an unchanged source produces byte-identical output, git reports no change, and no commit is made. A commit therefore exists only when the exported KNOWLEDGE differs. Verified: two independent clones of the same source produce the identical output commit SHA, because export commits are timestamped from the source commit rather than the clock.

What this does NOT mean: the pure mapper in internal/okf was KEPT, unchanged — it was the valuable half. Only the server-side plumbing (EnsureOKF, okfWriteTree, the ref write, the marker, okf_trigger.go, the /okf/<branch>.tar.gz route, push_guard.go, okf_marker.go, MapperVersion) was deleted. It also does not mean the server can no longer be a source: the tool fetches over ordinary git, so knomit's own read-only smart-HTTP endpoint is a perfectly good URL to point it at.
