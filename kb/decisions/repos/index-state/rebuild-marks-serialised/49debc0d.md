---
type: observation
domain: [repos, web, index, concurrency, api]
confidence: 0.95
sources: 1
entities: [handleStartRebuild, MarkIndexRebuildStart, MarkIndexRebuildDone, RepoInstance.publishIndex, mirrorIndexing, Manager.openOne, Manager.Add, openRegistered, Manager.Start, TaskHub.Start, IndexStateIndexing]
motifs: [single-writer-by-refusal, visible-failure-over-silent]
refs: ['kb://3ec012f5b4d2/kb/invariants/repos/index-state/publish-chokepoint/4110015c.md', 'kb://3ec012f5b4d2/kb/decisions/repos/create-job/done-means-indexed/0c1b3368.md', 'kb://3ec012f5b4d2/kb/incidents/repos/clone-create-index-stuck-indexing/d8faf2fd.md', 'kb://3ec012f5b4d2/kb/incidents/web/index-chip-stale-after-heal/1368e90f.md', 'kb://3ec012f5b4d2/kb/architecture/repos/lifecycle/fa0ea608.md', 'src://7b4887ce51d9/internal/web/handlers_jobs.go@3b6f80f564f1b7371e35ed152c5b7741bf029ccd:ab33cba2df04eea9cd067ed537ef9650cad277bc', 'src://7b4887ce51d9/internal/repos/instance.go@3b6f80f564f1b7371e35ed152c5b7741bf029ccd:a9e0e940da8f7eae7a8fe86b0b40825c86a0320f', 'src://7b4887ce51d9/internal/repos/lifecycle.go@3b6f80f564f1b7371e35ed152c5b7741bf029ccd:d3d19e708815b88b3a7562719ee7fac94d64c885', 'https://github.com/knomit/knomit/pull/221']
---
# A manual rebuild marks index state through the same chokepoint as the heal, and is REFUSED with 409 while the state is already indexing — serialisation was chosen over a claim/release count because a leaked release pins `indexing` forever and silently, while a 409 fails visibly

Decided 2026-09-18 (PR #221). A rebuild brackets `IndexManager().Rebuild` with `MarkIndexRebuildStart` / `MarkIndexRebuildDone(err)` — exported aliases for the same three marks the startup heal uses, so everything still publishes through `publishIndex` — and the endpoint refuses the request with **409** while `IndexStatus()` is already `indexing`, with a body naming the state and saying it is temporary.

HISTORY, the pre-fix state: `handleStartRebuild` called `Rebuild` directly with ZERO references to `markIndex*`, so `IndexStatus()` read **`ready` for the whole of a manual rebuild**. The chip, the REST payload and #219's event stream all reported a ready index while it was being rebuilt. #219 deliberately scoped itself to the startup heal and left this; that is what this decision closes.

WHY A GATE WAS NEEDED AT ALL. Marking makes the rebuild a SECOND writer of a cell with no notion of who owns it, and the create job INFERS from that cell: `mirrorIndexing` returns the instant the state leaves `indexing`, and its return goes straight into `emit(Event{Step:"done", Pct:100, IndexState: indexState})`. A rebuild finishing first while a heal was in flight would flip the cell to `ready` and make the create report done at 100% over a half-built index — `done-means-indexed` violated directly.

THE THREE OPTIONS.

1. **Claim/release count** — claim increments and sets `indexing`, release decrements, only the last sets a terminal. It preserves the create job's inference exactly, its failure mode is a LATE `done` rather than a wrong one, and it keeps one cell with one meaning. All three points were accepted as correct.
2. **A separate rebuild state** — rejected for putting a second concept in front of the UI.
3. **Serialise** — refuse the rebuild while indexing. Chosen.

WHY SERIALISE WON, and it was on FAILURE MODE, not elegance. A count converts "every exit path marks" into "every claim pairs with a release" — strictly harder to audit, because a release must happen on every exit including panics, cancellations and early returns, and **a leaked claim pins `indexing` forever and is silent**. That is precisely the failure class of the 2026-08 stuck-indexing incident, in the same cell. A 409 has no pairing to leak, keeps single-writer LITERALLY rather than by mechanism, and fails VISIBLY — an operator sees it and retries. The count's remaining advantage is producers that do not exist.

WHAT THE 409 RESTS ON, and how it breaks. The mirror case — a heal starting while a rebuild is in flight — cannot be gated, because a heal is not a request. It is structurally unreachable, and ONLY while `openOne`'s callers stay what they are: `Manager.Add` (create, where the repo is brand new so no rebuild can target it; and restore, where Archive tore the instance down through `shutdown`, cancelling `indexCtx` and waiting `indexWg`, making an in-flight rebuild and a restore of the same repo contradictory) and `openRegistered` (called only from `Manager.Start`, at boot, when no repo is open). **A THIRD `openOne` CALLER — a hot reload, a rescan-in-place — SILENTLY REINTRODUCES THE RACE**, with nothing in the endpoint to notice it. That argument is a comment on the 409 in the code for exactly this reason.

Two rebuilds are a different conflict: `hub.Start("rebuild", ...)`'s per-op single-flight already refuses the second with "Job already running", and the two 409s are deliberately distinguishable by title.

ACCEPTED COSTS, decided rather than overlooked:

- **The manual-rebuild escape from a pinned `indexing` state is deliberately gone.** An operator whose heal is stuck can no longer rebuild out of it. The stuck-indexing class was fixed at its cause in August and a restart re-heals. **No force flag** — it would be a second writer through a side door, the exact thing this excludes.
- **`error` from a rebuild is indistinguishable from `error` from a heal** in the chip, the REST payload and the create job. The cost of one cell, and the reason option 2 was rejected arriving through the error state instead.
- **No progress mirroring**: a rebuild shows `indexing` with done/total 0/0, so the chip is indeterminate rather than counting up. `setIndexProgress` would be a further write site.
