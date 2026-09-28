---
type: observation
domain: [fleet, auth, decisions]
confidence: 0.95
sources: 1
entities: [auth.CertGrants, auth.PushOwn, knomit grants]
refs: ['kb://3ec012f5b4d2/kb/decisions/auth/instance-read-implicit/85dd6b5d.md', 'kb://3ec012f5b4d2/kb/architecture/store/git-serve/receive-pack/7642c213.md', 'src://7b4887ce51d9/internal/auth/certgrants.go@c7423c3e3a7ad8280a73b9269d1f416844d14656:90f075df7e1f0abb4e733f50bee69308404a6d19', 'https://github.com/knomit/knomit/pull/340']
---
# Every enrolled instance may push its own agent branch to a host without a per-peer grant: push:own is implicit for chained instance certificates (user ruling "1. A", F11)

OPTIONS (as put to the user, 2026-09-28, F11 RCA open decision 1): (A) every enrolled instance automatically — `CertGrants` gains `push:own` beside `read`; (B) only instances the host's operator grants it to, one `knomit grants add instance:<fingerprint>@cert push:own` per peer. The question named that (A) REVERSES `kb/decisions/auth/instance-read-implicit` (ruling 5: the chain is admission, not authorisation; everything beyond `read` is a grants row).

CHOICE (user, verbatim): "1. A".

RATIONALE: `push:own` lets a peer do exactly one thing — update ITS OWN agent branch on the host; receive-pack refuses every other ref. A per-peer grant was a step every operator would always take, the kind of per-instance knob the user has been stripping (R7). Built in PR #340: `auth.CertGrants` returns `{read, push:own}` for KindInstance + ViaCert + non-empty ID.

REJECTED: the per-peer `push:own` grant.

NON-SCOPE: this reverses ruling 5 for `push:own` ONLY. `write`, `merge:main`, `operator` and `admin` remain grants rows; an operator-role certificate still holds only its rows; anonymous loopback, socket bridges and bearer tokens cannot push at all (no agent branch; the OAuth listener has no /git).
