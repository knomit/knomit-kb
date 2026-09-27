---
type: observation
domain: [repos, control-db, cli, operations, decisions]
confidence: 0.95
sources: 1
entities: [migrate-registry, migrateRegistryCmd, refuseUnmigratedHome, anyRepoDBFile, HasLegacyLensSchema, upgradeLensSchema, migrate.ControlSchemaSQL, running.marker, --ignore-running-marker, cmd/migrate_registry.go, cmd/root.go, internal/repos/manager.go, internal/store/migrate/migrate.go]
motifs: [dead-migration-path-removal, simplify-by-deleting]
refs: ['src://7b4887ce51d9/cmd/migrate_registry.go@ff8fc29a748b6e0f10672b6e4ba4f86167a5a2d6:09540d63ffd656c573c52a7926893c5fecd11b77', 'src://7b4887ce51d9/cmd/root.go@ff8fc29a748b6e0f10672b6e4ba4f86167a5a2d6:8b5760480e264e0b8e2875b3539d2c94e1d8e05d#L27', 'src://7b4887ce51d9/internal/repos/manager.go@ff8fc29a748b6e0f10672b6e4ba4f86167a5a2d6:0115ba0d1c1052b35bd271feae68ddf4f137b73d#L1014-L1060', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@ff8fc29a748b6e0f10672b6e4ba4f86167a5a2d6:59963c6f3d7470400ff310117fbef201330de33e#L82-L114', 'kb://3ec012f5b4d2/kb/gotchas/repos/migrate-registry/running-marker-is-not-a-lock/e699ca0c.md', 'kb://3ec012f5b4d2/kb/invariants/repos/migrate-registry-ordering/4c3cd3e6.md', 'kb://3ec012f5b4d2/kb/invariants/repos/lens-registry-no-auto-migrate/9ca80457.md', 'kb://3ec012f5b4d2/kb/architecture/store/migrations/ec215752.md', 'https://github.com/knomit/knomit/issues/326']
---
# `knomit migrate-registry` is REMOVED outright together with the legacy-home boot detection (refuseUnmigratedHome / anyRepoDBFile) — no replacement guidance, no version pointer, because every home in use has already been migrated; running.marker and upgradeLensSchema stay

**Context (2026-09-27).** `knomit migrate-registry` converted a home that predates the control.db repo registry. It had carried a deprecation notice ("will be REMOVED in a future build") and the server's boot path (`refuseUnmigratedHome`, manager.go L1014-L1060 at ff8fc29a, called from Manager.Start) refused to open a pre-registry home and told the operator to run it. The designer asked for simplification. This decision is about REMOVING that machinery; the bring-up ordering inside controlUp (migrator-ordering) and the lens-schema self-upgrade (lens-registry-no-auto-migrate) are separate facts that this decision only narrows (the "pre-registry shape is reachable only via migrate-registry" clause becomes "not reachable at all"). The "two appliers" synthesis (ec215752) loses one applier: after this, control.db's schema text has ONE applier, the versioned migrator, and the IF-NOT-EXISTS property rests on recovery re-execution alone.

**Options considered.**
1. Remove the command; keep the boot detection; rewrite its refusal into recovery guidance (name the last release that ships the converter, v0.5.3; tell the operator to run it on a copy of the home, or to start fresh and re-add each knowledge base, with a `sqlite3 … SELECT name, url, branch FROM remotes` query to recover the URLs; warn that opening a legacy repo .db with the new build drops those columns). Proposed by the worker's inventory.
2. Remove the command; keep the detection with a one-line refusal ("this home predates the control.db repo registry; start from a fresh home").
3. Remove the command AND the detection. A pre-registry home would boot with its old `repos/<name>.db` files silently ignored, as if it held no repos.

**Rationale.** The designer stated that EVERY home in use has already been through the migration, so the pre-registry shape exists nowhere; code that recognises it, and any message about it, is dead weight. Option 1 is an error message that describes a world nobody is in. Option 2 keeps detection code for a shape that cannot occur. The designer chose 3 explicitly: "get rid of the whole thing".

**The choice.** Option 3. Delete: `cmd/migrate_registry.go` and its tests (incl. `refuseIfServerRunning`, `--ignore-running-marker`, the deprecation notice); the `AddCommand` line in cmd/root.go (L27); `refuseUnmigratedHome`, `anyRepoDBFile`, their call site and the tests asserting the refusal text; `migrate.ControlSchemaSQL` (L82-L114) and its test, whose only production caller was the command. Reword comments that name the command; leave applied .sql migration files untouched, comments included.

**What STAYS, and why.** `<home>/running.marker` stays: its role is the unclean-exit warning at the next `serve` boot, unrelated to the command (see running-marker-is-not-a-lock). `upgradeLensSchema` stays: it heals the lens-schema shape of homes that DID run the command. `HasLegacyLensSchema` stays because registry.go still calls it (L122 at ff8fc29a). `TestControl_UpMigrationsAreIdempotentDDL` keeps its CREATE-IF-NOT-EXISTS-only rule with its reason reworded to recovery re-execution; relaxing that rule is a separate decision, NOT bundled.

**Non-scope.** This does not authorize removing other deprecated commands, changing control.db migrations, or relaxing the idempotent-DDL rule. Nothing in the repo may hard-code v0.5.3 as a pointer.

**Consequence for the corpus.** Facts about the command's internals (migrate-registry-ordering, running-marker's migrate-registry section, ksuid-shaped-repo-name guard, schema-then-rows, the RESOLVED baseline-accessor gotcha) describe code that no longer exists after the merge and are retracted or updated in the PR's experiment close-out; ec215752 is updated to one applier.

**Misreading to avoid.** "A legacy home now boots silently, so add a warning" re-creates option 2. There are no legacy homes; if one ever appears it is a bug in whatever produced it, not a supported state.
