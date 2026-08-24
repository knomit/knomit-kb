---
type: observation
domain: [store, schema, migrations]
confidence: 0.95
sources: 0
entities: [migrate.All, migrate.Core, migrate.Control, internal/store/migrate/repo/, internal/store/migrate/control/, 000001_schema.up.sql, 000001_control_baseline.up.sql, 000017_remotes_drop_connection.up.sql]
refs: ['src://knomit/internal/store/migrate/repo/000001_schema.up.sql@307b67d', 'src://knomit/internal/store/service.go@307b67d', 'src://knomit/internal/store/migrate/repo@bba77a8', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@a51f639904872e3d4d574d708e16c0e85a58c028:df9fc147042e41d9147a478e207ee5d887625681', 'src://7b4887ce51d9/internal/store/migrate/control/000001_control_baseline.up.sql@a51f639904872e3d4d574d708e16c0e85a58c028:885242761784968e49f88091b97fb501a1c57eda']
---
# Schema migrations are numbered .up.sql/.down.sql files in internal/store/migrate/repo/, with a sibling internal/store/migrate/control/ for control.db

Schema migrations live in two SIBLING directories under internal/store/migrate/, each holding its own numbered NNNNNN_<name>.up.sql + .down.sql files, embedded via a separate //go:embed:

- internal/store/migrate/repo/ (embedded as repoFS via `//go:embed repo/*.sql`) — the per-repo store schema. 17 migrations, 000001–000017 as of HEAD.
- internal/store/migrate/control/ (embedded as controlFS via `//go:embed control/*.sql`) — the schema for the single machine-wide <home>/control.db. 1 migration, 000001_control_baseline, as of HEAD.

The two sequences are INDEPENDENT: each is applied to a DIFFERENT database FILE, each keeps its own schema_migrations table, and each is numbered from 1 — repo/'s 000001 and control/'s 000001 share nothing.

Three exported entry points in internal/store/migrate/migrate.go, all built on the shared newMigrator(db, fsys, dir):
- migrate.All(db) — every migration in repo/, against a repo store DB opened with the sqlite3_knomit driver (sqlite-vec loaded). Called by store.Open.
- migrate.Core(db) — only repo/'s version 1 (all base tables), against a plain "sqlite3" driver, no self-heal (a rewind on a dirty later version would migrate DOWN to 1 and destroy schema). Used by storegit.NewMemoryStorer and internal/git tests, which only ever hand it a fresh :memory: database.
- migrate.Control(db) — every migration in control/, against <home>/control.db, opened with the stock "sqlite3" driver (control.db needs no sqlite-vec). Must run after Manager.Start's unmigrated-home guard and upgradeLensSchema.

Current repo/ set (17 migrations, 000001–000017 as of HEAD): 000001_schema (objects, refs, kv, meta, branches, facts, branch_facts, pipeline_*, remotes, tool_*, commit_log, branch_commits, fact_entities, fact_domains), 000002_facts_vec (sqlite-vec virtual table), 000003_graphqlite (cypher graph), 000004_cluster_cache, 000005_pipeline_session_phase (adds phase column to pipeline_sessions), 000006_facts_kind (adds kind column for pragmatic vs epistemic facts), 000007_commit_parents (DAG parent-edge table), 000008_fact_domain_tokens (canonical domain-token junction), 000009_embed_dynamic_vec (dynamic embedding-dimension vec table), 000010_relocate_session_tables, 000011_commit_log_author_name (author name/email columns), 000012_facts_origin (authored|distilled|discovered origin column), 000013_drop_cluster_cache (removes the cluster_cache table — clustering moved in-process/scoped, see PR #98), 000014_graph_eav (property-graph tables owned outright now that the GraphQLite extension is gone), 000015_edges_merge_identity (MERGE semantics enforced by a UNIQUE index rather than convention), 000016_per_branch_graph_schema_version (per-branch graph_schema_version:<branch> meta rows), 000017_remotes_drop_connection (remote connection identity moved to control.db's repo_origins; only local replica STATUS stays in repo/). NOTE: migration FILES are append-only history — 000004 created cluster_cache and 000013 later dropped it; both files remain.

control/ set (1 migration as of HEAD): 000001_control_baseline creates repos, repo_origins, lenses, lens_reads.

All migrations in BOTH directories use CREATE TABLE/INDEX IF NOT EXISTS (or equivalent) so re-running is idempotent — see kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md, which now covers control/ too.

Adding a new migration: pick the next number IN THAT DIRECTORY, write both .up.sql and .down.sql; the matching entry point's //go:embed picks it up and runs unapplied migrations in order.
