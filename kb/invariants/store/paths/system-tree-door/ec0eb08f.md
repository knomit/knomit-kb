---
kind: pragmatic
type: policy
domain: [store, paths, templates, invariant]
confidence: 0.9
sources: 1
entities: [Service.WriteSystemTree, ErrSystemTreePath, fact.TemplatePathAllowed, batchWriteLocked, validatePath, internal/store/templates.go]
motifs: [narrowest-bypass, normalization-breaks-external-convention]
refs: ['src://7b4887ce51d9/internal/store/templates.go@80688b34ba468bd4b2ac82e3a3c1993ea6f7d596:d749ff0c0246ee1ca03eb3a1be72dacca0e17ec7', 'src://7b4887ce51d9/internal/fact/template.go@80688b34ba468bd4b2ac82e3a3c1993ea6f7d596:5f385b5b13a7511a2be89f3d4c9a22da5ffafdda', 'src://7b4887ce51d9/internal/store/templates_test.go@80688b34ba468bd4b2ac82e3a3c1993ea6f7d596:50aaba1b726aa1f831726d0ffc22717bb2a44765', 'kb://3ec012f5b4d2/kb/gotchas/store/paths/lowercased-nested-writes/f60cbf8e.md']
---
# Service.WriteSystemTree is the ONE write door that keeps the case of a nested path; it accepts only README.md and paths under .knomit/, refuses two paths differing only by case, and is never on the MCP surface

RULE: every fact write door lowercases nested paths (kb/gotchas/store/paths/lowercased-nested-writes), so a template's `.knomit/skills/x/SKILL.md` and `README.md` written through BatchWriteFacts land as `skill.md` and `readme.md`. F24 needed the template copied byte for byte, so `Service.WriteSystemTree` (internal/store/templates.go) writes WITHOUT lowercasing. Because it goes around that normalization it takes the narrowest set that works: exactly `README.md` and paths under `.knomit/` (`fact.TemplatePathAllowed`), and no two paths equal under lowercasing; anything else is `ErrSystemTreePath` before any object is written. Otherwise it is batchWrite: read-only refused, validatePath per path, `batchWriteLocked` (signer taken first, ctx trailers stamped, one commit), `notifyCommit` under the branch lock.

CONSEQUENCE FOR NEW CODE: do not widen it to 'root-level files': a root LICENSE, .gitmodules or a readme.md beside README.md is a file knomit never writes (internal/repos/manifest.go), and a case-duplicate is resolved by forges as the README. Do not expose it through MCP or REST: `.knomit/` stays closed to the fact tools (F25); the door exists for knomit's own server act of creating a repo from a template.

MISREADING: it is not a general 'exact path' write; WriteRootFile (root-only) is still the door for a root file rewrite.
