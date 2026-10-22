---
type: observation
domain: [store, migrations, schema, recovery]
confidence: 0.95
sources: 0
entities: [migrate.All, migrate.Core, internal/engine/store/migrate/migrate.go, schema_migrations.dirty, migrate.Migrate.Force, store.Open, Manager.Add, internal/corpus/repos/manager.go]
refs: ['src://7b4887ce51d9/internal/engine/store/migrate/migrate.go@0610c20083fcef9e84f42ab0311a14708bcb6193:782b1259087d987dfb9b86d3f1de8f4162ef8d99', 'src://7b4887ce51d9/internal/corpus/repos/manager.go@0610c20083fcef9e84f42ab0311a14708bcb6193:85952799dd62df86b0def2737f81bbe252e42cf0', 'https://github.com/knomit/knomit/issues/33', 'kb://3ec012f5b4d2/kb/architecture/store/migrations-numbered-up-down/78bbac3d.md']
---
# An interrupted migration self-heals inside migrate.All — Force(N-1) then Up(), attempted exactly ONCE per process

golang-migrate sets `schema_migrations.dirty = 1` before a migration body and clears it after, in transactions SEPARATE from the body. A crash/SIGKILL/power loss in that span leaves `dirty = 1` forever, and `migrate.All` then fails, so `Manager.Add` drops the repo with only a WARN — the user sees their knowledge base vanish from `GET /api/v1/repos` (issue #33).

## Options considered

1. **Self-heal in `migrate.All`** — detect dirty at version N, `Force(N-1)`, re-run `Up()`, log loudly.
2. **Explicit `knomit repair --repo <name>` command** — recovery is always a deliberate operator action.
3. **Visibility only** — escalate the log to ERROR with the db path and remedy, and report the failed repo through the API in a degraded state instead of omitting it.

## Choice

Option 1, plus the actionable-log half of option 3. Recovery is automatic and requires no operator knowledge; the log line at the skip site names the database file so the residual failure is still diagnosable.

## Rationale

Self-healing is safe here specifically because the migration BODY is transactional: an interruption leaves the schema either fully applied or fully unapplied, never torn. Combined with knomit's idempotent migrations (`IF NOT EXISTS` / `INSERT OR IGNORE`), re-running version N cannot corrupt anything. Option 2 was rejected as leaving the repo invisible until an operator who does not know the command intervenes. Option 3's API half was rejected as a breaking response-shape change (`[]string` → objects) for a failure mode that option 1 removes.

## Non-scope — the guard, and why it exists

This decision does NOT authorize retrying a migration until it succeeds. Self-heal covers INTERRUPTED migrations, not migrations that fail DETERMINISTICALLY on the data (e.g. a `CREATE UNIQUE INDEX` that the existing rows violate). Those re-fail identically on retry, so an unguarded loop would clear `dirty`, re-fail, and re-dirty on every boot — converting a one-time manual recovery into a permanent one. Idempotency does not help: it makes re-execution safe, it does not make the data satisfy a new constraint. Therefore recovery is attempted EXACTLY ONCE per call, and on failure `migrate.All` reports the UNDERLYING migration error (the actionable one), leaving `dirty` set.

Corollary for migration authors: a migration that can fail on pre-existing data must repair that data itself (as `000015_edges_merge_identity` does by deduping before indexing) rather than relying on this recovery machinery to retry it.

Regression tests: `TestMigrate_RecoversFromDirtyVersion`, `TestMigrate_DoesNotLoopOnDeterministicFailure`.
