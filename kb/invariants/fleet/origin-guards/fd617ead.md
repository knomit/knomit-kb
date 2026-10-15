---
kind: pragmatic
type: policy
domain: [fleet, web, repos, invariant]
confidence: 0.9
sources: 1
entities: [Manager.DetachRepoOrigin, guardFleetDetach, refuseNoHubListener, ErrFleetUpstreamFixed, RepoInstance.FleetMergeRefusal, member_keeps_origin, fleet_upstream_fixed, not_master, judge_refused, listener_required, addresses_required]
motifs: [gate-per-door]
refs: ['src://7b4887ce51d9/internal/repos/fleet_membership.go@fd991d4fa4295afefbaa354723824f23a86b4de0:fc4908b9950b336903ddafbe34c7086f4968a9f8', 'src://7b4887ce51d9/internal/web/handlers_origin_hal.go@fd991d4fa4295afefbaa354723824f23a86b4de0:18d1119b5885293f3908233e5bf03e6cf67c5074', 'src://7b4887ce51d9/internal/web/handlers_pushed_branches.go@fd991d4fa4295afefbaa354723824f23a86b4de0:5a005165699ead7705796f29c718344026d2ced4', 'kb://3ec012f5b4d2/kb/invariants/fleet/baked-ontology/read-only/dd11deec.md', 'kb://3ec012f5b4d2/kb/architecture/store/git-serve/pushed-branch-merge/f9ec4794.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/create/fleet-op-lock/8d73df73.md', 'https://github.com/knomit/knomit/pull/525']
---
# The fleet repository's origin routes carry the R3.3 guards: a MEMBER cannot detach its origin (`member_keeps_origin`), the master's detach (which makes its copy the hub) needs [tls].addr and external_addresses and runs under fleetOpMu with the master check, nobody can change the upstream (`fleet_upstream_fixed`), a member's manual merge is `not_master` and the master's re-judges (`judge_refused`)

THE GUARDS (PR #525, design R3.3, verified at merge commit fd991d4f). Each is one check in an existing handler; all are on the FLEET repository only (ri.IsFleet()), every other repository is unchanged.
1. DELETE /repos/{fleet}/origin goes through `Manager.DetachRepoOrigin`, which takes fleetOpMu and then runs `guardFleetDetach` and the detach as ONE step (a stand-down, which passes while the copy has an origin, could otherwise remove the root key between the master check and the detach and leave a hub with no master). A member: 409 `member_keeps_origin` ('a member's fleet copy keeps its origin; only the master hosts the fleet'; without an origin a member has no main and nothing to push to). The master: allowed only if `refuseNoHubListener` passes, i.e. [tls].addr is set (422 `listener_required`) and external_addresses is set (422 `addresses_required`); the texts end with 'run `knomit reload`'. The same function is the create's pre-flight. On success the master's copy is the hub (GET /fleet hosting `none`).
2. PATCH /repos/{fleet}/origin/upstream: 409 `fleet_upstream_fixed` for everyone ('the fleet repository's consensus branch is fixed at create').
3. POST /repos/{fleet}/branches/{b}/merge: a non-master is 409 `not_master`; the master re-judges the branch (tip minus main) and refuses with 409 `judge_refused` (kb/architecture/store/git-serve/pushed-branch-merge/f9ec4794.md).
NOT GUARDED: attach / PUT origin on the fleet repository (it is how a forge token is rotated and how the master switch repoints a hub).

TEST. TestFleetMembership_DetachGuard (member refused and keeps its origin; the master's copy becomes the hub), TestFleetMembership_DetachTakesTheFleetLock (the detach waits for fleetOpMu; it does NOT pin that the check and the detach share ONE hold: a lock taken after the guard stays green), TestFleetREST_MemberOriginAndMergeGuards, TestFleetREST_MasterMergeIsReJudged.

MISREADINGS. 'The fleet repository's origin cannot be touched': only detach and upstream are refused. 'The master may detach freely': without [tls].addr and external_addresses it is refused, since members would have nowhere to push.
