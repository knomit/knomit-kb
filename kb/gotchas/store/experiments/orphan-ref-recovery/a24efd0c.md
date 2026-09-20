---
type: observation
domain: [store, branches, experiments, recovery]
confidence: 0.9
sources: 1
entities: [OpenExperiment, RollbackExperiment, ErrOrphanExperimentRef, dropExperiment, DropBranch, CreateBranch, EnsureBranch, pipeline_watermarks]
motifs: [adoption-hides-provenance, recovery-misses-own-case]
refs: ['kb://3ec012f5b4d2/kb/conventions/store/create-branch-clones-three-things/8ef32469.md', 'kb://3ec012f5b4d2/kb/decisions/store/experiments/fork-copies-watermarks/33dba5d6.md', 'src://7b4887ce51d9/internal/store/experiment.go@acd9f39e21f5dc0150194e21a6066c6d26b108a4:119e25b665da771d5f0e8fcffd9f7d554be1667a', 'src://7b4887ce51d9/internal/store/branch.go@acd9f39e21f5dc0150194e21a6066c6d26b108a4:b0b37fdca41c41d2fc7f0b29da60cd5efc418747', 'https://github.com/knomit/knomit/pull/235']
---
# OpenExperiment REFUSES an exp/<name> ref that has no experiments row instead of adopting it, and RollbackExperiment is the recovery — except when the ref has no branches row either, which CreateBranch's ref-before-row ordering can produce

CreateBranch NO-OPS when the ref already exists, so an OpenExperiment that just called it would hand back a branch carrying whatever that ref pointed at while the new row claims a fresh fork of the agent branch — someone else's commits, presented as yours, and committable into the agent branch under a name that says "new experiment". OpenExperiment therefore refuses with ErrOrphanExperimentRef before creating anything.

That refusal is only reasonable because RollbackExperiment CLEARS an orphan: it validates the name, confirms the ref resolves, and runs the same dropExperiment teardown (branch, pipeline_watermarks rows, the meta `last_commit:`/`graph_schema_version:` keys, the row), returning nil. Without that, the only exit would be deleting a ref by hand, and the person who hits this is working in a UI. Genuinely absent — no ref AND no row — is still ErrNoSuchExperiment, so the orphan path cannot turn "there was never anything here" into a silent success.

THE KNOWN GAP, and do not assume it away: the cleanup goes through DropBranch, which needs the `branches` row. CreateBranch writes the git REF BEFORE calling EnsureBranch, so an EnsureBranch failure leaves an orphan ref with NO branches row — and RollbackExperiment then returns DropBranch's error instead of clearing it. The recovery does not cover its own worst case. The fix is for dropExperiment to tolerate a missing branches row and remove the ref directly; whether CreateBranch should invert its ordering is a separate question affecting every caller.
