---
type: observation
domain: [store, migrations, control-db, recovery]
confidence: 0.9
sources: 1
entities: [TestControl_UpMigrationsAreIdempotentDDL, migrate.Control, upWithRecovery, recoverDirty, forcePastCommittedBody, alreadyApplied, migrate.ControlSchemaSQL, knomit migrate-registry]
motifs: [replayability-constrains-schema, rationale-outlives-its-cause]
refs: ['src://7b4887ce51d9/internal/store/migrate/control_test.go@49e396887c1888e3c538969847219bc20fa5e9eb:eec52544d282c98c825bb076afbedbd099d9b410#L44-L72', 'src://7b4887ce51d9/internal/store/migrate/migrate.go@49e396887c1888e3c538969847219bc20fa5e9eb:782b1259087d987dfb9b86d3f1de8f4162ef8d99#L69-L235', 'kb://3ec012f5b4d2/kb/architecture/store/migrations/ec215752.md', 'kb://3ec012f5b4d2/kb/decisions/store/migrate/control-chain-replayable/b0d77bc9.md', 'kb://3ec012f5b4d2/kb/conventions/store/migrations/idempotent-up-bodies/037ee64d.md', 'https://github.com/knomit/knomit/issues/326']
---
# control.db up-migrations must still be CREATE ... IF NOT EXISTS only after migrate-registry's removal (knomit#326); recovery re-execution is now the ONLY reason, and the fallback that tolerates a non-idempotent body recognises just two errors

TestControl_UpMigrationsAreIdempotentDDL (internal/store/migrate/control_test.go) still requires every statement in every control/*.up.sql to match `CREATE (TABLE|UNIQUE INDEX|INDEX) IF NOT EXISTS`. Until knomit#326 the stated reason was `knomit migrate-registry`, which exec'd the WHOLE chain as one script (migrate.ControlSchemaSQL) inside its own transaction. The command and ControlSchemaSQL are gone; the rule is UNCHANGED and now rests on recovery alone.

THE PATH THAT RE-RUNS A BODY: migrate.Control -> upWithRecovery. On a dirty version N, recoverDirty forces N-1 and calls Up again, which re-executes N's body against a database that may already hold what it created. With idempotent DDL that re-run is a no-op and recovery succeeds on its first path.

WHY NON-IDEMPOTENT SQL IS STILL A TRAP: the fallback, forcePastCommittedBody, is reached only when the re-run fails with an error alreadyApplied recognises ("already exists" / "duplicate column name"). So a re-run ALTER TABLE ADD COLUMN happens to be survivable, but a re-run INSERT double-applies SILENTLY and a re-run DROP fails with "no such table" and leaves control.db dirty. control.db is machine-wide, so a permanently dirty version makes EVERY repo unreachable at once (issue #33).

CONSEQUENCE: do not read the command's removal as licence to write ALTER/INSERT/DROP in a control migration. Loosening the rule to "re-runnable" rather than "CREATE IF NOT EXISTS only" is a separate design decision nobody has taken; it would need its own test replacing this one.

ALSO STILL TRUE: applied .sql files are immutable, so the comments inside control/000002, 000005, 000007 and 000009 that name the removed command stay as they are. A `grep migrate-registry` over *.sql is therefore never empty, and should not be made so.

MISREADING TO AVOID: facts that give the replay tool as the reason (the ec215752 synthesis "two appliers", b0d77bc9 on repo_subscriptions being a presence table) describe why the rule was ADOPTED. Their conclusion (idempotent DDL only) still holds; their reason does not.
