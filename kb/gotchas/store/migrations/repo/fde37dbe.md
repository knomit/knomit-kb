---
type: observation
domain: [store, sqlite, migrations]
confidence: 0.9
sources: 1
entities: [000031_trigger_recipe.up.sql, upWithRecovery, trigger_fires, facts_vec, TestMigration000031_ReRunsAndKeepsTheLog]
motifs: [rebuild-not-alter, schema-wide-revalidation]
refs: ['src://7b4887ce51d9/internal/store/migrate/repo/000031_trigger_recipe.up.sql@476c8037d3d2c905aab5a6243584edf5d02911ce:de1b6986697a032a21ceb97800c2c6e0d8a7fe14', 'src://7b4887ce51d9/internal/store/triggers_recipe_test.go@476c8037d3d2c905aab5a6243584edf5d02911ce:a993d80481950b4a022353a73b2f6fcd0de811bc', 'kb://3ec012f5b4d2/kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md', 'https://github.com/knomit/knomit/pull/346']
---
# A repo migration cannot ALTER TABLE ... RENAME (SQLite re-parses the whole schema and fails on the facts triggers that name the sqlite-vec virtual table: "no such table: main.facts_vec") and cannot ADD COLUMN (upWithRecovery re-runs a body); to add columns, copy the table OUT with CREATE TABLE … AS SELECT, DROP, recreate, copy BACK, drop the copy — 000031 is the template

Hit while adding recipe_source/recipe_rev/run_id to trigger_fires (F07 PR 5). The first draft (CREATE trigger_fires_next; INSERT … SELECT; DROP trigger_fires; ALTER TABLE trigger_fires_next RENAME TO trigger_fires) failed every store open with `migrate.All: error in trigger facts_after_delete: no such table: main.facts_vec` — the migration connection does not load sqlite-vec, and modern SQLite's RENAME re-validates every trigger in the schema. ADD COLUMN is ruled out by the idempotent-up-bodies convention (a second duplicate-column collision on a rewound head database is a hard error).

Working shape (internal/store/migrate/repo/000031_trigger_recipe.up.sql): DROP TABLE IF EXISTS <copy>; CREATE TABLE <copy> AS SELECT <old columns> FROM t; DROP TABLE t; CREATE TABLE t (<new schema>); INSERT INTO t (<old columns>) SELECT <old columns> FROM <copy>; DROP TABLE <copy>; CREATE INDEX IF NOT EXISTS …. It re-runs over an applied schema, keeps ids (AUTOINCREMENT continues after them) and survives down+up — pinned by TestMigration000031_ReRunsAndKeepsTheLog, which executes the up/down files directly. Only viable for a table no trigger or view references (trigger_fires has none); preserves data, unlike 000023's DROP+CREATE of a derived cache.
