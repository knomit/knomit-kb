---
type: observation
domain: [fleet, repos, web]
confidence: 0.88
sources: 1
entities: [last_refusal, fleetPeers.refusals, PeerRefusal, join_refused, record_state]
motifs: [state-invisible-to-reader]
refs: ['src://7b4887ce51d9/internal/corpus/repos/fleet.go@0610c20083fcef9e84f42ab0311a14708bcb6193:756540a5025bc7b3199f2964602124a03c7cba49', 'src://7b4887ce51d9/internal/corpus/repos/fleet_master.go@0610c20083fcef9e84f42ab0311a14708bcb6193:d8c79f63e13474a2c844ed395c6e7c4df1a2071d', 'kb://3ec012f5b4d2/kb/architecture/fleet/master-merges/91e3f6da.md', 'https://github.com/knomit/knomit/pull/532']
---
# KNOWN GAP: a master's refusal of a joiner's branch (last_refusal) cannot reach the joiner, so a joiner whose record the master refuses after the push stays `pending` with no explanation

Observed in PR #532 (verified at a1b0ca5d). Refusals live only in the master's memory (fleetPeers.refusals and its ring of 200); nothing about them is written to the fleet repository, and a joiner syncs only main and its own branch. So: on a hub, a refusal at PUSH time (receive-pack) already reaches the joiner as join_refused or last_error; a LATER refusal (the merger's re-judge against a moved main, or any forge fetch-judge refusal) is invisible to the joiner, which keeps reading record_state `pending`. Only the master sees it (GET /fleet/members last_refusal, GET /fleet/activity, its log). A real instance: a stale left record that revoked the rejoin's fresh leaf (fixed by leaf_serial) left the rejoin pending forever, because rule B refuses a revoked leaf and nothing told the joiner.

CONSEQUENCE: never write joiner copy that promises an outcome or an error for 'pending'; the pending text says only that the record is on this instance's branch and the master has not accepted it. A channel from the master to the joiner (a record or artifact on main) would be new work. MISREADING: 'pending means the master has not looked yet': it may have refused it.
