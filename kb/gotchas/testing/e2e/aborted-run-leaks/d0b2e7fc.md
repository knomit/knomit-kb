---
type: observation
domain: [testing, e2e, build, disk]
confidence: 0.85
sources: 1
entities: [e2e/global-setup.ts, e2e/global-teardown.ts, e2e/helpers/cleanup.ts, knomit-e2e, KNOMIT_HOME, .e2e-state.json, knomit serve, make e2e-clean]
motifs: [happy-path-cleanup]
refs: ['src://7b4887ce51d9/e2e/global-setup.ts@a2439e311436e56374081ce94f553e42f007a133:43db898456cda3b8584a41e81ff6daa570d43bb9', 'src://7b4887ce51d9/e2e/global-teardown.ts@a2439e311436e56374081ce94f553e42f007a133:34faa1ef43decce5e0905ab98026b164a8084e5e', 'src://7b4887ce51d9/e2e/helpers/cleanup.ts@a2439e311436e56374081ce94f553e42f007a133:4647029d284c436028a7062cd004d41916067af1', 'src://7b4887ce51d9/e2e/clean.mjs@a2439e311436e56374081ce94f553e42f007a133:1022fe8df0718921fda2a793996d252718370a0a', 'kb://3ec012f5b4d2/kb/gotchas/testing/e2e/fc0aa59a.md']
---
# A KILLED or Ctrl-C'd e2e run still leaves its knomit serve process and its knomit-e2e temp home behind (a setup that THROWS now cleans itself); each home holds a copy of the model cache, and leaked ones filled the disk

e2e/global-setup.ts makes `$TMPDIR/knomit-e2e-*` as KNOMIT_HOME, copies the ONNX model cache into it (about 0.9 GB per home), and spawns `knomit serve` DETACHED in its own process group. Since PR #484 it writes `.e2e-state.json` as soon as the home exists (home only), rewrites it with pid, port and baseURL right after the spawn, and a setup that throws calls the same `cleanupFromState()` that e2e/global-teardown.ts uses (e2e/helpers/cleanup.ts), so a failing readiness wait, `createRepo` or seed no longer leaks. A stale state file from an earlier run is cleaned first. The `freshKnomit` fixture and the remote-migration instances likewise start their servers inside try/finally and kill the server and remove the home when setup fails.

What STILL leaks: a run killed (SIGKILL, closed terminal) or Ctrl-C'd before teardown runs, because no JS cleanup executes; the next run's stale-state cleanup then removes only what the state file names, and only when a later run starts. A disk-full abort during the model copy can also leave a partial home. History: 2026-10-10, during the facets build, several aborted runs by parallel workers filled the disk (37 GB, 100%) with leaked `knomit serve` processes and `knomit-e2e-*` homes, and a later run died with ENOSPC; that was before the self-cleaning setup.

Consequence: after any killed e2e run, run `make e2e-clean` from the checkout that ran it (scope rule and limits: kb/gotchas/testing/e2e/fc0aa59a.md). Do NOT hand-sweep every `knomit-e2e-*`: another session's live run owns one, and deleting it mid-run (it happened) breaks that run's setup while the model cache is still being copied. Limit a parallel build to one e2e runner at a time.

WHAT THIS DOES NOT MEAN: a clean run is not affected; teardown removes its server and home. It is also not true any more that a throwing setup leaks (older versions of this fact said so and that the fix 'has not been done'); do not treat a failed-setup run as needing manual cleanup before you have checked that `make e2e-clean` finds anything. `make e2e-clean` from one checkout does not clean servers started from another checkout's binary.
