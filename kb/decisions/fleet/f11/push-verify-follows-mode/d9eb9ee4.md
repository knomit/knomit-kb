---
type: observation
domain: [fleet, verify, decisions]
confidence: 0.95
sources: 1
entities: [verify_signatures, judgeCommit, CheckRange, receivePack.verify]
refs: ['kb://3ec012f5b4d2/kb/architecture/store/git-serve/receive-pack/7642c213.md', 'src://7b4887ce51d9/internal/store/httphandler_receivepack.go@c7423c3e3a7ad8280a73b9269d1f416844d14656:d4c5a0a6f3a5384d6fb9f9c54a2d5942389786f5', 'https://github.com/knomit/knomit/pull/340']
---
# Signature checks on a push follow the repo's verify_signatures, read at the HOST's main: off accepts unchecked, log accepts and logs, enforce refuses naming the commit (user ruling "3. yes.", F11)

OPTIONS (F11 RCA D-verify / open decision 3): (a) mode-driven, read at the host's upstream tip exactly as CheckRange reads it at Main; (b) always verify pushed commits; (c) never verify on push, leave it to the merge gate.

CHOICE (user, verbatim): "3. yes." — confirming (a).

RATIONALE: R2 ("what we discussed was ONLY to turn signing checking on or off via verify_signatured in the ontology") and the trust ruling: capabilities, not verifying the world. Reading the mode at the HOST's main means a pushed branch cannot choose its own check by editing its ontology. Accepted keys are the fleet repository's member records (judgeCommit); no key list anywhere. `enforce` with no fleet repository refuses (the gate could not run); `log` logs and accepts.

CONSEQUENCE stated to the user: the original design's "authentication IS F09" holds only when a repo turns verification on; with `off` (the default) a push is authenticated by the certificate and the own-branch rule alone.

REJECTED: an always-on signature check on push.
