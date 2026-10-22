---
type: observation
domain: [store, migrations, schema, repos, control-db]
confidence: 0.95
sources: 1
entities: [migrate.Control, migrate.All, migrate.Core, internal/engine/store/migrate/control/, internal/engine/store/migrate/repo/, registrySchema, originsSchema, lensSchema, Registry.EnsureSchema, upWithRecovery]
refs: ['kb://3ec012f5b4d2/kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md', 'kb://3ec012f5b4d2/kb/architecture/store/migrations-numbered-up-down/78bbac3d.md', 'kb://3ec012f5b4d2/kb/invariants/repos/control-db/migrator-ordering/52a6318e.md', 'src://7b4887ce51d9/internal/engine/store/migrate/migrate.go@0610c20083fcef9e84f42ab0311a14708bcb6193:782b1259087d987dfb9b86d3f1de8f4162ef8d99', 'src://7b4887ce51d9/internal/corpus/repos/registry.go@0610c20083fcef9e84f42ab0311a14708bcb6193:edf510a81971fa5cf03b9248f92cbee8e446a4cb', 'src://7b4887ce51d9/internal/corpus/repos/origins.go@0610c20083fcef9e84f42ab0311a14708bcb6193:2c94e9b5b8aaceb3b976078e89d91bf8e230ba32', 'src://7b4887ce51d9/internal/corpus/repos/lens.go@0610c20083fcef9e84f42ab0311a14708bcb6193:9ffec3d96d95eb3efba43d1b13acf816ab1d645f']
---
# control.db gets golang-migrate via an idempotent IF NOT EXISTS baseline inside internal/store/migrate — no new package, no Force-based stamping

control.db's three tenants (`repos` via `Registry.EnsureSchema`, `repo_origins` via `OpenOrigins`, `lenses`/`lens_reads` via `OpenLensRegistry`) each applied their own DDL constant with no version record. They move to golang-migrate.

**Options considered.** (a) Idempotent baseline: migration 000001 is the current DDL verbatim with every `IF NOT EXISTS` retained, so it no-ops on existing homes and creates on fresh ones. (b) Probe-and-Force: detect an existing `repos` table and `Force(1)` to stamp without running the body. (c) Clean non-idempotent baseline, fresh homes only.

**Rationale.** (a) needs no detection logic at all — the version stamp becomes self-correcting — and it is the only option that already satisfies the standing policy that every up-migration body must be safe to re-run, which `upWithRecovery` can force at any time. (b) adds a probe whose failure mode is a home stamped at a version whose schema it does not have. (c) breaks live homes.

**The choice.** `control/000001_control_baseline.up.sql` = `registrySchema` + `originsSchema` + `lensSchema` in that order (`repos` before its referents), all `IF NOT EXISTS`. Down drops children first. `Registry.EnsureSchema` is deleted; `migrate.Control(db)` replaces it. The exported `RegistrySchemaSQL` / `OriginsSchemaSQL` / `LensSchemaSQL` constants are deleted — the .sql file is now the single source those constants existed to approximate.

It lives in the EXISTING `internal/engine/store/migrate` package, not a new one: the recovery machinery is already there, and `internal/corpus/repos` already imports `internal/store` while nothing under `internal/store` imports `internal/repos`, so there is no cycle. The two migration directories are renamed `migrations/` → `repo/` and the new one is `control/`. Renaming is safe against live databases because `schema_migrations` records version integers only, never source paths. `Control` calls the same `upWithRecovery` as `All`, so it inherits issue #33 dirty recovery — which matters more here, since a permanently-dirty control.db takes down every repo at once.

**Non-scope.** This does not authorize renaming the `internal/store/migrate` package (accepted naming wart: control.db is not a store), does not touch the session database's schema mechanism, and does not change any of the 17 existing repo-store migrations beyond moving the directory.
