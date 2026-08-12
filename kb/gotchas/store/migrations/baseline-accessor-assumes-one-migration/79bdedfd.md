---
type: observation
domain: [store, repos, migrations, control-db, tooling]
confidence: 0.9
sources: 1
entities: [migrate.ControlBaselineSQL, applyControlDB, TestControlHasExactlyOneMigration, migrate.Control, schema_migrations, controlUp]
refs: ['src://7b4887ce51d9/internal/store/migrate/migrate.go@a6463a71', 'src://7b4887ce51d9/internal/store/migrate/control_test.go@a6463a71', 'src://7b4887ce51d9/cmd/migrate_registry.go@a6463a71', 'kb://3ec012f5b4d2/kb/decisions/repos/control-db/ddl-in-tx-not-migrator/5e08abca.md', 'kb://3ec012f5b4d2/kb/gotchas/repos/control-db/stamped-legacy-home/99478d8e.md']
---
# ControlBaselineSQL returns migration 000001 ONLY — adding a control 000002 without changing migrate-registry silently strands homes at the v1 lens shape

`migrate-registry` rebuilds control.db's lens tables by exec'ing the baseline's DDL TEXT inside its transaction, because the migrator cannot join that transaction. The accessor it uses, `migrate.ControlBaselineSQL`, hardcodes `control/000001_control_baseline.up.sql`.

That is correct only while 000001 IS the entire control chain. Once a 000002 exists the failure is silent and permanent:

1. Any `repos.OpenRegistry` touch stamps a home at the latest version — v2.
2. The operator runs `migrate-registry`. `applyControlDB` drops `lenses`/`lens_reads` and recreates them from the **v1** text.
3. The post-commit `migrate.Control` sees v2 already applied and no-ops.
4. The home now carries a v1 lens schema while `schema_migrations` reads v2. No migration will ever repair it, because every one of them is already recorded as applied.

There is no runtime error anywhere in that sequence — the mismatch only surfaces later as queries failing against columns the stamp implies exist.

**The guard is `TestControlHasExactlyOneMigration`**, which globs `control/*.up.sql` and fails the moment a second file appears. Verified by adding a probe migration: it fails with a message telling the author to go fix `applyControlDB` rather than update the expected list. When it trips, the two real options are to replay the whole `control/` chain inside migrate-registry's transaction, or to make the accessor return the chain.

**The general shape, worth recognising elsewhere:** any helper that extracts "the schema" from a migration sequence by naming ONE file has silently pinned itself to a chain length. It keeps working, and keeps being wrong, for as long as nobody adds the next migration. A tripwire test on the sequence length is cheap; noticing the bug in production is not.
