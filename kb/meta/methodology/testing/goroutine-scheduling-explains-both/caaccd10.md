---
type: methodology
domain: [testing, concurrency, methodology]
confidence: 0.85
sources: 1
entities: [indexHealGate, passedThrough, waitArrived, Manager.openOne, markIndexing]
motifs: [shared-cause-two-rarities]
refs: ['src://7b4887ce51d9/internal/repos/create_job_test.go@108b7ffd8d2ca763b831447b4171c299ef8d5ba3:9aef891649516016729747b10c41d0c3e9796f9a', 'https://github.com/knomit/knomit/pull/228', 'kb://3ec012f5b4d2/kb/meta/methodology/guard-setup-outruns-hazard/2b230458.md']
---
# When a flake and its detector are both rare on the same machine, suspect ONE mechanism rather than two coincidences

Two puzzles on the same fixture looked independent and were not:

1. A CI flake ("the create never reported an index phase") that failed twice on ubuntu but would not reproduce locally in 30 runs under load.
2. A sabotage that the same fixture detected only 5 times in 30, while its sibling fixture detected it 30/30.

One mechanism explains both: WHETHER THE HEAL GOROUTINE HAS BEEN SCHEDULED YET. `openOne` calls `markIndexing()` SYNCHRONOUSLY and only then launches the goroutine, so a caller can observe `indexing` before the goroutine has run a single instruction. On a box that schedules it promptly the original race is rarely lost — and a fixture reading "has the heal passed the gate?" rarely sees the mutation, because false there ALSO means "not started yet".

THE GENERAL SHAPE: a state flag sampled at one instant conflates "not yet arrived" with "arrived and waiting". A fixture built on that flag inherits the scheduling race it was written to eliminate, and its detection rate will track the very flake it is meant to catch. When both numbers are low on the same machine, that correlation is the evidence — look for the shared cause rather than filing two mysteries.
