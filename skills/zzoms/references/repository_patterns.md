# Repository Patterns — DBTX, Beginner, Cross-Service Atomicity

The OMS data layer is where silent bugs love to live. The patterns below enforce one shape for repositories so the bugs can't hide — and so the type system rejects the configurations that cause them.

## Single field, two interface shapes

Every OMS repo holds **exactly one** database field:

- **`db DBTX`** — narrow: `Exec`, `Query`, `QueryRow`. Used by repos that never need multi-statement atomicity.
- **`db Beginner`** — DBTX plus `Begin`. Used by repos that need to start their own transactions for multi-statement writes (e.g. inserting a spread order plus its legs atomically).

**Never split into `pool *pgxpool.Pool + db DBTX`.** The split shape is how every silent "cannot run inside a transaction" bug is born — some methods use the pool (bypassing tx), others use the db (respecting tx), and the choice is invisible at the call site.

A single field forces every method in the repo to go through the same path. If that path is a tx-scoped DBTX, the whole repo respects the tx; if it's a pool, the whole repo bypasses it. No methods can silently diverge.

## `WithTx` takes concrete `pgx.Tx`

```go
func (r *SpreadRepo) WithTx(tx pgx.Tx) *SpreadRepo {
    return &SpreadRepo{db: tx}
}
```

Not `DBTX`. Passing a pool into `WithTx` is nonsense — `WithTx` means "give me a copy of this repo scoped to this transaction," and the type system should refuse a pool at that site.

The returned repo has the same field shape as the original (same `db` field), but the field now holds a `pgx.Tx` instead of a pool. Every method on the copy implicitly runs against the tx.

## Transactional methods type-switch on `r.db`

Repos that need their own atomicity (Beginner field) type-switch to handle both "I am top-level, start my own tx" and "I am tx-scoped, trust the outer tx":

```go
func (r *SpreadRepo) Create(ctx context.Context, spread *domain.SpreadOrder) error {
    if tx, ok := r.db.(pgx.Tx); ok {
        // tx-scoped: already inside a tx, just use it
        return r.insertSpread(ctx, tx, spread)
    }
    // top-level: start our own tx
    tx, err := r.db.Begin(ctx)
    if err != nil {
        return fmt.Errorf("begin: %w", err)
    }
    defer func() { _ = tx.Rollback(ctx) }()
    if err := r.insertSpread(ctx, tx, spread); err != nil {
        return err
    }
    return tx.Commit(ctx)
}
```

`insertSpread` takes a `pgx.Tx` explicitly. It's a private helper that's always called inside a tx — no ambiguity about whether it will commit or not.

**The type switch is the pattern.** A repo that doesn't type-switch and always starts its own tx will fail when called from inside a cross-service tx runner (nested Begin on a connection is an error in most drivers). A repo that doesn't type-switch and always assumes it's tx-scoped can't be called from top-level code.

## Cross-service atomicity via TxRunner

When multiple services need to participate in a single transaction (e.g. `GroupService.adjustGroupAtomic` touches groups, positions, orders, accounts — each owned by a different repo), use a **context-threaded transaction runner**:

```go
func (r *TxRunner) RunTxCtx(ctx context.Context, fn func(context.Context) error) error {
    tx, err := r.pool.Begin(ctx)
    if err != nil {
        return err
    }
    defer func() { _ = tx.Rollback(ctx) }()

    txCtx := ContextWithConn(ctx, tx)
    if err := fn(txCtx); err != nil {
        return err
    }
    return tx.Commit(ctx)
}
```

Inside the closure, services call their injected **top-level** repos. Each repo's `conn(ctx)` helper picks up the tx from the context:

```go
func (r *PositionRepo) conn(ctx context.Context) DBTX {
    if tx, ok := ConnFromContext(ctx); ok {
        return tx
    }
    return r.db
}
```

Every method in the repo uses `r.conn(ctx)` instead of `r.db` directly. When called outside a tx context, it falls through to the normal field. When called inside a tx context, it uses the tx. No explicit `WithTx` calls are needed at the caller — the context carries the tx transparently.

**This is the only reliable pattern for cross-service transactions.** Passing `pgx.Tx` explicitly through every method signature clutters the code and tempts people to bypass it. The context thread is invisible but enforced at every repo access.

## Administrative bulk operations refuse to run inside a tx

Some operations touch many tables and hold row locks for the duration. Running them inside an outer business transaction causes:

- **Lock queue pileups.** Other transactions wait on the locks for minutes.
- **Idle-in-transaction bloat.** The outer tx sits holding locks while the bulk op runs.
- **Autovacuum starvation.** Vacuum can't process dead rows while any tx has a snapshot covering them.

`AccountRepo.ResetAccount` is the canonical example. It touches ~10 tables. The pattern:

```go
func (r *AccountRepo) ResetAccount(ctx context.Context, accountKey string) error {
    if _, ok := r.db.(pgx.Tx); ok {
        return ErrResetAccountNested
    }
    // safe: we're top-level, start our own tx (or don't — per-table batches)
    ...
}
```

The type switch detects tx-scoped calls and rejects them. Callers must invoke `ResetAccount` on the top-level repo, not on a tx-scoped copy. The error is a sentinel so callers can match it and handle it explicitly.

**Lint rule enforcement.** Teams that want to guarantee the shape across the codebase add a lint rule that checks every OMS repo for:

- Exactly one `db` field (no `pool + db` split).
- `WithTx(pgx.Tx)` signature, not `WithTx(DBTX)`.
- Type-switch on `r.db` in methods that start their own tx.

The lint rule is what stops the shape from drifting under pressure. Code review alone is not enough; the same bugs reappear every few months in a team without enforcement.

## Why no repo-level savepoints

Savepoints let a single transaction roll back partial work without aborting the whole tx. Tempting for error recovery inside a repo method. **Avoid them in OMS repos.**

Reasons:

- **Callers bail on the first error.** OMS services treat a repo error as a fatal event — they don't try to recover with alternative logic. Savepoints serve no caller.
- **Savepoints confuse the observability view.** Metrics, traces, and log correlation are easier when a tx is linear.
- **Savepoints don't help with the lock problem.** Held savepoints still hold locks.

The one exception is **statement-error containment at the tx runner**: if a `RunTxCtx` closure invokes a nested `RunTx` call (e.g. a helper that internally calls another service), the runner wraps the nested closure in a savepoint so a statement error inside it doesn't force the outer tx to abort. This is a runner feature, not a repo feature.

## Testing repositories

Two test flavors:

- **Unit** with `pgxmock`. Fast, no container, tests the happy path and the type switch branches.
- **Integration** with a real Postgres (testcontainers or shared dev DB). Tests the multi-statement atomicity, the uniqueness constraints, the rollback-on-error path.

Both are needed. Unit tests catch bugs in the repo's logic; integration tests catch bugs in the SQL, the constraints, and the tx semantics.

**After changing repo interfaces, regenerate mocks.** If mocks are manually maintained, they drift; if they're generated, they stay in sync. The mock file for a repo is load-bearing for service-level tests.
