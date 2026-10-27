---
kind: pragmatic
type: policy
domain: [web, origin-sessions, concurrency, windows]
confidence: 0.92
sources: 1
entities: [sessionManager, claimRun, reclaimRun, reclaimDeadRuns, isChild, removeChild, pki.LockFile, runHooks, TestReclaimRun_NeverStripsANewRunOfItsLock, sessionManager.released, sessionManager.close]
motifs: [liveness-by-kernel-lock, check-act-gap]
refs: ['src://7b4887ce51d9/internal/edge/web/api/origins/session_manager.go@c3c293b01e0b4004b9f61bb0a6fb1716ac6385b0:e3110a814265b2f88804a69dc814f949afef4dfa', 'src://7b4887ce51d9/internal/foundation/pki/lock_unix.go@c3c293b01e0b4004b9f61bb0a6fb1716ac6385b0:065d9bea604f6ac273d10ef7d5652350167c5d17', 'src://7b4887ce51d9/internal/foundation/pki/lock_windows.go@c3c293b01e0b4004b9f61bb0a6fb1716ac6385b0:7b061707538dc5768bb74348b30e56a495fb8303', 'src://7b4887ce51d9/internal/edge/web/api/origins/session_manager_test.go@c3c293b01e0b4004b9f61bb0a6fb1716ac6385b0:948bd6f8d712079e6f580b291d2c0b3e0a35189b', 'src://7b4887ce51d9/internal/app/origin_sessions_wiring_test.go@c3c293b01e0b4004b9f61bb0a6fb1716ac6385b0:011a5e18a7d03c17af15731f860991311c133632', 'kb://3ec012f5b4d2/kb/invariants/web/origin-sessions/7971585d.md', 'https://github.com/knomit/knomit/pull/626', 'https://github.com/knomit/knomit/pull/652']
---
# Each origin-session manager keeps its session folders in its own run directory <base>/run-<id>/ and holds an OS file lock on <base>/run-<id>.lock for its whole life; the startup sweep removes only runs whose lock it can take (owner exited), never a run whose lock is held, whichever process holds it

WHY. Two processes on one home is supported: a `knomit serve` beside the desktop's (internal/app/tls.go), an unsupervised serve that finds the socket held and serves TCP only, a desktop that attaches to a serve or spawns `serve --supervised`. The first version of PR #626 swept every `session-*` folder at startup; that deleted the OTHER process's live wizard and made its /commit SwapStore fail. Hence per-process runs.

LAYOUT. `<base>/run-<id>.lock` (liveness lock, held for the manager's life), `<base>/run-<id>/` (run directory), `<base>/run-<id>/session-*` (one folder per session). base is Server.OriginSessionsDir (`<home>/origin-sessions` under internal/app); "" gives a private base under os.TempDir() (knomit-origin-sessions-*) made on the first Create and removed when the lifetime ends.

LIVENESS IS THE KERNEL LOCK, NOT A PID. The lock is pki.LockFile (flock on POSIX, LockFileEx on Windows); a pid can be reused, the kernel drops the lock when its holder dies.

ORDER. claimRun makes the lock file (os.CreateTemp) and locks it BEFORE it makes the run directory, then checks the path still names the file it locked (os.SameFile); on a mismatch it retries, three attempts in all. A sweeper that probed the new file in the gap would take it for a dead run's and remove it.

RECLAIM. reclaimRun takes the lock (50 ms try), removes the run directory, and unlinks the lock file, all WHILE HOLDING the lock; only then unlocks and closes. Why the held unlink matters: unlinking after the unlock let a claimer that had just created that file take the lock in the gap, pass its sameFileAt check and make its run directory, and the late unlink then stripped a LIVE run of its lock file (which leaks forever if that process later crashes). Found as a re-review LOW on #626, fixed in 76feaee7. WINDOWS: the held unlink fails there (Go opens files without FILE_SHARE_DELETE), so reclaimRun retries the unlink after the close. That is still safe: a live claimer keeps its handle open from CreateTemp for the run's whole life, so the removal fails while any claimer has the file and succeeds only when nobody does. If the directory removal fails the lock file is kept and the next start retries. (The manager's own close() unlocks, closes, then unlinks its lock file, after its run directory is already gone; nothing is left for a sweeper to strip.)

CONTAINMENT. Every removal goes through isChild(parent, p, prefix, suffix): a direct child of the parent with the layout's prefix/suffix. A run folder with no lock file, a foreign `.lock` file, an old-layout `session-*` folder directly in the base, and anything outside the base are never touched.

BACKGROUND. newSessionManager runs the sweep (reclaimDeadRuns) on its own goroutine, off the boot path; it starts at the first router build (`a.Handler()` in cmd/serve.go), before app.OpenListeners. That is safe because the sweep touches only dead runs. `staleDone` closes when it finishes, for tests.

DOES NOT MEAN: that folders are always cleaned at exit. Lifetime-end removal is asynchronous (context.AfterFunc) and nothing waits for it, so a process that exits right after Close can leave its run behind; the next manager over the same base reclaims it because the lock died with the process. Nor does it mean the sweep can be pointed at a base shared with non-knomit files: only `run-*.lock` files and their `run-*` siblings are ever considered.

Tests: TestSessionManager_StartupSweepLeavesAnotherLiveManagersRun, TestSessionManager_StartupSweepReclaimsADeadRun, TestSessionManager_RemovalStaysInsideTheLayout, TestReclaimRun_NeverStripsANewRunOfItsLock (forces the interleaving deterministically through runHooks, a PARAMETER not package vars so other tests' background sweeps can never reach them), TestNew_OriginSessionsHaveTheAppsLifetimeAndHomeDir (internal/app).
