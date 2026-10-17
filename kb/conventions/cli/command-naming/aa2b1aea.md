---
kind: pragmatic
type: heuristic
domain: [cli, verify, naming]
confidence: 0.85
sources: 1
entities: [knomit verify ci, knomit verify merge, knomit verify branch]
refs: ['kb://3ec012f5b4d2/kb/gotchas/verify/forge-ci/main-moves-after-verify/92c35626.md', 'src://7b4887ce51d9/tools/ci/verify-signatures.yml@6f241959415647560a58bc14f2f8324369382f6f:ddf2ec126d5bc00ad14906101d7eeb66dc150e8d']
---
# User naming note: name the CI verification gate for what it does, not where it runs; `knomit verify ci` is the wrong name and should be renamed (e.g. `knomit verify merge` or `knomit verify branch`) when the command is next touched

USER, verbatim (2026-10-17): "`knomit verify ci` (which is a stupid command line, why "ci"?)".

RULE: a command is named for what it does, not for where it runs. `ci` names a venue (the workflow runner); the command verifies a candidate branch's merge into main. Candidate names, not decided: `knomit verify merge` (also the name of the postponed atomic verify-and-merge command in kb/gotchas/verify/forge-ci/main-moves-after-verify/92c35626.md) or `knomit verify branch`.

STATUS: not renamed. The user asked for the rename to happen when the command is next touched, not as a task of its own. No decision yet between the candidate names, and the postponed `knomit verify merge` is a different, atomic command: do not assume the rename target is that command.

CONSEQUENCE: when touching `verify ci` (flags, behaviour, the template workflow tools/ci/verify-signatures.yml), rename it in the same change, keep the old name as an alias if released users could rely on it, and ask the user which name. New commands follow the same rule: no venue names (ci, cron, hook) in a command name.

MISREADING: this is not a request to rename now, and not permission to choose the new name unasked.
