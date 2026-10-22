---
type: observation
domain: [testing, logging, repos, concurrency]
confidence: 0.9
sources: 1
entities: [syncBuf, captureLog, captureLogs, restoreLoggerLast, zerolog log.Logger, t.Cleanup, triggerDispatcher]
motifs: [lifo-teardown-order, unsynchronized-shared-sink]
refs: ['kb://3ec012f5b4d2/kb/gotchas/testing/global-state/748698d5.md', 'kb://3ec012f5b4d2/kb/gotchas/testing/windows/beca29df.md', 'src://7b4887ce51d9/internal/corpus/repos/fleet_reconcile_test.go@9baf12dc2575ba4b0dcb16fb8645aef5c90087c1:dee14e8fc4335d4ae7c1a46fb9937adf23cea984', 'src://7b4887ce51d9/internal/corpus/repos/testhelpers_test.go@9baf12dc2575ba4b0dcb16fb8645aef5c90087c1:433644bf0de53c173fa9fab556e815f1b5ed31ac', 'src://7b4887ce51d9/internal/corpus/repos/triggers_test.go@9baf12dc2575ba4b0dcb16fb8645aef5c90087c1:5f09363c94769f3f711726874d6bd6408cf19713', 'https://github.com/knomit/knomit/pull/603']
---
# A test that captures log output while background goroutines can still log must write into a mutex-guarded buffer (syncBuf locks Write AND String), never a bare bytes.Buffer, and must restore the global logger AFTER the fixture's teardown

Swapping the global zerolog log.Logger to zerolog.New(&bytes.Buffer) is a data race whenever more than one goroutine logs: zerolog does not lock a plain writer. A fleet test runs one trigger dispatcher goroutine per member repo, so TestFleetTick_NothingInstalledLogsNothing raced Buffer.Write against Buffer.Write (the reviewer reproduced it at about 5 failing runs out of 15 under -race before the fix, none after).

Two rules, both needed:
1. The capture sink is syncBuf (internal/corpus/repos: captureLog, captureLogs): a mutex around Write AND around String. zerolog.SyncWriter locks only writes, so the assertion's buf.String() would still race with a late write. The same mutex-guarded type is used for the other captures that have server goroutines: tools/bridge/knomitapi, tools/bridge/claude, internal/foundation/pki and internal/engine/store tests.
2. Register the global-logger restore BEFORE building the fixture (restoreLoggerLast(t) in internal/corpus/repos testhelpers). t.Cleanup is LIFO, so a restore registered after the Manager/fleet fixture runs while the fixture's goroutines can still log and re-write the global var.

WHAT THIS DOES NOT MEAN: the swap of the global logger variable itself is not made safe against live goroutines of a PARALLEL test; log-swapping tests stay sequential (the package's TestMain keeps them so). Captures with no goroutines at all (a plain strings.Builder in a goroutine-free test, or a SyncWriter that is closed before it is read) were deliberately left alone. The per-logger .Level(...) does not touch zerolog.GlobalLevel, so the GlobalLevel save/restore rule in the sibling gotcha does not apply to these captures.
