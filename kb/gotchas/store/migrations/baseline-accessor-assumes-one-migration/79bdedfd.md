---
type: observation
domain: [store, repos, migrations, control-db, tooling]
confidence: 0.9
sources: 2
entities: [migrate.ControlSchemaSQL, migrate.ControlBaselineSQL, applyControlDB, migrate.Control, schema_migrations, controlUp, knomit migrate-registry, 000002_repo_subscriptions, 000003_client_sessions]
motifs: [false-universal-default]
refs: ['src://7b4887ce51d9/internal/store/migrate/migrate.go@a6463a71', 'src://7b4887ce51d9/internal/store/migrate/control_test.go@a6463a71', 'src://7b4887ce51d9/cmd/migrate_registry.go@a6463a71', 'kb://3ec012f5b4d2/kb/decisions/repos/control-db/ddl-in-tx-not-migrator/5e08abca.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/control-db/stamped-legacy-home/99478d8e.md', 'src://7b4887ce51d9/cmd/migrate_registry.go@1a2d1fa1', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@1a2d1fa1', 'kb://3ec012f5b4d2/kb/decisions/repos/control-db/migrate-registry-schema-then-rows/c301b088.md', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/migrator-ordering/52a6318e.md', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@ebc15ccf4d418f23fd4ea7092d334fd14ae70a4a:59963c6f3d7470400ff310117fbef201330de33e', 'src://7b4887ce51d9/cmd/migrate_registry.go@ebc15ccf4d418f23fd4ea7092d334fd14ae70a4a:77d0342826d5b9436cc97a10f7f528c034476911', 'kb://3ec012f5b4d2/kb/decisions/store/migrate/control-chain-replayable/b0d77bc9.md']
---
# RESOLVED: the control schema accessor now returns the WHOLE chain (migrate.ControlSchemaSQL globs control/*.up.sql) — the former single-file ControlBaselineSQL would have stranded homes at the v1 lens shape once a 000002 existed

`migrate-registry` rebuilds control.db's lens tables by exec'ing the control schema's DDL TEXT inside its own transaction, because the migrator cannot join that transaction. The accessor it used, `migrate.ControlBaselineSQL`, hardcoded `control/000001_control_baseline.up.sql` — correct only while 000001 WAS the entire control chain.

**The failure that shape would have produced once a 000002 existed** (silent and permanent):
1. Any `repos.OpenRegistry` touch stamps a home at the latest version — v2.
2. The operator runs `migrate-registry`. `applyControlDB` drops `lenses`/`lens_reads` and recreates them from the **v1** text.
3. The post-commit `migrate.Control` sees v2 already applied and no-ops.
4. The home carries a v1 lens schema while `schema_migrations` reads v2, and no migration will ever repair it because every one is recorded as applied. No runtime error anywhere; the mismatch surfaces later as queries failing against columns the stamp implies exist.

**RESOLVED at HEAD (verified 2026-09-15, dev at ebc15ccf).** The chain now has THREE control migrations (000001_control_baseline, 000002_repo_subscriptions, 000003_client_sessions), and the resolution taken was the second of the two options this fact originally named: the accessor returns the chain. `migrate.ControlSchemaSQL()` (internal/store/migrate/migrate.go) globs every `control/*.up.sql` and concatenates them; `applyControlDB` (cmd/migrate_registry.go) reads it once before opening its transaction and execs the whole text, IF NOT EXISTS throughout, so tables that survived the drops are left as they are. `ControlBaselineSQL` and the `TestControlHasExactlyOneMigration` tripwire NO LONGER EXIST — do not look for them, and do not re-add a single-file accessor.

This is exactly why every control migration must stay IF NOT EXISTS-replayable (kb/decisions/store/migrate/control-chain-replayable/b0d77bc9.md): the replay applier consumes the same text as the versioned migrator.

**The general shape, still worth recognising elsewhere:** any helper that extracts "the schema" from a migration sequence by naming ONE file has silently pinned itself to a chain length. It keeps working, and keeps being wrong, for as long as nobody adds the next migration. Either make the accessor consume the sequence (done here) or add a tripwire on the sequence length.
