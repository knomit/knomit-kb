---
type: observation
domain: [store, history, materialize]
confidence: 0.8
sources: 1
entities: [vcs.LiveVersionAt, vcs.LiveChange, HistoryQuery.LiveAt, HistoryQuery.LiveRevision, recordedCommit, path_changes, commit_log]
motifs: [proxy-not-the-thing]
refs: ['kb://3ec012f5b4d2/kb/invariants/store/index/recorded-commit-holds-blob/4e532929.md', 'src://7b4887ce51d9/internal/engine/store/internal/vcs/live_version.go@33303d4d98634d2dd302fa8b3f67097c8641164d:852787268d9b99cfd05c872f02fc96a20928e653', 'src://7b4887ce51d9/internal/engine/store/index/factset_reads.go@33303d4d98634d2dd302fa8b3f67097c8641164d:bedac2b623ded9575cbed407905f2b6ea3b34efa']
---
# A historical 'version live at commit X' read from the derived history must take LiveChange's RESOLVED ENTRY as the commit (what recordedCommit stores in branch_facts), and must check commit_log 'deleted' rows itself: path_changes holds only add/modify rows

vcs.LiveVersionAt (internal/engine/store/internal/vcs/live_version.go), behind HistoryQuery.LiveAt's historical path, answers "which version of path was live at commit X" with no git object decoded. Two traps it had to avoid:

1. COMMIT: vcs.LiveChange returns (resolvesTo, row). branch_facts.commit_hash (index.recordedCommit, invariant 4e532929) is the RESOLVED ENTRY: a merge that carried the content unchanged resolves to the commit that wrote it. LiveRevision returns the ROW (the merge). A reader that must agree with branch_facts, e.g. a FactSet replay after the branch moved (F10), must take the entry, or the commit it shows differs from the fast path's for the same blob.
2. DELETION: path_changes has one row per add/modify on the first-parent line and NO row for a deletion, so LiveChange alone still answers the old version after a retraction. LiveVersionAt also scans commit_log rows with action 'deleted' for the path whose commit_fp depth lies between the change's depth and X's, and treats the path as gone when one is on X's first-parent line (jump pointers). A deletion followed by a re-add is live: the re-add is a newer change.

CONSEQUENCE: use HistoryQuery.LiveAt (fast path = branch_facts while the watermark IS the anchor; historical otherwise) instead of LiveRevision + a git tree read when you need a version at a past commit.

WHAT THIS DOES NOT MEAN: it is not 'the earliest introduction of the blob' (OPTION 1's resolution point for refs); it is the live introduction on X's first-parent line.
