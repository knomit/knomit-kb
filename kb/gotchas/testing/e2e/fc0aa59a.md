---
type: observation
domain: [testing, e2e, disk]
confidence: 0.9
sources: 1
entities: [make e2e-clean, e2e/clean.mjs, knomit-e2e, knomit-remote, KNOMIT_HOME, knomit serve, E2E_BINARY, e2e/helpers/cleanup.ts]
motifs: [scope-by-ownership]
refs: ['src://7b4887ce51d9/e2e/clean.mjs@a2439e311436e56374081ce94f553e42f007a133:1022fe8df0718921fda2a793996d252718370a0a', 'src://7b4887ce51d9/Makefile@a2439e311436e56374081ce94f553e42f007a133:474060ecf7538ee32fd959050d3b4a9c957a5a32#L379-L382', 'src://7b4887ce51d9/e2e/helpers/cleanup.ts@a2439e311436e56374081ce94f553e42f007a133:4647029d284c436028a7062cd004d41916067af1', 'kb://3ec012f5b4d2/kb/gotchas/testing/e2e/aborted-run-leaks/d0b2e7fc.md']
---
# `make e2e-clean` kills only `knomit serve` processes started from THIS checkout's binary (dist/knomit, .e2e-bin/knomit or E2E_BINARY) with a KNOMIT_HOME under knomit-e2e-*, then removes knomit-e2e-* and knomit-remote-* temp homes that no surviving process holds

e2e/clean.mjs (Makefile `e2e-clean`) reads the process table with `ps` (environment included) and kills a process only when ALL of these hold: its command is `<binary> serve` with the binary exactly one of `<checkout>/dist/knomit`, `<checkout>/.e2e-bin/knomit` or `$E2E_BINARY`, AND its environment has `KNOMIT_HOME=<...>knomit-e2e-*`. Any other `knomit serve` (a user instance on port 19278, another checkout's e2e server) is never signalled. It then deletes `knomit-e2e-*` and `knomit-remote-*` directories in the temp dir EXCEPT any home that a surviving `serve` process still names (printed as 'kept ... held by a process not started from this checkout'), and removes `e2e/.e2e-state.json`.

Consequence: run it from the checkout that started the leaked run, with the same E2E_BINARY if you used one; use it instead of hand-sweeping `knomit-e2e-*`. Verified in the PR: it killed 3 leaked servers of the checkout and removed their homes.

WHAT THIS DOES NOT MEAN: it is not a clean-everything command. A leaked server whose binary path is not one of the three above (for example a binary run via a different E2E_BINARY than the one in your shell now) is neither killed nor is its home removed, and a home whose server belongs to another checkout survives by design. It also only recognises `knomit-e2e-*` homes for kills, while the `knomit-remote-*` homes of the remote-migration journey are only deleted, never used to identify a process to kill.
