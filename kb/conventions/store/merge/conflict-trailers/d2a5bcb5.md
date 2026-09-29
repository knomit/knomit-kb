---
kind: pragmatic
type: policy
domain: [store, merge, conflicts, provenance]
confidence: 0.9
sources: 1
entities: [TrailerMerge, TrailerConflict, appendTrailerLines, TrailerValues, conflictLine, mergeTreesWithStrategy, Knomit-Merge, Knomit-Conflict]
motifs: [record-what-was-dropped, trailer-paragraph-sharing]
refs: ['src://7b4887ce51d9/internal/store/conflict_merge.go@856759258ead41821f1ab17626a5bc0f05d50fe0:10bf40f92409a4447960507fe31f60a6bc336026', 'src://7b4887ce51d9/internal/store/branch_merge.go@856759258ead41821f1ab17626a5bc0f05d50fe0:57485dd93a335efea8f4573f16d81a7b0a46b51e', 'src://7b4887ce51d9/internal/repos/conflict_merge_test.go@856759258ead41821f1ab17626a5bc0f05d50fe0:83f08f05c905d9eb46341d47cd8e0f02b43e53c8']
---
# Every conflict a commit settles is recorded in its LAST paragraph: Knomit-Merge for a merged (or retracted) fact, Knomit-Conflict for a side-pick. This includes plain LocalWins/RemoteWins with the conflicts setting absent, so 'nobody is told' no longer holds

Formats (one line per path, sorted by path, in the last paragraph; `none` marks an absent version):
- `Knomit-Merge: <path> strategy=<confidence|upstream> base=<blob> src=<blob> dst=<blob> out=<blob> decided=<fields>`. A retraction adds `out=none dropped=<side>-modify decided=delete`.
- `Knomit-Conflict: <path> kept=<src|dst> dropped=<side>-<modify|delete|add> strategy=<conflict strategy> base=… src=… dst=…`, with `reason=<why>` when a merge fell back (not-a-fact, src-unparsable, kind-mismatch, src-lossy, …) or a human chose the whole-set side (`reason=chosen`).

WHO WRITES THEM:
- the LocalWins and RemoteWins arms of mergeTreesWithStrategy, which now returns its record; a side-pick where both sides made the SAME change drops nothing and writes no line;
- factMergeResolutions;
- MergePushed's whole-set `side`.

Experiment per-path resolutions are a human's adjudication and are NOT recorded. A tree-identical no-op writes no commit, so there is nothing to record.

A REPLAYED commit (the rebase fallback) keeps its own message. The lines are appended INTO its existing trailer paragraph when the last paragraph already is one (`appendTrailerLines`), so its Knomit-Trace/Cause/Trigger still read: TrailerValue reads only the last paragraph. Read the lines back with `store.TrailerValues(msg, key)`, which returns every occurrence.

MISREADINGS:
- 'the trailer replaces the parents as evidence' is wrong: both input versions are the path's blobs in the merge commit's two parents, and the trailer names their blob ids.
- 'Knomit-Conflict only appears with conflicts: merge' is wrong: it appears whenever a merge commit dropped a side, the setting absent included.
