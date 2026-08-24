---
type: observation
domain: [repos, migrations, control-db, tooling, atomicity]
confidence: 0.95
sources: 1
entities: [applyControlDB, migrate.ControlBaselineSQL, migrate.Control, knomit migrate-registry, schema_migrations, sqlite3.WithInstance, NoTxWrap, upWithRecovery, planLenses, detectLegacyControlDB]
refs: ['src://7b4887ce51d9/cmd/migrate_registry.go@94e763c3', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@94e763c3', 'kb://3ec012f5b4d2/kb/decisions/repos/control-db/migrate-registry-schema-then-rows/c301b088.md', 'kb://3ec012f5b4d2/kb/decisions/store/migrations/control-db-golang-migrate/c06bcaae.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/control-db/stamped-legacy-home/99478d8e.md']
---
# migrate-registry runs the baseline DDL TEXT inside its transaction and the migrator only stamps afterwards — reversing the earlier schema-then-rows split, which silently destroyed lens rows on abort

This REVERSES the earlier schema-then-rows decision. That one moved schema creation out of `applyControlDB`'s transaction on the premise that golang-migrate "cannot nest inside a caller's transaction", and accepted the abort semantics as merely "empty-but-correct tables, still covered by control.db.bak". That acceptance was wrong, and a code review caught it.

**The bug it caused.** The legacy `DROP TABLE lens_reads/lenses` moved from `tx.Exec` to a bare `db.Exec`, so it committed immediately and the migrator recreated the tables EMPTY. If the row transaction then failed, the rollback restored `repos`/`repo_settings` but could not restore the lens rows. Worse, the re-run completed SILENTLY: `detectLegacyControlDB` still returned true via the surviving `repo_settings`, and `planLenses` read the empty new-shape `lenses` and planned nothing. The operator saw "migration complete" having lost every lens definition. `control.db.bak` existed but nothing restored from it and nothing told them to.

**What the constraint actually is.** `sqlite3.WithInstance` takes a `*sql.DB`, never a `*sql.Tx`, so the migrator cannot be handed a caller's transaction. With `SetMaxOpenConns(1)` — which control.db uses — calling it while holding an open `*sql.Tx` does not merely fail to nest, it DEADLOCKS on the single checked-out connection. `Config.NoTxWrap` plus raw `BEGIN`/`COMMIT` through `db.Exec` would technically work, but `NoTxWrap` removes the per-migration transaction that `upWithRecovery`'s correctness argument rests on, so it is not available for free.

**The resolution.** The MIGRATOR does not need to be in the transaction — only the DDL does, and the baseline DDL is idempotent by construction. So `applyControlDB` now runs, in one transaction: drop the legacy lens tables, execute `migrate.ControlBaselineSQL` (IF NOT EXISTS throughout, so it recreates only the missing lens tables and no-ops on `repos`/`repo_origins`), then every row insert. `migrate.Control(db)` runs AFTER the commit, purely to stamp.

**What this buys beyond fixing the bug.** The `DROP TABLE IF EXISTS schema_migrations` is no longer needed at all — the transaction recreates the lens tables itself, so the stamp is irrelevant. That removes the "unconditional drop rewinds every future migration" hazard and the whole stamped-legacy-home failure mode with it. `ControlBaselineSQL` is an accessor over the embedded `.sql`, not a second copy, so the single-source-of-DDL property the deleted `*SchemaSQL` constants failed to provide is kept.

**Non-scope.** This does not reintroduce exported per-tenant DDL constants, and it does not make `NoTxWrap` available anywhere. The boot path is unchanged: `Manager.Start` still runs guard → re-key → `migrate.Control` with no outer transaction.
