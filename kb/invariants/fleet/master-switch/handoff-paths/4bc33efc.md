---
kind: pragmatic
type: policy
domain: [fleet, master-switch, invariant, pki]
confidence: 0.85
sources: 1
entities: [Manager.flushBeforeSwitch, Manager.StandDown, Manager.FleetRootFiles, Manager.DecommissionFleet, Manager.LeaveFleet, hostFleetHere, refuseOldMasterActive, claimFleetMaster, fleetCandidate, fleetMasterPause, FollowFleetHost, confirm_loss]
motifs: [flush-before-handover, enumerated-exits]
refs: ['kb://3ec012f5b4d2/kb/invariants/fleet/master-switch/flush-first/510ee1d1.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/master-marker/13dcd692.md', 'kb://3ec012f5b4d2/kb/decisions/fleet/master-switch/stand-down-first/9efa08ef.md', 'src://7b4887ce51d9/internal/repos/fleet_membership.go@b4677384bbdc9b3d2084d4a53ffb179d26576604:6c1a802ce18e942c97b119c9b77047a8bed63856', 'src://7b4887ce51d9/internal/repos/fleet_master.go@b4677384bbdc9b3d2084d4a53ffb179d26576604:d7f7ec4501fd224b699235c54ae804bbbc7c4bd5', 'src://7b4887ce51d9/internal/repos/fleet_host.go@b4677384bbdc9b3d2084d4a53ffb179d26576604:a70d2e0010f6b6615a0c2348643f9887b86047c4', 'src://7b4887ce51d9/internal/repos/fleet_switch_flush_test.go@b4677384bbdc9b3d2084d4a53ffb179d26576604:f71f8bf3cabe77f14bfce9e0775d872da25099b6', 'https://github.com/knomit/knomit/pull/537']
---
# No path by which the acting fleet master changes hands unpublished revokes, renewals or CRLs without a flush or the operator's explicit confirm_loss: 13 paths were walked at PR #537 (561f2288), and a new path, or a change to one, must be walked again

RULE. Every way an instance stops being the acting master, or another becomes one, must either flush first (flushBeforeSwitch: kb/invariants/fleet/master-switch/flush-first/510ee1d1.md) or need the operator's explicit confirm_loss (Host here, the dead-master fallback). The final re-review of PR #537 (read of fleet_membership.go, fleet_host.go, fleet_master.go, fleet_create.go at 561f2288; re-read at the merge b4677384) walked these 13:
1. Stand-down, DELETE /fleet/root: flushes, under fleetOpMu and the merger's runMu, then deletes the marker, then the root.
2. Root export, GET /fleet/root: flushes under fleetOpMu. Not itself a hand-off; the role changes only at 1, 3 or 8.
3. FORGE decommission (origin set): flushes under fleetOpMu only, 409 keeps repository, root key and marker.
4. HUB decommission (no origin): no flush, because a hub master's writes reach its main at once (publishFleetMainNow) and it does not delete the marker, so main still names it; a successor needs path 8 with confirm_loss.
5. Leave by a root holder: refused is_master whenever the root is held (holdsFleetRoot).
6. Candidate claim race (claimFleetMaster, fleetCandidate): needs main to name NO master, and the marker is deleted only by 1 or 3, both flushed. Exception: a pre-marker legacy fleet, whose only root holder is its own master.
7. Demotion after a lost claim: a candidate is not the acting master before its claim lands, so it has no master writes to lose.
8. Host here (hostFleetHere): without confirm_loss, refuseOldMasterActive needs main to name no master or this instance; with confirm_loss it is the dead-master fallback (documented in openapi as "may lose those commits"). Host here by the current forge master on itself is not a hand-off.
9. Marker deletion by others: a member's branch cannot change artifacts/fleet/master (host-record rule); the writers are 1, 3 and 8.
10. Pause (fleetMasterPause, master_paused / not_hosting, reached by FollowFleetHost with a cert origin): flushBeforeSwitch skips a non-acting master; a hub master pointed elsewhere creates no new master (main still names it), a successor still needs 8 with confirm_loss. Judged by reading, not by a test.
11. Uninstall or root mismatch (pki.Uninstall in leave, decommission, join undo): leave is refused for a root holder (5), decommission is 3 or 4, the join undo has no master.
12. Hub-to-forge or forge-to-forge move (fleet_move.go): same master, new hosting, gated by isFleetMaster.
13. Drop commit (/fleet/drop-commit): the operator removes a refused own commit on purpose.
The only paths that can lose unpublished master writes are 4 then 8 and 10 then 8, both through confirm_loss.

HOW TO USE. Adding a way to hand on or take the master role, or changing any of the 13, means adding it to this table with its flush or its confirm_loss, and a test in internal/repos/fleet_switch_flush_test.go (TestFleetSwitch_GivingUpTheRoleAfterAnEarlyExport covers 1, 2 and 3; ExportAndStandDownWaitForTheFlush 1 and 2; ADecommissionWaitsForTheFlush 3).

MISREADING. "Every master change flushes": the hub decommission (4), the pause (10) and Host here (8) do not; each is safe for the reason in its row, not by the flush. Rows 10 and the hub half of 4 are read, not tested; the hub's not-held 409 wording has no test either (no fixture makes the hub's local publish fail without the judge).
