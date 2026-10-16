---
type: observation
domain: [fleet, decisions]
confidence: 0.95
sources: 1
entities: [StandDown, HostFleetHere, fact.FleetMasterPath, confirm_loss, old_master_active]
motifs: [unsupported-by-ruling]
refs: ['kb://3ec012f5b4d2/kb/decisions/fleet/two-masters-unsupported/daea77c9.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/root-move/b41b75ed.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/master-marker/13dcd692.md', 'https://github.com/knomit/knomit/pull/531']
---
# DECISION (user rulings 2026-10-16): the master switch is 'the old master stands down FIRST, then the new master takes the role'; no handover protocol, no flush, no contact between the two; the only safety check is the new master's own synced main (the master marker), with an explicit loss confirm

WHAT WAS DECIDED (PR #531 rework). After a review found the first fix incomplete, the user ruled in four steps the same day: (1) first 'flush, freeze, then hand over' (a handover state on the old master, the new master verifying its final main); (2) OVERRIDE: 'It could be as simple as A stops accepting changes and stops acting as master (no more merges). Then, C sets the role master and starts merging. There is no reason for A to talk to C or the other way around.'; (3) CLARIFICATION: a master that stood down must be able to become master again by the same steps (A -> C -> A); 'must not act as master once it stood down' means it never RESUMES by itself; (4) FINAL: only a marker naming the acting master (artifacts/fleet/master), replacing a stood-down line in the host record; explicitly NOT a root-signed record, NOT 'master: none', NO isFleetMaster change, NO new judge rule beyond adding the path to the host-record rule.

OPTIONS REJECTED: the handover/flush protocol (state, cancel, drain of in-flight pushes, a git capability advertising the final main) was partly built and removed before any commit: more machinery than the problem needs. An authoritative master record (isFleetMaster requires it; the judge refuses master-only writes from non-named agents; root-signed) was analysed and stopped: commit signatures are instance keys, so 'signed by the root holder' needs a root-signed artifact; 'delete the record' collides with legacy fleets; isFleetMaster reading main blocks the new master's own Host here.

CONSEQUENCES: see kb/invariants/fleet/master-marker/13dcd692.md and kb/invariants/fleet/root-move/b41b75ed.md. KNOWN RESIDUE: a master replaced while it was down (confirmed takeover) cannot learn it was replaced until it points at the new host; if it stands down BEFORE pointing, its marker deletion is a host-record-rule change the new host refuses for good (last_error_commit_refused; TestFleetMasterSwitch_FallbackThenStandDownIsRefusedForGood). On a forge-hosted fleet nothing checks the marker: importing the root makes the new master work at once, and its round claims the marker only once the old one is gone.

MISREADING: 'two masters at once is now detected': no, it is still unsupported (kb/decisions/fleet/two-masters-unsupported/daea77c9.md); the marker only lets the new master see, on its own main, whether the old one stood down.
