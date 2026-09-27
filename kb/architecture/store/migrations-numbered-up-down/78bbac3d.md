---
type: observation
domain: [store, schema, migrations]
confidence: 0.95
sources: 0
entities: [migrate.All, migrate.Core, migrate.Control, internal/store/migrate/repo/, internal/store/migrate/control/, 000001_schema.up.sql, 000001_control_baseline.up.sql, 000026_fact_expires.up.sql, 000011_oauth_idp.up.sql]
refs: ['src://7b4887ce51d9/internal/store/migrate/repo/000001_schema.up.sql@1cc5413c593830e521cbd7cb44bf38473c51a003:d85345916c7eb518d60486ec8ba9d2c96b08e1d6', 'src://7b4887ce51d9/internal/store/service.go@1cc5413c593830e521cbd7cb44bf38473c51a003:de881e4241c287c7480370fdfce3ace2938c250e', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@1cc5413c593830e521cbd7cb44bf38473c51a003:782b1259087d987dfb9b86d3f1de8f4162ef8d99', 'src://7b4887ce51d9/internal/store/migrate/control/000001_control_baseline.up.sql@1cc5413c593830e521cbd7cb44bf38473c51a003:885242761784968e49f88091b97fb501a1c57eda', 'kb://3ec012f5b4d2/kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md']
---
# Schema migrations are numbered .up.sql/.down.sql files in internal/store/migrate/repo/, with a sibling internal/store/migrate/control/ for control.db

Schema migrations live in two SIBLING directories under internal/store/migrate/, each holding its own numbered NNNNNN_<name>.up.sql + .down.sql files, embedded via a separate //go:embed:

- internal/store/migrate/repo/ (embedded as repoFS via `//go:embed repo/*.sql`) — the per-repo store schema. 26 migrations, 000001–000026, as of dev 1cc5413c (2026-09-27).
- internal/store/migrate/control/ (embedded as controlFS via `//go:embed control/*.sql`) — the schema for the single machine-wide <home>/control.db. 11 migrations, 000001–000011, as of the same commit.

The two sequences are INDEPENDENT: each is applied to a DIFFERENT database FILE, each keeps its own schema_migrations table, and each is numbered from 1 — repo/'s 000001 and control/'s 000001 share nothing.

Exported entry points in internal/store/migrate/migrate.go, all built on the shared newMigrator(db, fsys, dir):
- migrate.All(db) — every migration in repo/, against a repo store DB opened with the sqlite3_knomit driver (sqlite-vec loaded). Called by store.Open.
- migrate.Core(db) — only repo/'s version 1 (all base tables), against a plain "sqlite3" driver, no self-heal (a rewind on a dirty later version would migrate DOWN to 1 and destroy schema). Used by storegit.NewMemoryStorer, which only ever hands it a fresh :memory: database.
- migrate.Control(db) — every migration in control/, against <home>/control.db, opened with the stock "sqlite3" driver (control.db needs no sqlite-vec). Must run after upgradeLensSchema; repos.controlUp owns that ordering and is the only production caller.
(The former migrate.ControlSchemaSQL, the concatenated control chain for a replay tool, was removed with that tool in merge 1cc5413c.)

repo/ set (26 migrations as of 1cc5413c): 000001_schema (objects, refs, kv, meta, branches, facts, branch_facts, pipeline_*, remotes, tool_*, commit_log, branch_commits, fact_entities, fact_domains), 000002_facts_vec (sqlite-vec virtual table), 000003_graphqlite (cypher graph, extension since removed), 000004_cluster_cache, 000005_pipeline_session_phase, 000006_facts_kind (pragmatic vs epistemic), 000007_commit_parents (DAG parent-edge table), 000008_fact_domain_tokens (canonical domain-token junction), 000009_embed_dynamic_vec, 000010_relocate_session_tables (tool_*/pipeline_session tables move to the ephemeral session DB; pipeline_watermarks stays), 000011_commit_log_author_name, 000012_facts_origin (authored|distilled|discovered), 000013_drop_cluster_cache (clustering moved in-process/scoped, PR #98), 000014_graph_eav (property-graph tables owned outright once GraphQLite was gone), 000015_edges_merge_identity (MERGE semantics by UNIQUE index), 000016_per_branch_graph_schema_version, 000017_remotes_drop_connection (connection identity moved to control.db's repo_origins), 000018_abstraction_axis (title-embedding axis + restatement tables, PR #99), 000019_fact_motifs, 000020_motif_aliases, 000021_motif_definitions, 000022_motif_backfill_judged, 000023_restatement_match_kind, 000024_drop_motif_backfill_judged, 000025_experiments, 000026_fact_expires. NOTE: migration FILES are append-only history — 000004 created cluster_cache and 000013 dropped it; 000022 created motif_backfill_judged and 000024 dropped it; all files remain.

control/ set (11 migrations as of 1cc5413c): 000001_control_baseline (repos, repo_origins, lenses, lens_reads), 000002_repo_subscriptions (presence table for subscribe-mode repos), 000003_client_sessions, 000004_session_bindings, 000005_binding_handles, 000006_client_session_bindings, 000007_handle_experiments, 000008_mount_experiments, 000009_grants, 000010_oauth, 000011_oauth_idp.

Migrations are written to be safe to re-run — see kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md. The control chain is stricter: CREATE ... IF NOT EXISTS statements only, enforced by TestControl_UpMigrationsAreIdempotentDDL. Runtime-width vec0 tables (facts_vec, fact_titles_vec) are code-managed at the active model's dimension rather than fixed by a migration.

Adding a new migration: pick the next number IN THAT DIRECTORY, write both .up.sql and .down.sql; the matching entry point's //go:embed picks it up and runs unapplied migrations in order. Counts in this fact go stale with every new migration — `ls internal/store/migrate/{repo,control}` is the authority.
