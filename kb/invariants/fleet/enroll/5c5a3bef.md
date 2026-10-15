---
kind: pragmatic
type: policy
domain: [fleet, pki, repos, invariant]
confidence: 0.92
sources: 1
entities: [refuseEnrolledID, Manager.EnrollFleetMember, FleetPKI.Cert, pki.IsRevoked, pki.IssueLeaf, pki.RecordIssuance, rekey_unsupported, enrolled]
motifs: [unreachable-after-overwrite]
refs: ['src://7b4887ce51d9/internal/repos/fleet_membership.go@fd991d4fa4295afefbaa354723824f23a86b4de0:fc4908b9950b336903ddafbe34c7086f4968a9f8', 'src://7b4887ce51d9/internal/store/fleet_read.go@fd991d4fa4295afefbaa354723824f23a86b4de0:81fe8ede2ed93d840b0e966562973b80013cfd79', 'src://7b4887ce51d9/internal/repos/fleet_membership_test.go@fd991d4fa4295afefbaa354723824f23a86b4de0:23a0c4f1438ca987158a74c76a0f6929dfccce4d', 'kb://3ec012f5b4d2/kb/decisions/fleet/membership/leave-means-revoke/97126da4.md', 'kb://3ec012f5b4d2/kb/architecture/pki/ledgers/as-bytes/dff1c1d8.md', 'https://github.com/knomit/knomit/pull/525']
---
# POST /api/v1/fleet/enroll refuses an agent id that already has a leaf on main: a different key is `rekey_unsupported`, the same key whose leaf is not on main's CRL is `enrolled` (revoke it first), and only the same key with its old leaf revoked is admitted (the rejoin, a fresh leaf with a new serial)

RULE (PR #525, review blocker B1, verified at merge commit fd991d4f; refuseEnrolledID in internal/repos/fleet_membership.go, run under fleetOpMu BEFORE any issuance). Enroll reads the leaf at certs/<agent_id>.pem on the fleet's MAIN (fleetAtMain). No leaf: issue. A leaf for a DIFFERENT key than the requested public_key: 409 `rekey_unsupported` (a lost or rotated key is a fresh start under a new agent id; a same-id rekey is not supported). The SAME key and the leaf's serial NOT on main's CRL: 409 `enrolled` ('revoke it first (POST /api/v1/fleet/revoke)'). The SAME key and the leaf's serial ON main's CRL: admitted, the fleet's design 'Rejoin': a fresh leaf with a new serial replaces certs/<id>.pem and the join of that bundle is the approval. The key check comes first, so a different key under a revoked leaf is still `rekey_unsupported`. Other refusals: `invalid_agent_id` and `invalid_key` (422), `own_id` (the master's own id, 409), `not_master`/`not_registered` (409).

WHY. Revoke finds a member's serial ONLY through certs/<id>.pem on main. Before the check, a second enroll of the same id silently replaced that file; the FIRST leaf was then never on the CRL and POST /fleet/revoke only ever reached the second, so a lost-key leaf stayed valid (chains to the root, passes mTLS, implicit read and push:own) for its whole lifetime.

ORDER OF WRITES. IssueLeaf (pure), then the leaf and its issued/<serial>.pem are committed on the master's agent branch, THEN pki.RecordIssuance appends the local issued.jsonl line (a failed append is a WARN), then publish (hub: fast-forward main; forge: push the agent branch, then main without force). The answer is 201 with published false and publish_error when main does not hold the leaf yet; a joiner is admitted only once it does.

PINNED BY TestFleetMembership_EnrollOverAnExistingID (asserts the codes, that certs/<id>.pem on main and the issued.jsonl line count are unchanged after every refusal, the rejoin's new serial, and that BOTH serials end up on main's CRL after the two revokes).

MISREADINGS. 'Enroll is idempotent for the same key': it is refused as `enrolled` until the leaf is revoked. 'Enroll refuses every id that was ever enrolled': a revoked leaf with the same key is the rejoin.
