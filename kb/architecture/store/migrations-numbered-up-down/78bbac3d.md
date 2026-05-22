---
type: observation
domain: [store, schema, migrations]
confidence: 0.95
sources: 0
entities: [migrate.All, internal/store/migrate/migrations/, 000001_schema.up.sql, 000006_facts_kind.up.sql]
refs: ['src://knomit/internal/store/migrate/migrations/000001_schema.up.sql@307b67d', 'src://knomit/internal/store/service.go@307b67d']
---
# Schema migrations are numbered .up.sql/.down.sql files in internal/store/migrate/migrations/

Schema migrations live in internal/store/migrate/migrations/ as numbered NNNNNN_<name>.up.sql + .down.sql files. Run by migrate.All(db) during Service.Open AFTER registering the sqlite3_knomit driver. Current set (commit 307b67d): 000001_schema (objects, refs, kv, meta, branches, facts, branch_facts, pipeline_*, remotes, tool_*, commit_log, branch_commits, fact_entities, fact_domains), 000002_facts_vec (sqlite-vec virtual table), 000003_graphqlite (cypher graph), 000004_cluster_cache, 000005_pipeline_session_phase (adds phase column to pipeline_sessions), 000006_facts_kind (adds kind column for pragmatic vs epistemic facts). All migrations use CREATE TABLE IF NOT EXISTS so re-running is idempotent. Adding a new migration: pick the next number, write both .up.sql and .down.sql; migrate.All embeds them via //go:embed and runs unapplied migrations in order.
