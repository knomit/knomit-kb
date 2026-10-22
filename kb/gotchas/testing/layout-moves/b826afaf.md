---
type: observation
domain: [testing, archtest, refactoring, layout]
confidence: 0.9
sources: 1
entities: [filepath.Join, modroot.Path, test/archtest, ref_consumers_test.go, motif_surfaces_test.go, restatement_conformance_test.go]
motifs: [rewrite-misses-split-form]
refs: ['kb://3ec012f5b4d2/kb/conventions/testing/cross-package-scans-from-module-root/a8fe58c7.md', 'src://7b4887ce51d9/test/archtest/store_no_embed_test.go@fb6cf903339656fa0629a7f1da4c20f5ee825484:c4fe778712ec05bdc7c9c3e32f5ca05fe1b491a5', 'src://7b4887ce51d9/internal/domain/fact/ref_consumers_test.go@fb6cf903339656fa0629a7f1da4c20f5ee825484:3c802955432c319debce725ff9b900c0c3b1e5c2', 'src://7b4887ce51d9/internal/mcp/motif_surfaces_test.go@fb6cf903339656fa0629a7f1da4c20f5ee825484:4f50e49b533eb252ade83a2257886a016b52e480', 'src://7b4887ce51d9/internal/synthesize/restatement_conformance_test.go@fb6cf903339656fa0629a7f1da4c20f5ee825484:8083555306e3a20f60324f5daa9d738876326a22', 'src://7b4887ce51d9/internal/domain/fact/ref_test.go@1ac3a7421fe33ddde51c6de8cb19c3d0423500f2:6a36ff0cac7e6a6d13baec51e4c34feba3db38c2', 'src://7b4887ce51d9/tools/bridge/claude/hook_post_edit_test.go@1ac3a7421fe33ddde51c6de8cb19c3d0423500f2:f095af729d5ee1d2973e4dab9201d9f06a6d88f2', 'https://github.com/knomit/knomit/pull/591']
---
# Archtest and test path guards that split a path into segments (filepath.Join("internal", "store"), modroot.Path(t, ...)) are missed by a text rewrite of "internal/store"

A bulk `internal/store` to `internal/engine/store` rewrite leaves behind guards that build the path from segments: `filepath.Join("internal", "store", ...)` and `modroot.Path(t, "internal", "store")`. Found in step 03 (PR #590): test/archtest (store_write_internals, store_reexports, store_no_embed, test_signer_isolation), internal/domain/fact/ref_consumers_test.go (8 allowlist keys), internal/mcp/motif_surfaces_test.go, internal/synthesize/restatement_conformance_test.go.

Most fail loudly (stale-allowlist-key, missing dir): that is the value of the 'key never walked' assertion. A scan whose root is missing narrows silently only if it lacks a minimum-files floor.

CHECK with `git grep -nE '"internal",[[:space:]]*"<pkg>"'` (git grep ERE has no \s).

CONVERSELY, the rewrite must SKIP data: src:// refs and fixture strings used as classifier input (ref_test.go wants, okf golden refs, hook_post_edit fixtures) are not layout references and broke tests when rewritten.

STEP 04A PRACTICE (PR #591, 1ac3a742): the corpus/edge move rewrote `internal/<pkg>` mentions in comments, CI files and guard keys with a lookbehind-bounded regex, and deliberately skipped data used as classifier input: internal/domain/fact/ref_test.go and ref_validation_test.go (src:// ref fixtures) and tools/bridge/claude/hook_post_edit_test.go (path-normalisation fixtures). src:// refs inside .claude/skills and bridge templates are historical and were left untouched. This does not mean comment mentions are optional: they are rewritten with the move; only fixtures and historical refs are not.
