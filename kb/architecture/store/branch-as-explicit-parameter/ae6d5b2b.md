---
type: observation
domain: [store, architecture, concurrency]
confidence: 0.85
sources: 0
entities: [git.Store, lockBranch, branchMu, configMu, SetOnCommit]
refs: ['src://knomit/.claude/plans/2026-03-27-git-branch-param.md@0938d83']
---
# git.Store has NO branch state; branch is explicit parameter on every method

git.Store is reduced to four fields: repo, storer, auth, signer — NO branch, NO agentID, NO db, NO commitLog atomic. Every public method takes branch string as the FIRST parameter (ReadFile, WriteFile, BatchWrite, DeleteFile, ListAll, ListDir, Log, LogPaginated, LastCommitForPath, Activity, WalkChangedFiles, Grep, DiffFiles, Tag, Sync, Push, HeadCommit, HasSharedHistory). Branch state is replaced by sync.Map-based per-branch locking: lockBranch(branch) returns the per-branch *sync.Mutex's unlock function. CONCURRENCY MODEL: reads take NO lock (git objects are immutable once written, ref reads atomic); writes take per-branch lock so branch A writes don't block branch B writes; concurrent reads on same branch are safe; concurrent read+write on same branch is safe (write holds mutex, read proceeds lockfree, sees old-or-new ref atomically). ConfigureRemote uses a separate configMu sync.Mutex (only during setup). The Store.SetOnCommit callback signature is func(branch, hash string) — branch is now part of the notification.
