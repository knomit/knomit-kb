---
type: reference
domain: [repos, consensus, merge, conflicts]
confidence: 0.9
sources: 1
entities: [TestConflictsMatrix_T1_HostedAuto, TestConflictsMatrix_T2_GitHubLikeOrigin, conflicts, 'consensus: auto']
motifs: [role-named-side, merge-base-moves-conflict]
refs: ['src://7b4887ce51d9/internal/repos/conflicts_matrix_test.go@62190f2ac3c64c38b72099c2f4551c26597a4754:4ea9957f3a891da83af682b2bb147cf2cb1fc6b8', 'src://7b4887ce51d9/internal/store/conflict_merge.go@62190f2ac3c64c38b72099c2f4551c26597a4754:55cc48c1d3192e9b7a0ec8017cdfbc234699f90c', 'kb://3ec012f5b4d2/kb/architecture/store/merge/merge-facts/6cd447f7.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/consensus/conflict-retry-peer-wins/c05b7b09.md']
---
# Two-instance outcome table for `conflicts` {facts, state} under consensus: auto (host B, peer A): what each value ends with on B's agent and consensus branches and on A, and which rows converge by themselves

Measured by TestConflictsMatrix_T1_HostedAuto (internal/repos/conflicts_matrix_test.go). The setup has real receive-pack pushes, B's consensus merger and A's ordinary sync, with no merge called by hand.

Fixture: from the base (body "base", confidence 0.7), A edits the fact to body "A body", confidence 0.95, entity ea, and B edits it to body "B body", entity eb. Both also change the non-fact notes.txt.

Sequence:
1. A pushes, and B's merger settles the conflict as the setting says, with B's own branch as the consensus side. Where a key is off it refuses the WHOLE merge.
2. A syncs and settles what B refused. Here the incoming consensus branch (B's version) is the consensus side, and A keeps its own version for a key set to off.
3. B's retry is clean.

Final versions (fact / notes.txt), the same on B's agent branch, B's consensus branch and A, unless noted:
- off/off: B keeps B/B, A keeps A/A. It does NOT converge: A's LocalWins merge is tree-identical, so no commit is written and B never retries. It is left in B's Pushed list for a human.
- off/consensus: A's fact, B's notes (first merge refused).
- merge/off: merged (A body, 0.95, [ea, eb]), A's notes (refused first: the notes conflict).
- merge/consensus: merged, B's notes (settled on B in one merge).
- merge:consensus/off: B body, 0.95 (only A changed it), [eb, ea]; A's notes.
- merge:consensus/consensus: the same fact, B's notes.
- consensus/off: B's fact whole, A's notes.
- consensus/consensus: B's fact and B's notes.
- absent (the auto default): identical to merge/consensus.

Every row adds zero commits over three idle rounds, and A's tip does not move.

T2 (TestConflictsMatrix_T2_GitHubLikeOrigin) uses a bare origin whose consensus branch is `trunk`, advanced only by a `git merge --no-ff` 'PR merge', with facts: merge / state: consensus. After A's PR lands, B's sync is where the conflict is met, with trunk as the consensus side. The result: the fact is merged (A's body by confidence, both entities) and notes.txt is trunk's (A's). Both instances and trunk end byte-identical, with zero origin commits over three idle rounds.

CONSEQUENCE: any value except off/off converges by itself within two rounds, and a key set to off on the HOST only delays settlement to the peer's next sync. MISREADING: 'mine/theirs' outcomes would differ by who merges first. `consensus` names a side by role (the consensus branch), so both instances name the same version.
