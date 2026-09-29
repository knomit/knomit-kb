---
kind: pragmatic
type: policy
domain: [mcp, store, learn, atomicity, fleet]
confidence: 0.9
sources: 1
entities: [knomit_learn, retract, BatchWriteFactsMustExist, RetractMissingError, batchWriteLocked, SetBatchWriteUnlockedHookForTest, applyDedupMerge, checkSameSubjectCollisions, internal/mcp/learn.go, internal/store/fact_write.go]
motifs: [check-under-the-lock, precondition-inside-critical-section]
refs: ['kb://3ec012f5b4d2/kb/invariants/store/batch-write-atomicity/71c6dd17.md', 'src://7b4887ce51d9/internal/store/fact_write.go@53d102e998b25671eb2922a9ba25b064652ec6fd:2e76224e55cfed12bb21583c1ee5dd61268fa29f', 'src://7b4887ce51d9/internal/mcp/learn.go@53d102e998b25671eb2922a9ba25b064652ec6fd:e0d0638b408a51b19c11a2631d0ac70886cee6d6', 'src://7b4887ce51d9/internal/mcp/learn_move_test.go@53d102e998b25671eb2922a9ba25b064652ec6fd:5a6828f821d42a9bfe9ba894790ce2e739c65fdc', 'src://7b4887ce51d9/internal/store/interfaces.go@53d102e998b25671eb2922a9ba25b064652ec6fd:0a302dd4482233bc759c57cecdcdf5cd24b0622a', 'src://7b4887ce51d9/internal/mcp/learn_same_subject.go@53d102e998b25671eb2922a9ba25b064652ec6fd:8a85fe06e4dd7339798e2ffc21f50566956eaf32', 'https://github.com/knomit/knomit/pull/348']
---
# knomit_learn `retract` (F04 move) is all-or-nothing ON ONE BRANCH: every retract path must exist at the tip, checked INSIDE the branch write lock by BatchWriteFactsMustExist, else *RetractMissingError and nothing is written — it serialises two takes on one branch and is NOT exclusive across agents

F08 PR A (M1). knomit_learn accepts `retract: [paths]`; the facts and the deletions are ONE commit. Rules a consumer must act on:

1. PRECONDITION UNDER THE LOCK. The caller's retract paths go to store.FactIndex.BatchWriteFactsMustExist(files, deletes, mustExist, …). batchWriteLocked checks each mustExist path with blobAtCommit against the head it has just resolved under lockBranch, BEFORE any object is written. A miss returns *store.RetractMissingError{Paths} (errors.As-able); learn answers "not on this branch: <paths>; nothing written". This is what makes two local moves of one task serialise: the second sees the first's commit. Do NOT replace it with a FactExists check before the call (knomit_retract's shape): that check is outside the lock and goes stale — TestLearn_ConcurrentMovesOneWins holds ten callers at store.SetBatchWriteUnlockedHookForTest's pre-lock barrier to prove it.
2. WHY IT IS NEEDED: plain BatchWriteFacts silently succeeds on a delete of an absent LEAF (removeEntry filters by name) — only an absent DIRECTORY errors. Without the precondition a take of an already-taken task commits.
3. Operation and message: facts+retract → author <id>+move@agents.knomit.io, message `move: <moment>`; retract only (facts: [] is allowed only with a non-empty retract) → `retract` / `retract: <moment>`; facts only → unchanged `learn`. Response adds `operation`, `retracted`, `commit`.
4. Normalisation is knomit_retract's: federate.WriteRepoPath → fact.NormalizePath → a private path that is not writable job state is refused. Duplicates collapse.
5. A path both written and retracted is refused ("both written and retracted"), checked after the build AND after dedup; applyDedupMerge declines a match the call retracts (the fact is written at its fresh path), and the same-subject gate skips candidates the call retracts. Subsumed-hypothesis deletions ride the same commit but are NOT preconditions.
6. The ref gate still judges the PRE-write head, so a new fact may cite the path the same call retracts (TestLearn_RefToRetractedPathPasses).

MISREADING TO AVOID: "atomic move = exclusive take". It is atomic per BRANCH only. Two agents on two branches can both move the same task: the two deletions merge clean (delete/delete) and both working copies survive. Cross-agent exclusivity does not exist anywhere in knomit; F08's claim protocol is a heuristic plus a backstop on top of this primitive. REST has no equivalent (MCP and the script host only).

KNOWN TEST GAP (reviewer N1, PR #348): the same-subject skip of candidates the call retracts (checkSameSubjectCollisions, the `retracting[...]` case logged "learn: same-subject candidate skipped, retracted in the same call" in internal/mcp/learn_same_subject.go) has NO test. The test fixtures use the length embedder, whose model id has no calibrated band in params.ForModel, so the same-subject gate is off in every move test and the skip is never reached. Removing that case would still pass the suite. A test needs an embedder with a calibrated model id and a candidate in the same-subject band that shares a non-generic entity. Until one exists, treat this skip as unverified behaviour.

Merged to dev in #348 (merge commit 53d102e9); refs pinned there.
