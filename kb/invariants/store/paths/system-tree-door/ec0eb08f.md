---
kind: pragmatic
type: policy
domain: [store, paths, templates, invariant]
confidence: 0.9
sources: 1
entities: [Service.WriteSystemTree, ErrSystemTreePath, fact.TemplatePathAllowed, batchWriteLocked, validatePath, internal/store/templates.go]
motifs: [narrowest-bypass, normalization-breaks-external-convention]
refs: ['kb://3ec012f5b4d2/kb/gotchas/store/paths/lowercased-nested-writes/f60cbf8e.md', 'src://7b4887ce51d9/internal/store/templates.go@7833068a79a87d8f6c369132a6d411290d7ee837:d749ff0c0246ee1ca03eb3a1be72dacca0e17ec7', 'src://7b4887ce51d9/internal/fact/template.go@97e735eef592629d64226d23227b04f579ae392f:5f385b5b13a7511a2be89f3d4c9a22da5ffafdda', 'src://7b4887ce51d9/internal/store/templates_test.go@7833068a79a87d8f6c369132a6d411290d7ee837:97b8952f07cd93f6167eae0b29040f6e480833d6', 'src://7b4887ce51d9/internal/store/fact_write_readonly_test.go@7833068a79a87d8f6c369132a6d411290d7ee837:f70890717385df2b7570f9d0855315eb1903eca1', 'https://github.com/knomit/knomit/pull/433', 'https://github.com/knomit/knomit/pull/435']
---
# Service.WriteSystemTree is the ONE write door that keeps the case of a nested path; it accepts only README.md and paths under .knomit/, refuses two paths differing only by case, and is never on the MCP surface

RULE: every fact write door lowercases nested paths (kb/gotchas/store/paths/lowercased-nested-writes), so a template's `.knomit/skills/x/SKILL.md` and `README.md` written through BatchWriteFacts land as `skill.md` and `readme.md`. F24 needed the template copied byte for byte, so `Service.WriteSystemTree` (internal/store/templates.go) writes WITHOUT lowercasing. Because it goes around that normalization it takes the narrowest set that works: exactly `README.md` and paths under `.knomit/` (`fact.TemplatePathAllowed`), and no two paths equal under lowercasing; anything else is `ErrSystemTreePath` before any object is written. Otherwise it is batchWrite: read-only refused (by its OWN gate: it calls batchWriteLocked directly, below the shared writeFileExact/deleteFile/batchWrite seam, so that copy is the only thing keeping a subscription's store from committing through this door; pinned in TestReadOnlyStore_RefusesAuthoredWritesOnEveryBranch, CONSEQUENCE: never delete it as 'unreachable through create'), validatePath per path, `batchWriteLocked` (signer taken first, ctx trailers stamped, one commit), `notifyCommit` under the branch lock.

CONSEQUENCE FOR NEW CODE: do not widen it to 'root-level files': a root LICENSE, .gitmodules or a readme.md beside README.md is a file knomit never writes (internal/repos/manifest.go), and a case-duplicate is resolved by forges as the README. Do not expose it through MCP or REST: `.knomit/` stays closed to the fact tools (F25); the door exists for knomit's own server act of creating a repo from a template.

MISREADING: it is not a general 'exact path' write; WriteRootFile (root-only) is still the door for a root file rewrite.
