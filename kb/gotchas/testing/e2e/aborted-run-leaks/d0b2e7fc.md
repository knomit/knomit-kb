---
type: observation
domain: [testing, e2e, build, disk]
confidence: 0.85
sources: 1
entities: [e2e/global-setup.ts, e2e/global-teardown.ts, knomit-e2e, KNOMIT_HOME, .e2e-state.json, knomit serve]
motifs: [happy-path-cleanup]
refs: ['src://7b4887ce51d9/e2e/global-setup.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:83368526f6c7747f143c051d23c4cdf580666fdf', 'src://7b4887ce51d9/e2e/global-teardown.ts@18598fbc4b5cfb3d078ab104efcfdf4463364421:961648beb15fded38cf78fd076997394da0de452']
---
# An aborted or failed-in-setup e2e run leaves its knomit serve process and its knomit-e2e temp home behind; each home holds a copy of the model cache, and leaked ones filled the disk

e2e/global-setup.ts makes `$TMPDIR/knomit-e2e-*` as KNOMIT_HOME, copies the ONNX model cache into it (about 0.9 GB per home), and spawns `knomit serve` DETACHED in its own process group. It writes `.e2e-state.json` (pid, home) only at the very end, after the readiness wait, `createRepo` and `seedFixture`. e2e/global-teardown.ts kills and deletes only what that state file names.

So these leave a live server and a full home with nothing to clean them: a run killed or Ctrl-C'd before teardown; a setup that throws after the server started but before the state file is written (only the readiness-timeout path kills the child; a failing `createRepo` or seed, as the stale seed fixture did on dev before PR C, does not); a disk-full abort during the model copy. Observed 2026-10-10 during the facets build: several aborted runs by parallel workers filled the disk (37 GB, 100%) with leaked `knomit serve` processes and `knomit-e2e-*` homes, and a later run died with ENOSPC.

Consequence: after any aborted e2e run, look for `knomit serve` processes whose binary is in your tree's dist/ and for `knomit-e2e-*` directories in the temp dir, kill the former, and remove only homes nothing holds open. Do NOT sweep every `knomit-e2e-*`: another session's live run owns one, and deleting it mid-run (it happened) breaks that run's setup while the model cache is still being copied. Limit a parallel build to one e2e runner at a time.

WHAT THIS DOES NOT MEAN: a clean run is not affected; teardown removes its server and home. This is also not a fix: moving the state-file write before the seed, or wrapping setup in a try/catch that kills the child and removes the home, would make setup self-cleaning but has not been done.
