---
type: observation
domain: [store, sqlite, migrations]
confidence: 0.9
sources: 1
entities: [000031_trigger_recipe.up.sql, upWithRecovery, trigger_fires, facts_vec, TestMigration000031_ReRunsAndKeepsTheLog]
motifs: [rebuild-not-alter, schema-wide-revalidation]
refs: ['src://7b4887ce51d9/internal/engine/store/migrate/repo/000031_trigger_recipe.up.sql@0610c20083fcef9e84f42ab0311a14708bcb6193:c0a6428c5436b0191370332ef89cf038bdcc0814', 'src://7b4887ce51d9/internal/engine/store/triggers_recipe_test.go@0610c20083fcef9e84f42ab0311a14708bcb6193:4d5dcc052edc9b96e9a4a29e22d081ba57654e3a', 'kb://3ec012f5b4d2/kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md', 'https://github.com/knomit/knomit/pull/346']
---
# A repo migration cannot ALTER TABLE ... RENAME (SQLite re-parses the whole schema and fails on the facts triggers that name the sqlite-vec virtual table: "no such table: main.facts_vec") and cannot ADD COLUMN (upWithRecovery re-runs a body); to add columns, copy the table OUT with CREATE TABLE … AS SELECT, DROP, recreate, copy BACK, drop the copy — 000031 is the template

Hit while adding recipe_source/recipe_rev/run_id to trigger_fires (F07 PR 5). The first draft (CREATE trigger_fires_next; INSERT … SELECT; DROP trigger_fires; ALTER TABLE trigger_fires_next RENAME TO trigger_fires) failed every store open with `migrate.All: error in trigger facts_after_delete: no such table: main.facts_vec` — the migration connection does not load sqlite-vec, and modern SQLite's RENAME re-validates every trigger in the schema. ADD COLUMN is ruled out by the idempotent-up-bodies convention (a second duplicate-column collision on a rewound head database is a hard error).

Working shape (internal/engine/store/migrate/repo/000031_trigger_recipe.up.sql): DROP TABLE IF EXISTS <copy>; CREATE TABLE <copy> AS SELECT <old columns> FROM t; DROP TABLE t; CREATE TABLE t (<new schema>); INSERT INTO t (<old columns>) SELECT <old columns> FROM <copy>; DROP TABLE <copy>; CREATE INDEX IF NOT EXISTS …. It re-runs over an applied schema, keeps ids (AUTOINCREMENT continues after them) and survives down+up — pinned by TestMigration000031_ReRunsAndKeepsTheLog, which executes the up/down files directly. Only viable for a table no trigger or view references (trigger_fires has none); preserves data, unlike 000023's DROP+CREATE of a derived cache.
