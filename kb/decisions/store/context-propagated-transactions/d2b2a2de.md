---
type: principle
domain: [store, concurrency, transactions]
confidence: 0.9
sources: 0
entities: [ctxKeyTx, Index.BeginTx, conn(ctx), context.Context, '*sql.Tx']
refs: ['src://knomit/.claude/plans/2026-03-30-context-tx-concurrency-design.md@0938d83']
---
# Context-propagated transactions: a single *sql.Tx in ctx participates in BOTH git storer and index writes

Concurrency safety across store + git uses context-propagated transactions. ctxKeyTx is SHARED between Index and Storer (both operate on the same *sql.DB). Every method that touches the database takes ctx context.Context as the FIRST parameter and calls conn(ctx) which returns the tx-from-context if present, otherwise the raw *sql.DB. When a caller starts a tx via idx.BeginTx(ctx), all downstream operations — git storer writes (s.conn(ctx)), index upserts (idx.conn(ctx)), pipeline updates — participate in the SAME transaction. This replaces the Storer's old stateful SetTx/ClearTx (not goroutine-safe). Caller-managed transactions in synthesize wrap git write + index update in ONE atomic commit. Anti-pattern to avoid: passing *sql.Tx as an explicit parameter — use ctx threading instead.
