---
type: observation
domain: [store, schema, migrations]
confidence: 0.95
sources: 0
entities: [migrate.All, internal/store/migrate/migrations/, 000001_schema.up.sql, 000012_facts_origin.up.sql, 000013_drop_cluster_cache.up.sql]
refs: ['src://knomit/internal/store/migrate/migrations/000001_schema.up.sql@307b67d', 'src://knomit/internal/store/service.go@307b67d', 'src://knomit/internal/store/migrate/migrations@bba77a8']
---
# Schema migrations are numbered .up.sql/.down.sql files in internal/store/migrate/migrations/

Schema migrations live in internal/store/migrate/migrations/ as numbered NNNNNN_<name>.up.sql + .down.sql files. Run by migrate.All(db) during Service.Open AFTER registering the sqlite3_knomit driver. Current set (13 migrations, 000001–000013 as of HEAD): 000001_schema (objects, refs, kv, meta, branches, facts, branch_facts, pipeline_*, remotes, tool_*, commit_log, branch_commits, fact_entities, fact_domains), 000002_facts_vec (sqlite-vec virtual table), 000003_graphqlite (cypher graph), 000004_cluster_cache, 000005_pipeline_session_phase (adds phase column to pipeline_sessions), 000006_facts_kind (adds kind column for pragmatic vs epistemic facts), 000007_commit_parents (DAG parent-edge table), 000008_fact_domain_tokens (canonical domain-token junction), 000009_embed_dynamic_vec (dynamic embedding-dimension vec table), 000010_relocate_session_tables, 000011_commit_log_author_name (author name/email columns), 000012_facts_origin (authored|distilled|discovered origin column), 000013_drop_cluster_cache (removes the cluster_cache table — clustering moved in-process/scoped, see PR #98). NOTE: migration FILES are append-only history — 000004 created cluster_cache and 000013 later dropped it; both files remain. All migrations use CREATE TABLE IF NOT EXISTS so re-running is idempotent. Adding a new migration: pick the next number, write both .up.sql and .down.sql; migrate.All embeds them via //go:embed and runs unapplied migrations in order.
