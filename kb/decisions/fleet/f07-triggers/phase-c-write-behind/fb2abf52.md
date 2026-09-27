---
type: observation
domain: [fleet, triggers, decisions, sqlite, performance]
confidence: 0.9
sources: 1
entities: [pendingFlush, settle, flushOnStop, triggerFlushGrace, RecordTriggerRuns, AdvanceTriggerWatermarks, internal/repos/triggers.go, internal/store/triggers.go]
motifs: [write-behind-buffer, busy-handler-backoff]
refs: ['src://7b4887ce51d9/internal/repos/triggers.go@0ef7edeecbf5863c6c9091f39a65be1aceb96d66:8411c6b3bef2f8b7c5768fb8960c890e281d7b3b', 'src://7b4887ce51d9/internal/store/triggers.go@0ef7edeecbf5863c6c9091f39a65be1aceb96d66:62bda301a444689511798e53d94622221b5d37a2', 'https://github.com/knomit/knomit/pull/335', 'kb://3ec012f5b4d2/kb/decisions/repos/triggers/fire-log-write-behind/db7bd869.md', 'kb://3ec012f5b4d2/kb/invariants/store/sqlite/write-serialization/9caa7895.md']
---
# Phase C of a dispatcher run is WRITE-BEHIND (flushed after 20 ms of writer silence, at a size bound, or at shutdown) rather than two transactions per run: a worker deviation, measured (per-run flushing cost the writer 5–7 ms per write through SQLite's busy-handler backoff), accepted by the master with the user informed and not vetoing

OPTIONS. (1) The proposal's letter: after every run, tx1 (fire rows + run row) then tx2 (watermarks + prune) immediately. (2) Buffer each run's phase C and flush when no kick has arrived for a short grace (20 ms), when the buffer reaches 10,000 rows or 256 runs, or at shutdown; under a burst several runs land in ONE tx1 and ONE tx2 after the burst.

RATIONALE (measured on the worker's laptop, alternating A/B blocks): the repo DB runs every transaction as BEGIN IMMEDIATE and a fact write takes that lock several times per commit; a dispatcher transaction landing mid-write costs the writer a busy-handler backoff step. Per-run flushing: 10.2 → 17.1 ms median write latency with 50 matching triggers (the kick alone: no change; phase C skipped: 12.4 ms). Write-behind: 5.4 → 6.5 ms. The reviewer reproduced it (flush-every-run 6.5 → 10.8 ms) and found the deviation rule-preserving: emits still happen per run immediately, one run row per run, tx1 still precedes tx2, phase A overlays the buffered bookmarks so an advance is never diffed twice, a crash before the flush re-fires the range (the at-least-once shape the proposal states for a crash before tx1), a failed or panicking flush drops the buffer, a buffer from an older store generation is dropped on SwapStore.

CHOICE: option (2), in #335 (`pendingFlush`, `settle`, `flush`, `flushOnStop`, `triggerFlushGrace`). Accepted by the master as within the user's asynchrony requirement ("Last thing I want is knomit to become super slow and unresponsive because we have triggers defined"); told to the user, not vetoed.

NON-SCOPE: only the TABLES lag a burst by ~20 ms plus the flush; nothing about emit timing, at-least-once or the run-row shape changed. A follow-up for PR 2: the lag under a sustained write stream.
