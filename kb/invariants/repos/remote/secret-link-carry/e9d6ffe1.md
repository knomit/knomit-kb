---
kind: pragmatic
type: policy
domain: [repos, remote, secrets]
confidence: 0.85
sources: 1
entities: [OriginSpec.SecretID, applyAttach, applySwap, applyDetach, restoreFleetOrigin, setOriginUpstream, linkedOriginSpec, remotesync.Origin.SecretID, DetachOriginUnlinking, Origins.DeleteAttached]
motifs: [full-replace-drops-fields]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/control-db/secret-link/845c96d6.md', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/secret-link/f7d454a9.md', 'src://7b4887ce51d9/internal/corpus/repos/events.go@b421c6cb73226b64efba5f7f3a9541016292fc4a:b32e7d1821965dd26cebaa7820e1cd82c58b8eac', 'src://7b4887ce51d9/internal/corpus/repos/secret_link_carry_test.go@b421c6cb73226b64efba5f7f3a9541016292fc4a:154ac08036cb42af02c32c647ef857f2a2dbebd2', 'src://7b4887ce51d9/internal/corpus/repos/control/origins.go@b421c6cb73226b64efba5f7f3a9541016292fc4a:0364f281b666fb3c29ae2e6e061946634dbfe31d', 'src://7b4887ce51d9/internal/edge/web/api/manage/handlers_origin_hal.go@b421c6cb73226b64efba5f7f3a9541016292fc4a:48f3614cdd591aa19283c58021d6f6d8c87b4902']
---
# OriginSpec.SecretID IS the link on attach, swap, create and fleet restore: a caller that rebuilds a spec from the stored origin must copy SecretID, or the attach silently unlinks the repo

applyAttach and applySwap (internal/corpus/repos/events.go) write control.Origin and the injected remotesync.Origin from the OriginSpec they are given, including SecretID; Origins.Set is a full replace, so a spec with an empty SecretID DELETES the repo's link. The paths that rebuild a spec from stored state carry it: create (lifecycle.go persists cs.Origin.SecretID), the Open stage (stages.go injects origin.SecretID), restoreFleetOrigin (prev.SecretID, cleared with cert), PUT /origin without credential fields (linkedOriginSpec copies existing.SecretID), PATCH /origin/upstream (setOriginUpstream re-injects existing.SecretID; CheckOriginBranch's spec carries it). Tests: TestCarry_* in internal/corpus/repos/secret_link_carry_test.go and internal/edge/web/origin_secret_test.go.

Rule for new code: any new AttachOrigin/SwapStore caller that builds its OriginSpec from Origins.Get or GetRemote must set SecretID (and leave AuthToken empty when it is set). Misreading to avoid: the light credential path (ChangeOriginCredential / SetCredential) is NOT a rebuild; it is the one place that changes the link on purpose.

Backstop: a caller that forgets is no longer silent. applyAttach (Origins.SetAttached) and applySwap (Origins.Replace) refuse with control.ErrOriginLinked (409 "Linked to a stored secret") when the stored origin is linked and the spec carries no SecretID, checked in the write's own transaction, so a spec built from a read taken before a concurrent link (TestCarry_AttachRacingALinkIsRefused) never drops it. Only callers past a gate that may unlink set OriginSpec.ReplaceLink: the origin-session commit after the admin gate, followFleetHost and restoreFleetOrigin (local-only fleet routes).

The detach path has the same backstop. DetachOrigin() runs Origins.DeleteAttached(uid, false), which refuses with control.ErrOriginLinked in the delete's own transaction when the repo is linked (TestCarry_DetachRacingALinkIsRefused). Only DetachOriginUnlinking() (replaceLink=true) drops a link: DELETE /origin after the admin gate on a linked repo, the fleet host takeover (hostFleetHere) and restoreFleetOrigin's detach back to the hub (local-only fleet routes).
