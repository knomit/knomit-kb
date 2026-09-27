---
kind: pragmatic
type: policy
domain: [store, verify, repos, clone]
confidence: 0.9
sources: 1
entities: [checkOwnLineage, ForeignLineageError, Service.SetOwnKeys, Service.FleetKeysOf, Manager.ownFleetKeys, InitFromRemote, verify_accepted]
motifs: [decide-before-ref-moves, input-attacker-cannot-choose]
refs: ['src://7b4887ce51d9/internal/store/verify_e4.go@d4205ad457f087b8a714c17f4745a5f7bea3675c:a5ab11d06981e40523657d23951fcde83fe9073a', 'src://7b4887ce51d9/internal/store/fleet_read.go@d4205ad457f087b8a714c17f4745a5f7bea3675c:407da5166db3fdb171b24c713812bb9810e0bec0']
---
# F09 first contact sets refs at origin's tips (no fold, since PR 5); E4 still adopts origin's copy of the own agent branch only if every commit beyond origin's main is signed by an own key (current key, plus every key the own fleet member record has held) or accepted, and otherwise FAILS the create, never starting a fresh lineage

Since F09 PR 5 (user ruling: verify once, at acceptance), first contact (InitFromRemote, InitSubscription, CloneFrom) no longer folds history: local refs are set at origin's tips, which are trusted as fetched.

E4 (checkOwnLineage) stays, because it IS acceptance: it is the point where this instance adopts origin's copy of its OWN agent branch, whose commits it will push again under its own identity. Every commit reachable from that branch and not from origin's main must be signed by an own key or be on this instance's accept list (repo-DB verify_accepted until #330 moves it). Own keys = the store's commit signer, plus (Service.SetOwnKeys, wired by the Manager next to SetSigner) every key this instance's fleet member record has held (Service.FleetKeysOf over the record's versions), so a key rotation does not make earlier own commits foreign. Standalone: the current key only. Otherwise the create FAILS with ForeignLineageError naming each refused commit; it never starts a fresh lineage (that would overwrite the remote branch on the first push and silently discard its commits).

Tests: verify_e4_test.go (own key adopted; unsigned, another key, no signer refused; pre-signing commits refused then adopted once accepted; an old record key adopted only with the record).
