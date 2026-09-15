---
type: synthesis
domain: [store, verify, index, branches, cli]
confidence: 0.92
sources: 1
evidence_weight: 0.7402597402597403
entities: [Service.Verify, listBranchRefsForVerify, repoBuilder.setupIndex, meta.last_commit, meta.graph_schema_version, --prune-generated-refs, refs/heads/okf, IntegrityReport]
motifs: [intent-misread-as-defect, signal-cannot-distinguish-cases]
refs: ['kb://3ec012f5b4d2/kb/decisions/store/verify/branch-scope-follows-the-indexer/e2a775c8.md', 'kb://3ec012f5b4d2/kb/architecture/store/index/maintained-branch-oracle/2418fbec.md', 'kb://3ec012f5b4d2/kb/decisions/store/verify/okf-residue-prune-is-opt-in/46edccb4.md']
---
# Verify's scope is derived from the indexer's own durable record (meta.last_commit:<branch>), not from the wider enumerable set (refs/heads/*): parity is an ERROR only where the index promised it, everything in the gap is reported once as informational, and removing residue is a named opt-in flag — never a side effect of detection

An integrity tool that asserts an invariant over a set wider than the set the system actually maintains reports every correctly-unmaintained member as a failure, so its scope must be derived from the maintainer's own durable record rather than from whatever is enumerable; the members in the gap are named once as information, and any removal of residue is a separate, explicit act rather than a consequence of looking.

How the three pieces fit:
- THE SCOPE FOLLOWS THE INDEXER. `listBranchRefsForVerify` used to enumerate every `refs/heads/*` and demand full SQLite parity for each, while `repoBuilder.setupIndex` maintains only agentBranch plus upstreamMain. On a live home every one of thousands of issues was on an unindexed branch (legacy `okf/*` export residue and another machine's agent branch arriving by fetch), exit code 1 permanently, zero issues on any indexed branch. Widening the indexer was rejected (the two-branch scope is deliberate and foreign agent branches would make index work unbounded); special-casing `okf/*` was rejected (the foreign-agent class does not age out). Parity checks now run only for maintained branches; every other ref is listed once, named, informational.
- THE ORACLE IS `meta.last_commit:<branch>`. The index writes it exactly when it syncs a branch, so the row exists precisely for branches it has indexed; nothing else records that set — setupIndex computes it without persisting, the `branches` table records branches the store merely KNOWS about, and refs/heads is wider still. `meta.graph_schema_version:<branch>` looks like the same signal and is NOT: migration 000016 backfilled it onto every branch known at the time, so an unindexed branch can carry one. Testing it would report unindexed branches as indexed.
- REMOVAL IS OPT-IN. Generated `refs/heads/okf/*` and their `okf:marker:*` kv rows are residue of a removed producer that shipped no cleanup. Pruning them at repo open was rejected because it makes opening a repo delete git refs and destroys the evidence before any report; verify DETECTS and names them, and only `--prune-generated-refs` deletes them, printing what it removed. The default invocation stays read-only, which is what makes verify safe to run anywhere.

WHAT THIS DOES NOT MEAN: unindexed branches are not thereby healthy or uninteresting — their parity is simply not an ERROR; a maintained branch showing drift is still an ERROR, the `branches`-table check still fires in both directions for maintained branches, and a `branches` row with no git ref remains an ERROR. The prune does NOT authorize a general `verify --fix`: it is scoped to refs the current code can never legitimately create, and it does not touch `refs/knomit-okf/source/*`, which lives outside refs/heads by design.
