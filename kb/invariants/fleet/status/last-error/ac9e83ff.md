---
kind: pragmatic
type: policy
domain: [fleet, web, ui, repos]
confidence: 0.9
sources: 1
entities: [FleetStatus.LastErrorCommitRefused, last_error_commit_refused, store.CommitRefusedByJudge, judgeIncoming, FleetSettings.tsx, fleet-commit-refused-notice, TestFleetStatus_LastErrorCommitRefusedIsTheJudgesRefusal]
motifs: [derive-from-recorded-text, false-retry-promise]
refs: ['src://7b4887ce51d9/internal/repos/fleet.go@b4874ee221c5e7c5698301e3be2ee2f343eb4391:435992b9d0f42454e13be3ea05b8fd4d584533a1', 'src://7b4887ce51d9/internal/store/httphandler_receivepack.go@b4874ee221c5e7c5698301e3be2ee2f343eb4391:35aeaa16d7dd895044a484c6311334516e34ac2b', 'src://7b4887ce51d9/web/src/FleetSettings.tsx@b4874ee221c5e7c5698301e3be2ee2f343eb4391:471562ee938ec5ae03e8749a142ecd3d7bc28c18', 'src://7b4887ce51d9/internal/repos/fleet_create_test.go@b4874ee221c5e7c5698301e3be2ee2f343eb4391:87e47f01ecc2181452b5553c8e94a6599b7f49d6', 'kb://3ec012f5b4d2/kb/invariants/store/remote-reconcile/replay-stop/5482c3ab.md', 'kb://3ec012f5b4d2/kb/invariants/fleet/root-move/b41b75ed.md']
---
# GET /fleet last_error_commit_refused is true when the recorded last_error carries store.CommitRefusedByJudge ("refused by verify_signatures=enforce", the host's receive-pack judge refusing a commit of this copy's push); Fleet Settings then shows fleet-commit-refused-notice and never "Retrying on the next sync."

RULE (branch fix/fleet-switch-replay, verified at b4874ee2; re-pin at the merge commit). FleetStatus.LastErrorCommitRefused = strings.Contains(row.LastError, store.CommitRefusedByJudge). The constant is the exact phrase judgeIncoming writes into its refusal ("knomit: commit <hash> refused by verify_signatures=enforce: <rule>: <reason>"), so the producer and the reader share one string: rewording the refusal without the constant breaks the flag. Like last_error_refused (TLS handshake) and last_error_fleet_moved, it is read from the recorded error only; nothing new is stored.

UI: in the Last sync failed box, last_error_refused wins (certificate refused), then last_error_commit_refused shows "The fleet's host refuses a commit on this instance's branch (the rule is named above). Every sync sends the same commit, so it is refused again until the fleet's main changes; ask the fleet's master." Only a failure with neither flag says "Retrying on the next sync." The raw last_error stays visible because it names the commit, the rule and the reason.

WHY: every sync re-sends the same branch, so a judge refusal repeats until main changes (a leaf or revocation lands, or the commit leaves the branch); a retry promise there is false. #528 N1 showed 'Retrying' forever after a master switch.

MISREADING: 'the flag means the refusal is permanent': no, a join-race no-cert refusal clears once the master publishes the leaf; the text says it is refused again UNTIL main changes. It is also not set on a forge fleet, where members' branches are judged by the master's fetch (PeerRefusal on the master), not refused to the member's push.

PINNED BY: repos TestFleetStatus_LastErrorCommitRefusedIsTheJudgesRefusal; vitest FleetSettings 'a commit the host refuses is not "Retrying"'. Sabotage of each went red.

UPDATE (branch fix/fleet-switch-replay, verified at a78d1b22; re-pin at the merge commit; PR #531 review S1/S4). The UI text is now: 'Retrying will not help: every sync sends the same commit. When the rule is host-record or an artifact rule, the refusal is permanent: to clear it, leave the fleet here and ask the fleet's master to enroll this instance again; what this instance wrote that the host does not hold is lost. Otherwise ask the fleet's master.' The earlier 'until the fleet's main changes' was false for host-record (judge.hostRecord diffs the commit against its own parents and never reads main). The flag is now driven end to end by a real refusal: cmd TestFleetMasterSwitch_FallbackThenStandDownIsRefusedForGood (an old master that came back after a confirmed takeover stands down before pointing at the new host; its master-marker deletion is refused host-record on every round). Sibling flag: last_error_stood_down (FleetPushRefusalStoodDown), a push refused by a hub whose master stood down; the page says to point at the new host. MISREADING kept: not every judge refusal is permanent (record rules depend on main).
