---
kind: pragmatic
type: policy
domain: [fleet, pki, decision]
confidence: 0.9
sources: 1
entities: [leave duty, leaf_serial, CRL, master duty]
motifs: [stale-names-old-object]
refs: ['src://7b4887ce51d9/internal/repos/fleet_leave.go@a1b0ca5d0f92de4498fde3db8246037f2e33c55e:2107309293d3555989b20c198f3fbf00c2987686', 'src://7b4887ce51d9/internal/repos/fleet_leave_test.go@a1b0ca5d0f92de4498fde3db8246037f2e33c55e:6d2ea678a9a47877bcfa3b45327ba31b8d51e19f', 'kb://3ec012f5b4d2/kb/decisions/fleet/membership/leave-means-revoke/97126da4.md', 'https://github.com/knomit/knomit/pull/532']
---
# Decision (user ruling 2026-10-16): "leave must revoke, the cert is valid per fleet, not in general" - a left member's leaf goes on the fleet's CRL through the master's duty, keyed by the serial the left record names

THE RULING (user, 2026-10-16, verbatim): "leave must revoke, the cert is valid per fleet, not in general". It was given while PR #532 was built, after the earlier ruling R6 (kb/decisions/fleet/membership/leave-means-revoke/97126da4.md) had been left as an unbuilt master duty (P5). Reading that the code relies on: a leaf is valid as a member of THIS fleet, so leaving ends it on this fleet's CRL (the CRL the fleet's root signs); the ruling does not extend to anything outside that CRL.

OPTIONS. (1) A left record leaves the leaf valid, a manual POST /fleet/revoke the only way to end it: the state before #532, rejected by the ruling. (2) The member revokes its own leaf: impossible, only the root key signs a CRL. (3) The master revokes automatically from the `left` record on main: CHOSEN. (4) Within (3), which leaf the left record means. The first build revoked main's current leaf unless it had landed AFTER the left record (a landing-order rule); a reviewer reproduced it revoking the fresh leaf of a rejoin when an operator revoked by hand and re-enrolled before the master merged the stale left record (a forge's fetch window is 300 s by default). Alternatives offered: the record names the leaf's serial and the duty revokes only that serial (CHOSEN, option a, a new left-record field leaf_serial); skip when the left commit was authored before the leaf's NotBefore (rejected: the author controls the time); document the limitation (rejected).

CONSEQUENCES. A left record without a serial (written before the field existed, or by an instance with no certificate installed) is never revoked automatically: the operator revokes by hand, as before. Details, guards and tests: the invariant kb/invariants/fleet/leave/master-revokes-left (written with this fact). MISREADING: 'the ruling makes Leave itself revoke': Leave still only writes the record, pushes once and uninstalls; the master revokes on its next round once the record is on main.
