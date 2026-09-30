---
kind: pragmatic
type: policy
domain: [store, merge, conflicts, provenance]
confidence: 0.9
sources: 1
entities: [TrailerMerge, TrailerConflict, appendTrailerLines, TrailerValues, conflictLine, mergeTreesWithStrategy, Knomit-Merge, Knomit-Conflict]
motifs: [record-what-was-dropped, trailer-paragraph-sharing]
refs: ['src://7b4887ce51d9/internal/store/conflict_merge.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:55cc48c1d3192e9b7a0ec8017cdfbc234699f90c', 'src://7b4887ce51d9/internal/store/branch_merge.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:3acd661c542c76cf6e2b5a6e667f8eea290d54ab', 'src://7b4887ce51d9/internal/store/conflict_merge_test.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:a2eb733ab2677afe8b439a4770961f6d557eb75a', 'src://7b4887ce51d9/internal/store/conflicts_object_test.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:1faa03f03ae50156926b742bbf72cf7ff6b93a31', 'src://7b4887ce51d9/internal/repos/conflict_merge_test.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:66a27743b2e3a444b2f0d8388c714605ddc6b66a', 'https://github.com/knomit/knomit/pull/355', 'https://github.com/knomit/knomit/pull/356']
---
# Every conflict a commit settles is recorded in its LAST paragraph: Knomit-Merge for a merged (or retracted) fact, Knomit-Conflict for a side-pick. This includes plain LocalWins/RemoteWins with the conflicts setting absent, so 'nobody is told' no longer holds

Formats (one line per path, sorted by path, in the last paragraph; `none` marks an absent version):
- `Knomit-Merge: <path> strategy=<merge|merge_consensus> base=<blob> src=<blob> dst=<blob> out=<blob> decided=<fields>`. A retraction adds `out=none dropped=<side>-modify decided=delete`.
- `Knomit-Conflict: <path> kept=<src|dst> dropped=<side>-<modify|delete|add> strategy=<consensus|local_wins|remote_wins|refuse> base=… src=… dst=…`, with `reason=<why>` when a merge fell back (not-a-fact, src-unparsable, kind-mismatch, src-lossy, …) or a human chose the whole-set side (`reason=chosen`).

WHO WRITES THEM:
- the LocalWins and RemoteWins arms of mergeTreesWithStrategy, which now returns its record; a side-pick where both sides made the SAME change drops nothing and writes no line;
- factMergeResolutions;
- MergePushed's whole-set `side`.

Experiment per-path resolutions are a human's adjudication and are NOT recorded. A tree-identical no-op writes no commit, so there is nothing to record.

A REPLAYED commit (the rebase fallback) keeps its own message. The lines are appended INTO its existing trailer paragraph when the last paragraph already is one (`appendTrailerLines`), so its Knomit-Trace/Cause/Trigger still read: TrailerValue reads only the last paragraph. Read the lines back with `store.TrailerValues(msg, key)`, which returns every occurrence.

MISREADINGS:
- 'the trailer replaces the parents as evidence' is wrong: both input versions are the path's blobs in the merge commit's two parents, and the trailer names their blob ids.
- 'Knomit-Conflict only appears with conflicts: merge' is wrong: it appears whenever a merge commit dropped a side, the setting absent included.

SINCE CONFLICT-MERGE PR 2 (#356, `conflicts` is an object {facts, state}): the form is unchanged, and only the `strategy=` values follow the new names. Knomit-Merge now says `merge` (the confidence rule, PR 1's `confidence`) or `merge_consensus` (PR 1's `upstream`). A Knomit-Conflict line with `strategy=consensus` means the consensus side's version was taken, by `facts: consensus` or by `state: consensus`. Any other strategy there is the site's own side-pick for a key set to off. Commits written by PR 1 binaries keep the old names, so a reader must accept both.
