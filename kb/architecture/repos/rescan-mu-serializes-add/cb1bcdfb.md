---
type: observation
domain: [repos, lifecycle, concurrency]
confidence: 0.95
sources: 0
entities: [Manager.Rescan, rescanMu, RescanResult, RescanError]
refs: ['src://knomit/internal/repos/manager.go@307b67d']
---
# Rescan holds rescanMu (separate from mu) so concurrent rescans cannot double-open a .db

Manager.Rescan() holds rescanMu (separate from m.mu) for the full scan duration to prevent two concurrent Rescan calls from racing to open the same .db twice. Inside the lock it globs repos/*.db, sorts, and for each: skips invalid names (no entry in any result slice); reports already-registered repos in Skipped; calls m.Add(name, dbPath) for new ones (errors go to Errors, do NOT abort the scan). All three result slices are non-nil empty []string{} / []RescanError{} on success — empty serializes as [] not null. Top-level error (unreadable repos dir or filepath.Glob failure) returns RescanResult{} (zero value) — callers should not inspect the result in that case. Anti-pattern: holding m.mu for the whole scan — Add itself takes mu briefly, would deadlock. rescanMu is the right granularity: serializes scans against each other without blocking concurrent Get/Set/Names from other code paths.
