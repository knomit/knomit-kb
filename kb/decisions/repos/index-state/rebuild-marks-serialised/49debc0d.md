---
type: observation
domain: [repos, web, index, concurrency, api]
confidence: 0.95
sources: 1
entities: [handleStartRebuild, Rebuild, Reply.Absorbed, indexSpec, publishRebuildTask, TaskHub]
motifs: [single-writer-by-refusal, visible-failure-over-silent]
refs: ['kb://3ec012f5b4d2/kb/decisions/repos/lifecycle/rebuild-replaces-or-absorbs/36f50eda.md', 'kb://3ec012f5b4d2/kb/invariants/repos/index-state/publish-chokepoint/4110015c.md', 'kb://3ec012f5b4d2/kb/decisions/repos/create-job/done-means-indexed/0c1b3368.md', 'src://7b4887ce51d9/internal/web/handlers_jobs.go@12a3bb8901b6f78ca19fbc37f99beeb9a243f667:e78ebec59f4fc28dc272701a1cf6a65eb94d1530', 'src://7b4887ce51d9/internal/repos/machine.go@12a3bb8901b6f78ca19fbc37f99beeb9a243f667:6abc9980c7e763ad4c0a5dbf95737ea48067d273', 'https://github.com/knomit/knomit/pull/221', 'https://github.com/knomit/knomit/issues/411', 'https://github.com/knomit/knomit/pull/414']
---
# A rebuild REPLACES a running index job (heal or other-branch rebuild) and ABSORBS an identical running one (same branch, same job id); the 409 of PR #221 is retired because there is exactly one Index life and no status cell to contend for

Rewritten for issue #411; the current decision is kb/decisions/repos/lifecycle/rebuild-replaces-or-absorbs/36f50eda.

Now: POST index-rebuilds sends Rebuild to the repo's lifecycle machine. A running full rebuild of the same branch absorbs the request (200, Reply{JobID, Absorbed: true}); anything else running on Index (a heal, another branch's rebuild) is cancelled, drained and replaced by a new job (201). There is no 409.

WHY THE 409 WENT. PR #221 chose serialisation-by-refusal because the index state was a CELL with several writers (heal marks, the rebuild's MarkIndexRebuildStart/Done bracket) and a claim/release count would pin 'indexing' forever on a leaked release. Under the machine there is one Index life and the state is derived from it (kb/invariants/repos/index-state/publish-chokepoint), so there is no cell and no second writer: Rebuild is just an event targeting the Index stage.

ACCEPTED COSTS kept from #221: no force flag; an error from a heal and an error from a rebuild share index_state 'error' (index_reason now carries the job's error text).

HISTORY: before #221 a manual rebuild touched no index state, so the chip read 'ready' throughout a rebuild; #221 bracketed it and 409'd while indexing; #411 replaced both with the machine.
