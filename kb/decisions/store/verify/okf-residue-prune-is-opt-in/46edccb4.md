---
type: observation
domain: [store, verify, okf, cli, migration]
confidence: 0.95
sources: 1
entities: [knomit verify, --prune-generated-refs, refs/heads/okf, 'okf:marker', TestVerify_NoGeneratedRefs, repoBuilder.build, fsck.Checker.PruneGeneratedRefs, internal/store/fsck/integrity.go]
motifs: [destruction-as-side-effect]
refs: ['src://7b4887ce51d9/internal/store/verify.go@4baae4ed3043cbcb37a797b625f5e8a1a071f912:03906d93d12af90dc6c822afe05032fc1d97f46c', 'src://7b4887ce51d9/cmd/verify.go@4baae4ed3043cbcb37a797b625f5e8a1a071f912:550fa8dc5108a55034cb49530de0fea0748671a7', 'kb://3ec012f5b4d2/kb/invariants/okf/source-refs-stay-local/9ba717ca.md', 'kb://3ec012f5b4d2/kb/architecture/store/fsck-child/bbc2a811.md', 'src://7b4887ce51d9/internal/store/fsck/integrity.go@8218a187df84ebf6886638d7e54ad26b72609d42:692714a0817599e2cdbaf91fd7eee9bf017312a6', 'src://7b4887ce51d9/cmd/verify.go@8218a187df84ebf6886638d7e54ad26b72609d42:f65604d866e1b3ef1e3ec490290187bad5604430']
---
# Legacy okf/* refs are pruned by an explicit verify flag, not silently at repo open — detection stays read-only

# okf/* residue is pruned on request, never implicitly

**Background.** Commit `bf9becbe` ("refactor(store): remove the server-side OKF export", 2026-07-25) deleted the code that created `refs/heads/okf/<branch>` and added `TestVerify_NoGeneratedRefs` to stop anything reintroducing generated branch refs. It removed the PRODUCER and shipped no cleanup, so every home created before that date still carries the refs plus orphan `okf:marker:*` rows in `kv`. On the live home measured 2026-08-12, three of five active repos still had them and they accounted for 6762 false verify errors.

**Options considered.**
1. Prune at repo open (`repoBuilder.build`) so homes self-heal with no operator action — rejected: it makes opening a repo delete git refs, which is a mutation on a path nobody expects to mutate, and it destroys the evidence before anyone sees a report of it.
2. Leave them and have verify skip non-maintained branches — rejected on its own: correct but leaves dead refs and dead `kv` rows on disk forever, and they keep showing up in any tool that enumerates refs.
3. **Chosen:** verify DETECTS and reports them (informational, naming each ref), and an explicit `--prune-generated-refs` flag deletes them along with the `okf:marker:*` kv rows.

**Rationale.** Keeps the default invocation read-only, which is what `verify` promises and what makes it safe to run anywhere. Removal is a separate, named, opt-in act with output saying exactly what it deleted.

**Non-scope.** This does NOT authorize a general `verify --fix`. The prune is narrowly scoped to refs the current code can never legitimately create (generated `okf/*` under refs/heads) and their markers. Any other repair — rebuilding an index, deleting ghost rows — stays out of verify. It also does not touch `refs/knomit-okf/source/*`, which is the live, deliberate location for knomit-okf source history and is outside refs/heads by design.

PATH NOTE at dev tip 8218a187 (PR #562; verified there 2026-10-20): internal/store/verify.go is now internal/store/fsck/integrity.go (the fsck child, kb/architecture/store/fsck-child/bbc2a811.md; not the commit judge internal/store/verify). Checker.PruneGeneratedRefs (integrity.go, line 436) deletes the generated okf/* refs and the `okf:marker:%` kv rows (line 477), Service.PruneGeneratedRefs in service_fsck.go is a one-line delegation to it, and cmd/verify.go still wires the `--prune-generated-refs` flag (line 117) to it. The decision is unchanged: detection stays read-only and the prune is opt-in.
