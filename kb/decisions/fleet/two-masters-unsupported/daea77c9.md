---
type: observation
domain: [fleet, decisions, consensus]
confidence: 0.95
sources: 1
entities: [isFleetMaster, holdsFleetRoot, fact.FleetMasterPath, claimFleetMaster, PublishUpstreamChange, ErrUpstreamMoved, fleetPeers]
motifs: [unsupported-by-ruling]
refs: ['kb://3ec012f5b4d2/kb/invariants/fleet/baked-ontology/master-refresh/d62746d2.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/master-marker/13dcd692.md', 'kb://3ec012f5b4d2/kb/decisions/fleet/master-switch/stand-down-first/9efa08ef.md', 'src://7b4887ce51d9/internal/repos/fleet_chokepoint.go@8ca72760a45bc4d54f3fa4d8e2d2358840e8a9d2:328ffd29bce2a7047545d249c48b173629c4f7d7', 'src://7b4887ce51d9/internal/repos/fleet_host.go@8ca72760a45bc4d54f3fa4d8e2d2358840e8a9d2:2e4d390a302d50377293a51f892adf1c4cc78e78', 'src://7b4887ce51d9/internal/repos/fleet_master.go@8ca72760a45bc4d54f3fa4d8e2d2358840e8a9d2:6d1491587efb0f59936d1ea3e9587d4baeeca3cd', 'https://github.com/knomit/knomit/pull/524', 'https://github.com/knomit/knomit/pull/531']
---
# DECISION (user rulings, PR #524 and #531): TWO fleet masters at once are NOT supported; since ruling 6 (2026-10-16) the master is the root holder that the master marker on MAIN names, so a second root holder is a member, and while main names no master root holders claim and git's non-fast-forward refusal decides (no election, no lease, no fencing)

RULING (user, PR #524): two masters at once are not supported, and knomit builds no detection, no lease and no fencing for a second one. RULING 6 (user, 2026-10-16, PR #531), verbatim: "If each puts their own key in their own branches, it means they're both candidates. Whoever the merge sets in main, that is the true master, and the second one will be demoted to a member."

WHAT 'MASTER' IS NOW (branch fix/fleet-switch-replay, verified at fc895fde; re-pin at the merge commit): Manager.isFleetMaster = holdsFleetRoot (<home>/fleet/root.key is a regular file AND the local root.crt equals artifacts/fleet/root.crt on main) AND the master marker on main (artifacts/fleet/master) names this instance. SUPERSEDED (true at PR #524, false now): 'master = root key plus root.crt on main' and 'copying the root key to a second machine makes a second master'. A second root holder that main does not name is a MEMBER: it merges, admits, publishes, enrolls, revokes and writes nothing master-only (kb/invariants/fleet/master-marker/13dcd692.md).

CLAIMS ARE NOT AN ELECTION. While main names NO master (after a stand-down, a forge decommission, or on a fleet created before the marker), every root holder is a candidate and claims the role at its next round: on a forge it publishes one commit naming itself straight onto main without force, so the first publish wins and the loser's push is refused as not a fast-forward (store.ErrUpstreamMoved); on the hub the claim lands on its own main. knomit still never chooses between instances: git's non-fast-forward rule does, and a root holder never claims while main names someone.

WHAT IS STILL UNSUPPORTED: two masters through the dead-master fallback. A master that is down when another root holder takes over (Host here with confirm_loss) cannot learn it was replaced; as the hub of its own stale main it keeps acting as master there until it points at the new host, where main names the other instance and it is demoted (its marker-only commits are dropped by the replay; a host-record rewrite it made meanwhile is refused for good). Nothing detects or fences that window.

MISREADINGS. (1) 'Holding the root key makes an instance master': no, main must name it. (2) 'Two masters are now impossible': no, the dead-master fallback window above remains and is unsupported. (3) 'The claim is an election knomit runs': no, it is a publish race git decides.

VERIFIED AT THE MERGE (PR #531, squash commit 8ca72760 on dev, 2026-10-16): isFleetMaster (fleet_chokepoint.go), fleetCandidate and claimFleetMaster (fleet_host.go: a forge claim is PublishUpstreamChange without force, a refused one logs 'the claim to the master role lost' on store.ErrUpstreamMoved) are as described; fleetMasterPause pauses only a cert origin, never a forge origin.
