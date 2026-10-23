---
type: observation
domain: [store-git, commit_log, pagination, web]
confidence: 0.85
sources: 1
entities: [CommitLogQuery, CommitLogCursorAfter, lookupMaxRowid, commit_log, handleFactCommits, logPaginated, max_rid]
motifs: [cursor-key-mismatch]
refs: ['src://7b4887ce51d9/internal/engine/store/git/commitlog.go@99e0d3a5ccf7148fc3248c9f79627d57f4599a5e:6a211b394fd99da45f507815ab41e6c8dbde04b6', 'src://7b4887ce51d9/internal/edge/web/handlers_fact_sub_test.go@99e0d3a5ccf7148fc3248c9f79627d57f4599a5e:581ecfef64f76bfeb64efbb03a7de41d6f4b9783', 'kb://3ec012f5b4d2/kb/conventions/store-git/commitlog-query-cursor-three-modes/43711204.md', 'kb://3ec012f5b4d2/kb/conventions/store-git/commitlog-path-cond-md-vs-glob/d801013a.md', 'https://github.com/knomit/knomit/pull/614']
---
# KNOWN DEFECT: a path-filtered CommitLogQuery with an After cursor at a multi-file commit returns that commit again, because the page key is computed over the filtered rows and the cursor key over all of the commit's rows

In storegit.Storer.CommitLogQuery (internal/engine/store/git/commitlog.go) each page row's max_rid is MAX(cl.rowid) over the commit's rows that match the PATH condition (GROUP BY cl.commit_hash inside the path-filtered subquery). The cursor's rid, from lookupMaxRowid, is SELECT MAX(rowid) FROM commit_log WHERE commit_hash = ? over ALL of that commit's rows, whatever the path. The After condition is (ts < ? OR (ts = ? AND max_rid < rid)). When the cursor commit touched the requested path AND another file whose commit_log row has a higher rowid, the filtered max_rid is lower than the cursor rid, so the cursor commit itself passes the condition and is returned again as the first item.

CONFIRMED (W0g2, PR #614 review; documented in the comment above the commits fixture in handlers_fact_sub_test.go): GET .../facts/<path>/commits?after=c5&limit=5, where c5 wrote the path kb/a.md and then kb/c.md in one batch, returns [c5, c3, c1] instead of [c3, c1]. The From cursor is unaffected (its condition is max_rid <= rid, which includes the cursor commit anyway), and so is Before (max_rid > rid, which still excludes it). The branch-wide log with no path is unaffected, because both keys then cover every file of the commit. Not fixed and not filed as of the merge commit; triage item.

CONSEQUENCE: a client paging a per-fact history with the next link (which uses after=<last commit>) can show one multi-file commit twice. A test that pins per-fact paging must not end a page at a multi-file commit unless it means to pin this defect (the W0g2 fixture ends pages at single-file commits). Any fix must make both keys cover the same rows; changing only the cursor lookup to the path-filtered rowid would also change From and Before.

DOES NOT MEAN: that the branch-wide /commits list, or From/Before paging of a per-fact log, repeats commits. It also does not mean the web layer is at fault: logPaginated passes the cursor through unchanged.
