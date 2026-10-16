---
kind: pragmatic
type: policy
domain: [fleet, web, decision]
confidence: 0.92
sources: 1
entities: [recordStateOf, record_state, RecordPending, record_pending_update, state registered]
motifs: [candidate-versus-accepted]
refs: ['src://7b4887ce51d9/internal/repos/fleet.go@a1b0ca5d0f92de4498fde3db8246037f2e33c55e:4283661b454197e33f91f197f44783e311750a0a', 'src://7b4887ce51d9/internal/repos/fleet_leave_test.go@a1b0ca5d0f92de4498fde3db8246037f2e33c55e:6d2ea678a9a47877bcfa3b45327ba31b8d51e19f', 'kb://3ec012f5b4d2/kb/decisions/fleet/derived-revoked/504782ff.md', 'https://github.com/knomit/knomit/pull/532']
---
# Decision (user ruling 2026-10-16): "our candidate record in our branch means pending. our record in main means accepted." - a joiner's record_state is pending until main holds its record in the same state, and the comparison is of STATE, not of record version

THE RULING (user, 2026-10-16, verbatim): "our candidate record in our branch means pending. our record in main means accepted." Built in PR #532 (a1b0ca5d on dev) as the one rule recordStateOf; its table and readers: the invariant kb/invariants/fleet/status/record-state-rule (written with this fact).

DECISIONS TAKEN UNDER IT. (1) STATE, not version: an accepted member whose newer record version differs only in advertised fields (an address refresh) stays `active` and carries record_pending_update; comparing blobs would flip every address refresh back to 'pending'. The author proposed it, the master decided it, the reviewer pinned it (probe: active with record_pending_update true; a blob-comparing mutation went red on the pre-existing TestFleetRecord_BootReconcile too). (2) `state: registered` in the join answer stays the LOCAL state machine ('has the fleet repository'), not acceptance; acceptance is record_state. (3) A joiner whose record is nowhere reads no record_state (omitted), where dev before the change read pending. (4) The same rule fixes a dev bug: a rejoin read `left`, because main still held the old left record while the joiner's new active record was only on its branch; it now reads pending.

MISREADINGS. 'pending means the master is processing it': nothing tells the joiner (see the gotcha kb/gotchas/fleet/join/refusal-invisible-to-joiner). 'record_state active means this instance has the newest record accepted': only that main holds an active record; record_pending_update says a newer version is not yet accepted.
