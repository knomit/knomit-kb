---
kind: pragmatic
type: policy
domain: [testing, windows, ci, desktop]
confidence: 0.9
sources: 1
entities: [t.Cleanup, t.TempDir, TestApplyLogConfigNeverWritesTheServersFile, applyLogConfig, tools/desktop/logging_test.go]
motifs: [platform-hides-ordering-bug, lifo-teardown-order]
refs: ['kb://3ec012f5b4d2/kb/gotchas/ci/windows/tempdir-receive-pack-staging/240205a1.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/manager/set-orphans-live-instance/00b49c8c.md', 'src://7b4887ce51d9/tools/desktop/logging_test.go@8be999a0a9b68e8ab28d8416876c34c61f221d94:58df8bb284e7b95f51101c632de115ac2c9e7f1a', 'https://github.com/knomit/knomit/pull/503']
---
# Register t.Cleanup AFTER t.TempDir() when the cleanup closes a file that lives inside that directory: cleanups run last-in-first-out, so registering it first makes TempDir's RemoveAll run while the file is still open, which Windows refuses ('being used by another process') and Linux and macOS never notice

Found by a Windows verification run of PR #503 (head b7c26728, fixed in 5f318d0b before merge 8be999a0): TestApplyLogConfigNeverWritesTheServersFile failed ONLY on Windows with 'TempDir RemoveAll cleanup: unlinkat ...knomit-desktop.log: The process cannot access the file because it is being used by another process', reproducible, after every assertion had passed.

CAUSE. The test registered `t.Cleanup(func() { _ = applyLogConfig(<reset>) })` BEFORE calling `dir := t.TempDir()`. t.Cleanup functions run in reverse registration order, and t.TempDir registers its own RemoveAll cleanup at the moment it is called, so the order at teardown was: TempDir RemoveAll first (the logger still holds knomit-desktop.log open), reset second. Windows cannot delete an open file; Unix can, so the same test was green on macOS and Linux.

RULE. In a test that points a long-lived handle (a logger, a database, a listener, a lock) at a file inside t.TempDir(), call t.TempDir() FIRST and register the cleanup that closes the handle AFTER it, so the close runs before the directory is removed. The fix in that test was moving the t.TempDir() line above the t.Cleanup line (tools/desktop/logging_test.go).

HOW TO RECOGNISE IT: a Windows-only failure naming 'TempDir RemoveAll cleanup' and a file, after green assertions. Do not confuse it with the receive-pack staging directory race (kb/gotchas/ci/windows/tempdir-receive-pack-staging) or a fixture that orphans open handles (kb/gotchas/repos/manager/set-orphans-live-instance); those have a different cause (background work or a leaked instance). Check cleanup registration order first.

NOT MEANT: it is not a product defect, and it is not fixed by retrying or by sleeping before the removal.
