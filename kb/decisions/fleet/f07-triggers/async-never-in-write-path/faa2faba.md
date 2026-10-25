---
type: observation
domain: [fleet, triggers, decisions, performance]
confidence: 0.95
sources: 1
entities: [triggerDispatcher, ri.onCommit, triggerKick, TestDispatch_WriteLatencyIndependentOfTriggers, TestDispatch_TriggerKickNeverBlocksWriter, internal/corpus/repos/triggers.go, internal/repos/builder.go]
motifs: [coalescing-pending-flag, off-the-write-path]
refs: ['src://7b4887ce51d9/internal/corpus/repos/trigger/triggers.go@ee850fd6cef7c936fc81af311371e34a6971350a:8b055e220ddc6a6925559e47d353fbfde05e5365', 'src://7b4887ce51d9/internal/repos/builder.go@0ef7edeecbf5863c6c9091f39a65be1aceb96d66:a8723f5bcf4833cbadf70371026d9035a7e266f0', 'https://github.com/knomit/knomit/pull/335', 'kb://3ec012f5b4d2/kb/architecture/repos/triggers/dispatcher/a01f40fc.md']
---
# Triggers are ASYNC and never in the write path: the observer callback only records 'branch advanced' (one non-blocking kick on a 1-slot channel), all diffing/matching/if/actions run on the dispatcher's own goroutine, matching is single-digit ms per advance, and write latency must be indistinguishable with 0 and 50 triggers; rejected any trigger work under the branch lock and any queue that can block the writer

OPTIONS. (1) Dispatch synchronously inside the commit observer (simplest; every write waits for every trigger's `if`). (2) A queue of advances fed from the observer (bounded queues block the writer when full; unbounded ones grow). (3) A coalescing pending flag — one 1-slot channel, non-blocking send — with a level-triggered dispatcher that diffs watermark → current head on its own goroutine with its own ctx, holding the store only for the diff/read and the two bookkeeping transactions.

RATIONALE (the user, verbatim, 2026-09-27 ~10:45): "for the trigger mechanism, I want to make it async - not in the write blocking path. I also want to make the trigger as efficient as possible, to the point that the matching machinery should run in ms. Last thing I want is knomit to become super slow and unresponsive because we have triggers defined." A pending flag is enough because at-least-once via the watermark is already the contract, so coalesced advances lose nothing; the reviewer measured one root tree diff at ~12 ms p50 on knomit-kb, and in-memory glob matching adds microseconds, so N × ChangesUnder per trigger (105 ms at 8) was rejected too.

CHOICE: option (3), binding on every F07 PR: the write path does NO trigger work; globs compiled once per ontology blob; back-pressure by coalescing, the fire log records the coalesced range; a write-latency test with 0 vs 50 triggers and a per-advance matching benchmark in the PR body. Built in 1a (compile cache, zero-allocation matcher) and 1b (the kick in ri.onCommit, phases A/B/C, TestDispatch_WriteLatencyIndependentOfTriggers, TestDispatch_TriggerKickNeverBlocksWriter).

NON-SCOPE: `if` conditions are user JavaScript and can make the DISPATCHER fall behind (100 ms cap per evaluation); writes and reads are unaffected, and the slow-trigger detector names the culprit.

MOVED (PR #632, 14c, merged as ee850fd6): the trigger code cited here now lives in internal/corpus/repos/trigger (triggers.go); file and identifier names in the text above are the pre-move ones (triggerDispatcher is trigger.Dispatcher, ri.triggerKick is ri.triggers.Kick(), a nil-safe non-blocking send on the 1-slot channel). The decision is unchanged: the write path only records 'branch advanced' and all trigger work runs in the dispatcher's own goroutine (kb/architecture/repos/triggers/dispatcher/a01f40fc.md). The refs are re-pinned to ee850fd6.
