---
type: principle
domain: [store, schema, git]
confidence: 0.95
sources: 0
entities: [commit_log, branch_commits]
refs: ['src://knomit/.claude/plans/2026-04-07-knomit-invariant-test-suite-design.md@0938d83', kb/invariants/store/two-table-commit-index/07d4fede.md]
---
# Commit indexing uses TWO tables: commit_log (branch-agnostic) + branch_commits (many-to-many)

Commit indexing is split into TWO tables matching git's separation of 'commit facts' from 'branch visibility': (1) commit_log — branch-agnostic, one row per (commit_hash, path), records timestamp/message/operation/author/action, immutable once written; (2) branch_commits — many-to-many, one row per (branch_id, commit_hash), records which branches see which commits, ON DELETE CASCADE on branch_id so dropping a branch reclaims visibility rows. commit_log rows persist as orphans (objects-stay-in-the-store, like git). Branch creation clones the parent's visibility set via one INSERT OR IGNORE ... SELECT — O(rows) at creation, zero copying afterward. All read paths JOIN through branch_commits. ORIGINAL DESIGN WAS WRONG: PK was (commit_hash, path) with a branch_id column — shared commits could only carry ONE branch_id, so INSERT OR IGNORE silently dropped rows for the second branch and every branch-scoped query returned wrong results. Discovered 2026-04-08 (commit 4fd7d4d).
