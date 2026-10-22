---
kind: pragmatic
type: policy
domain: [fact, ontology, triggers, private-paths]
confidence: 0.95
sources: 1
entities: [compileGlob, substitutePlaceholders, .knomit, isFactPath, internal/domain/fact/glob.go]
motifs: [private-state-invisible]
refs: ['src://7b4887ce51d9/internal/domain/fact/glob.go@0610c20083fcef9e84f42ab0311a14708bcb6193:3ddb2d7cb779465cd7ed582bf22a1b66b69431a3', 'src://7b4887ce51d9/internal/domain/fact/triggers.go@0610c20083fcef9e84f42ab0311a14708bcb6193:7ca350d0255f8d992ab101b0fc31d46ac853c021', 'kb://3ec012f5b4d2/kb/invariants/fact/private-paths/b6babb00.md', 'https://github.com/knomit/knomit/pull/332']
---
# A trigger `match` can never name a dot path: `.knomit/` and every private segment are refused at compile, now and in future (user ruling D-c)

`compileGlob` (internal/domain/fact/glob.go) refuses any pattern segment that begins with "." (after placeholder substitution), and `substitutePlaceholders` refuses a placeholder value beginning with ".". A trigger whose `match` names `tasks/.knomit/**` or `tasks/x/.private/*.md` is `invalid` with a warning.

User ruling, verbatim, 2026-09-27: "no, not now not in the future .knomit is for data and code only." This supersedes the F07 revision-2 design sentence that `.knomit/` paths are matchable when named explicitly, and the idea of an agent inbox under the private root. That idea had already been rejected in `.claude/future/fleet/02-rejected-alternatives.md` ("Listing private state as an inbox").

It is also consistent with the episode source: the 1b dispatcher diffs trees with F05's `isFactPath` filter, which excludes dot segments, so such a match could never fire anyway. Refusing it at compile makes the author see that, instead of a trigger that silently never fires.

MISREADING: this is about what FIRES. A changed trigger script or recipe under `.knomit/` still RELOADS its compiled program (user: "if a loaded trigger or recipe is changed in the repo, then knomit must reload that trigger / recipe goja VM"). That reload is dispatcher bookkeeping, not an episode.
