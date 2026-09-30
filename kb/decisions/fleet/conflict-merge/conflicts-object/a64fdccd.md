---
type: observation
domain: []
confidence: 0.95
sources: 1
entities: [AttrConflicts, ReadConflicts, validConflicts, conflictsPolicy, factMergeResolutions, ConflictsConsensus, ConflictsMergeConsensus]
refs: ['kb://3ec012f5b4d2/kb/decisions/fleet/conflict-merge/rulings-1-5/141d28ba.md', 'kb://3ec012f5b4d2/kb/architecture/store/merge/merge-facts/6cd447f7.md', 'kb://3ec012f5b4d2/kb/architecture/repos/consensus/conflicts-matrix/fe2cd47a.md', 'kb://3ec012f5b4d2/kb/decisions/fact/ontology/root-attributes/90268b66.md', 'src://7b4887ce51d9/internal/fact/conflicts_settings.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:b7dd6009d769a118c4d7e153041f2a4d72dce093', 'src://7b4887ce51d9/internal/store/conflict_merge.go@b9cdf382af4dd52878f896f8f3c9cdea6213260a:55cc48c1d3192e9b7a0ec8017cdfbc234699f90c', 'https://github.com/knomit/knomit/pull/356']
---
# `conflicts` is an OBJECT {facts, state}, not one scalar: facts off|merge|merge:consensus|consensus settles a fact both sides changed; state off|consensus settles every non-fact path and any fact that cannot be merged losslessly. Sides are named by ROLE (the consensus side), never mine/theirs

THE USER (2026-09-30, verbatim):
- "files that aren't facts: for this I think we need another attribute, we cannot overload "conflicts". Or maybe we should make "conflicts" an object, and have "conflicts.facts", and "conflicts.state". For "conflicts.facts" the valid options are "merge", "mine", "theirs", "merge:consensus", for "conflicts.state" only "mine" or "theirs" - decide if "merge:consensus" is a valid optoin."
- After the master showed that "mine"/"theirs" are relative to whoever merges (a race, not a rule) and proposed `consensus` instead: "so far so good".
- On the PR 2 list: "- reshape: ok - add: ok - stop: ok - ok".

OPTIONS CONSIDERED:
(a) overload the one scalar `conflicts` for facts and non-facts;
(b) a separate new attribute for non-fact paths;
(c) `conflicts` as an object with `facts` and `state`, with values `mine`/`theirs`;
(d) the object with ROLE-named values: `consensus` = the side that is on, or will reach, the consensus branch.

THE CHOICE: (d).

```yaml
conflicts:
  facts: merge        # off | merge | merge:consensus | consensus
  state: consensus    # off | consensus
```

`facts: consensus` takes the consensus side's whole version, and does so for a modify/delete too. `state: consensus` takes the consensus side's version of any non-fact path, and of a fact MergeVersions refuses (unparsable, kinds differ, lossy). `merge:consensus` is NOT a state value, because there is nothing to merge field by field.

The consensus side per site (PR 1's mapping, verified for the new values):
- a peer's sync: the incoming consensus branch (src);
- the rebase replay: onto (dst);
- MergePushed/MergeConsensus: the host's own agent branch (dst).

The scalar form is gone, with no migration (nothing was released). A bad value is handled as for `consensus`: a warning on open, fatal only for a new ontology, and read as off for both keys.

RATIONALE: mine/theirs name a side RELATIVE to whoever runs the merge. The host and the peer would each keep their own version, which makes the outcome depend on who merges first: a race, not a rule. A role name picks the same version on both instances, and the two-instance matrix (architecture/repos/consensus/conflicts-matrix) shows every non-off combination converging with zero idle commits. One scalar cannot say different things for facts, which can be field-merged, and for state files, which cannot.

REJECTED:
- mine/theirs (relative, racy);
- overloading one scalar for facts and non-facts;
- `merge:consensus` for state;
- the PR 2 UI "merge" button / `resolution: "merge"` (with `conflicts` set, the plain Merge already applies the setting; a button would be a second way to do the same).

NON-SCOPE: this is not a per-path or per-topic setting, and not a human or LLM merger. `off` still means each site's own side-pick (the host refuses), recorded.
