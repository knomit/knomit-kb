---
type: observation
domain: [repos, web, index, concurrency, api]
confidence: 0.95
sources: 1
entities: [handleStartRebuild, Rebuild, Reply.Absorbed, indexSpec, publishRebuildTask, TaskHub]
motifs: [single-writer-by-refusal, visible-failure-over-silent]
refs: ['kb://3ec012f5b4d2/kb/decisions/repos/lifecycle/rebuild-replaces-or-absorbs/36f50eda.md', 'kb://3ec012f5b4d2/kb/invariants/repos/index-state/publish-chokepoint/4110015c.md', 'kb://3ec012f5b4d2/kb/decisions/repos/create-job/done-means-indexed/0c1b3368.md', 'src://7b4887ce51d9/internal/edge/web/api/jobs/handlers.go@97ae7c195e0f4ac76868f9059728e4dd2b950957:4823488153149246a3cb531458000092bfe6a2c1', 'src://7b4887ce51d9/internal/corpus/repos/machine.go@0610c20083fcef9e84f42ab0311a14708bcb6193:6abc9980c7e763ad4c0a5dbf95737ea48067d273', 'https://github.com/knomit/knomit/pull/221', 'https://github.com/knomit/knomit/issues/411', 'https://github.com/knomit/knomit/pull/414', 'https://github.com/knomit/knomit/pull/627']
---
# A rebuild REPLACES a running index job (heal or other-branch rebuild) and ABSORBS an identical running one (same branch, same job id); the 409 of PR #221 is retired because there is exactly one Index life and no status cell to contend for

Rewritten for issue #411; the current decision is kb/decisions/repos/lifecycle/rebuild-replaces-or-absorbs/36f50eda.

Now: POST index-rebuilds sends Rebuild to the repo's lifecycle machine. A running full rebuild of the same branch absorbs the request (200, Reply{JobID, Absorbed: true}); anything else running on Index (a heal, another branch's rebuild) is cancelled, drained and replaced by a new job (201). There is no 409.

WHY THE 409 WENT. PR #221 chose serialisation-by-refusal because the index state was a CELL with several writers (heal marks, the rebuild's MarkIndexRebuildStart/Done bracket) and a claim/release count would pin 'indexing' forever on a leaked release. Under the machine there is one Index life and the state is derived from it (kb/invariants/repos/index-state/publish-chokepoint), so there is no cell and no second writer: Rebuild is just an event targeting the Index stage.

ACCEPTED COSTS kept from #221: no force flag; an error from a heal and an error from a rebuild share index_state 'error' (index_reason now carries the job's error text).

HISTORY: before #221 a manual rebuild touched no index state, so the chip read 'ready' throughout a rebuild; #221 bracketed it and 409'd while indexing; #411 replaced both with the machine.

MOVED (12c, PR #627, merge 97ae7c19, re-read there): handleStartRebuild is in internal/edge/web/api/jobs/handlers.go; it sends Rebuild(branch) to the lifecycle machine and takes no status cell or 409.
