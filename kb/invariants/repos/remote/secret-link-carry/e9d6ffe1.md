---
kind: pragmatic
type: policy
domain: [repos, remote, secrets]
confidence: 0.85
sources: 1
entities: [OriginSpec.SecretID, applyAttach, applySwap, restoreFleetOrigin, setOriginUpstream, linkedOriginSpec, remotesync.Origin.SecretID]
motifs: [full-replace-drops-fields]
refs: ['src://7b4887ce51d9/internal/corpus/repos/events.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:88e53033e81f2b3690c7cae3bb67e5d074f25c13', 'src://7b4887ce51d9/internal/corpus/repos/secret_link_carry_test.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:3f0d5fd0818d1e75602ebf767efeb9289dc0c181', 'src://7b4887ce51d9/internal/corpus/repos/control/origins.go@968ad8e4bd19903426eadbf9cc838ec0bd8203ab:766cafa3f7696a93d4bdc67edd9400088913230e', 'src://7b4887ce51d9/internal/edge/web/api/manage/handlers_origin_hal.go@5a75cc87fd5021139e3e370cf93833703efdd610:928e668e582401b9ca6134561919cf511835eaa5', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/secret-link/845c96d6.md']
---
# OriginSpec.SecretID IS the link on attach, swap, create and fleet restore: a caller that rebuilds a spec from the stored origin must copy SecretID, or the attach silently unlinks the repo

applyAttach and applySwap (internal/corpus/repos/events.go) write control.Origin and the injected remotesync.Origin from the OriginSpec they are given, including SecretID; Origins.Set is a full replace, so a spec with an empty SecretID DELETES the repo's link. The paths that rebuild a spec from stored state carry it: create (lifecycle.go persists cs.Origin.SecretID), the Open stage (stages.go injects origin.SecretID), restoreFleetOrigin (prev.SecretID, cleared with cert), PUT /origin without credential fields (linkedOriginSpec copies existing.SecretID), PATCH /origin/upstream (setOriginUpstream re-injects existing.SecretID; CheckOriginBranch's spec carries it). Tests: TestCarry_* in internal/corpus/repos/secret_link_carry_test.go and internal/edge/web/origin_secret_test.go.

Rule for new code: any new AttachOrigin/SwapStore caller that builds its OriginSpec from Origins.Get or GetRemote must set SecretID (and leave AuthToken empty when it is set). Misreading to avoid: the light credential path (ChangeOriginCredential / SetCredential) is NOT a rebuild; it is the one place that changes the link on purpose.

Backstop: a caller that forgets is no longer silent. applyAttach (Origins.SetAttached) and applySwap (Origins.Replace) refuse with control.ErrOriginLinked (409 "Linked to a stored secret") when the stored origin is linked and the spec carries no SecretID, checked in the write's own transaction, so a spec built from a read taken before a concurrent link (TestCarry_AttachRacingALinkIsRefused) never drops it. Only callers past a gate that may unlink set OriginSpec.ReplaceLink: the origin-session commit after the admin gate, followFleetHost and restoreFleetOrigin (local-only fleet routes).
