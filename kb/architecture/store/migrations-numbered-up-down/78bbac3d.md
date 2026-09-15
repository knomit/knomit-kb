---
type: observation
domain: [store, schema, migrations]
confidence: 0.95
sources: 0
entities: [migrate.All, migrate.Core, migrate.Control, migrate.ControlSchemaSQL, internal/store/migrate/repo/, internal/store/migrate/control/, 000001_schema.up.sql, 000001_control_baseline.up.sql, 000024_drop_motif_backfill_judged.up.sql, 000003_client_sessions.up.sql]
refs: ['src://knomit/internal/store/migrate/repo/000001_schema.up.sql@307b67d', 'src://knomit/internal/store/service.go@307b67d', 'src://knomit/internal/store/migrate/repo@bba77a8', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@a51f639904872e3d4d574d708e16c0e85a58c028:df9fc147042e41d9147a478e207ee5d887625681', 'src://7b4887ce51d9/internal/store/migrate/control/000001_control_baseline.up.sql@a51f639904872e3d4d574d708e16c0e85a58c028:885242761784968e49f88091b97fb501a1c57eda', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@ebc15ccf4d418f23fd4ea7092d334fd14ae70a4a:59963c6f3d7470400ff310117fbef201330de33e']
---
# Schema migrations are numbered .up.sql/.down.sql files in internal/store/migrate/repo/, with a sibling internal/store/migrate/control/ for control.db

Schema migrations live in two SIBLING directories under internal/store/migrate/, each holding its own numbered NNNNNN_<name>.up.sql + .down.sql files, embedded via a separate //go:embed:

- internal/store/migrate/repo/ (embedded as repoFS via `//go:embed repo/*.sql`) — the per-repo store schema. 24 migrations, 000001–000024, as of dev ebc15ccf (2026-09-15).
- internal/store/migrate/control/ (embedded as controlFS via `//go:embed control/*.sql`) — the schema for the single machine-wide <home>/control.db. 3 migrations, 000001–000003, as of the same commit.

The two sequences are INDEPENDENT: each is applied to a DIFFERENT database FILE, each keeps its own schema_migrations table, and each is numbered from 1 — repo/'s 000001 and control/'s 000001 share nothing.

Exported entry points in internal/store/migrate/migrate.go, all built on the shared newMigrator(db, fsys, dir):
- migrate.All(db) — every migration in repo/, against a repo store DB opened with the sqlite3_knomit driver (sqlite-vec loaded). Called by store.Open.
- migrate.Core(db) — only repo/'s version 1 (all base tables), against a plain "sqlite3" driver, no self-heal (a rewind on a dirty later version would migrate DOWN to 1 and destroy schema). Used by storegit.NewMemoryStorer, which only ever hands it a fresh :memory: database.
- migrate.Control(db) — every migration in control/, against <home>/control.db, opened with the stock "sqlite3" driver (control.db needs no sqlite-vec). Must run after Manager.Start's unmigrated-home guard and upgradeLensSchema (via controlUp).
- migrate.ControlSchemaSQL() — the concatenated text of EVERY control/*.up.sql, consumed by `knomit migrate-registry`'s in-transaction replay; it is why every control migration must be IF NOT EXISTS-replayable.

repo/ set (24 migrations as of HEAD): 000001_schema (objects, refs, kv, meta, branches, facts, branch_facts, pipeline_*, remotes, tool_*, commit_log, branch_commits, fact_entities, fact_domains), 000002_facts_vec (sqlite-vec virtual table), 000003_graphqlite (cypher graph, extension since removed), 000004_cluster_cache, 000005_pipeline_session_phase, 000006_facts_kind (pragmatic vs epistemic), 000007_commit_parents (DAG parent-edge table), 000008_fact_domain_tokens (canonical domain-token junction), 000009_embed_dynamic_vec, 000010_relocate_session_tables (tool_*/pipeline_session tables move to the ephemeral session DB; pipeline_watermarks stays), 000011_commit_log_author_name, 000012_facts_origin (authored|distilled|discovered), 000013_drop_cluster_cache (clustering moved in-process/scoped, PR #98), 000014_graph_eav (property-graph tables owned outright once GraphQLite was gone), 000015_edges_merge_identity (MERGE semantics by UNIQUE index), 000016_per_branch_graph_schema_version, 000017_remotes_drop_connection (connection identity moved to control.db's repo_origins), 000018_abstraction_axis (title-embedding axis + restatement tables, PR #99), 000019_fact_motifs, 000020_motif_aliases, 000021_motif_definitions, 000022_motif_backfill_judged, 000023_restatement_match_kind, 000024_drop_motif_backfill_judged. NOTE: migration FILES are append-only history — 000004 created cluster_cache and 000013 dropped it; 000022 created motif_backfill_judged and 000024 dropped it; all files remain.

control/ set (3 migrations as of HEAD): 000001_control_baseline (repos, repo_origins, lenses, lens_reads), 000002_repo_subscriptions (presence table for subscribe-mode repos), 000003_client_sessions.

All migrations in BOTH directories use CREATE TABLE/INDEX IF NOT EXISTS (or equivalent) so re-running is idempotent — see kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md, which covers control/ too. Runtime-width vec0 tables (facts_vec, fact_titles_vec) are code-managed at the active model's dimension rather than fixed by a migration.

Adding a new migration: pick the next number IN THAT DIRECTORY, write both .up.sql and .down.sql; the matching entry point's //go:embed picks it up and runs unapplied migrations in order. Counts in this fact go stale with every new migration — `ls internal/store/migrate/{repo,control}` is the authority.
