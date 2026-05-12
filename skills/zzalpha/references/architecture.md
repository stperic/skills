# AlphaDB Architecture

AlphaDB is an MCP server that gives LLM trading agents a unified interface to market data. It routes each request to the right upstream provider and uses its own database exclusively for **computed insights** that providers don't offer (sentiment scores, IV rank/percentile, correlation, volatility analysis, beta).

## Key principles

- AlphaDB does NOT replicate upstream provider data. Polygon (Massive plan) and Alpha Vantage serve raw market data directly. The database stores only what AlphaDB computes itself or needs for computing insights.
- **Ticker management is simplified.** Clients add/remove tickers for analytics tracking. AlphaDB decides internally what to collect based on `TypeDefaults` per ticker type.
- **Long-term vision: Agentic Analytics.** LLM analysts create custom insight jobs at runtime. Future work, not current priority.

## Layer overview

```
cmd/alphaserver/main.go           ← DI wiring, startup, graceful shutdown
cmd/alphadb/main.go               ← CLI entrypoint → internal/cli/cmd/

internal/transport/http/           ← Gin handlers, routes, middleware
internal/service/                  ← Business logic (MarketDataService, InsightsService)
internal/queue/                    ← River job queue (workers, client, adapter)
internal/repository/postgres/      ← pgx repositories, batch writers
internal/provider/                 ← Provider implementations (Polygon, tastytrade, AV, FRED, Schwab)
internal/domain/                   ← Models, error types, enums
internal/config/                   ← Viper configuration
internal/timeutil/                 ← Eastern timezone utilities
internal/observability/            ← OpenTelemetry tracing, metrics, logging
internal/notify/                   ← Notification services (Pushover)
internal/worker/                   ← Background scheduler
internal/cli/                      ← Cobra commands + HTTP API client
internal/oms/                      ← Order Management System (paper + Schwab live trading)
migrations/oms/                    ← OMS database schema (oms_schema.sql embedded)
```

## Dependency injection

- **Functional options** for optional deps: `NewBarRepo(pool, WithBarRepoTracer(t))`.
- **Constructor injection** for required deps: `NewMarketDataService(barRepo, ...)`.
- **Late binding** via `WorkerDependencies` pointer populated after service init — used to break circular deps between workers and services.

## Proxy pattern

Provider proxies implement `ProxyProvider` (one method: `ProxyRequest`). `CachedProxy` wraps any provider with Redis caching.

| Provider | Auth | Notes |
|---|---|---|
| tastytrade | Bearer token (auto-refresh via `AuthManager`) | Unique market_metrics source |
| Schwab | Bearer token (auto-refresh via `AuthManager`) | Tokens fetched from Cloudflare Worker `schwabauth.stperic.workers.dev` |
| Polygon | Query param `apiKey` | Massive plan |
| FRED | Query param `api_key` | Rates, macro series |
| Alpha Vantage | Query param `apikey` | 75 req/min rate limit — chunk calls |

The `QUOTE_PROVIDER` env var (`schwab` default, `polygon`, `tastytrade`) switches the upstream for `bars`, `quotes`, `options_chain`, `options_expirations`, and composite tools (`symbol_context`, `market_context`). `bars` uses Schwab (default) or Polygon (tastytrade has no bars). `market_metrics` always uses tastytrade.

## River job queue

Workers are thin — each delegates to a processor interface. Key files: `internal/queue/workers.go`, `workers_bulk.go`, `client.go`, `job_service.go`, `adapter.go`.

**Adding a new job:** Define `XxxJobArgs` with `Kind()` in `workers.go` → create worker → register in `NewClient()` → wire processor in `main.go`. For symbol+date range jobs, embed `SymbolSyncArgs` and use `symbolSyncWork()`.

### Nightly sync workflow DAG

```
stocks_gen → stocks_1m → stocks_1d ─┐
                                    ├─→ options_1m → options_1d
rates (no deps) ───────────────────┘
indices_gen → indices_1m → indices_1d
stocks_1d + rates + options_1d → iv_daily:* (per symbol)
stocks_1d + rates + options_1d → sync_monitor
sentiment:* (per symbol, no deps — rate-limited queue)
```

### Job patterns

- **Date ranges** come from `sync_tracking` dates and `BULK_SYNC_*_START_DATE` env vars, never hardcoded lookbacks.
- **Scheduling** is driven by `tracked_symbols` + `TypeDefaults`.
- **Rate limits** — chunked API calls for Alpha Vantage (75 req/min) and Polygon.
- **Verification** — every data-writing job must register a verifier in `verification.go`. Verification failures fail the job and trigger a Pushover notification.

## OMS segregation

Fully segregated under `internal/oms/`. Coupling points are a small set of injected interfaces:

- `*pgxpool.Pool` (OMS-dedicated pool, `OMS_DB_URL`)
- `*schwab.AuthManager`
- `PriceProvider`
- `OptionsPricer`
- `GreeksCalculator`
- `CorporateActionChecker`

**No other package imports from `internal/oms/`.** The OMS has its own database (`alphadb_oms`) with schema in `migrations/oms/oms_schema.sql` (embedded).

Services:

- `OrderService` — order lifecycle, fills, positions, collateral, simulation
- `PortfolioService` — Greeks, risk dashboard
- `GroupService` — strategy groups, rolls, adjustments
- `ReconciliationService` — nightly Schwab ↔ OMS reconciliation
- `PositionMonitorService` — continuous exit-rule evaluation

Exit rules use a generic `ExitRule[C]` registry pattern — see `oms_exit_rules.md`.

## Two-process deployment (live + replay)

A single `alphaserver` binary runs as **two systemd units** sharing one OMS DB. Isolation is by `account_key` at the service layer through `RoleGuard` (`internal/oms/domain/role.go`):

- `alphaserver-live` (`:8080`, `ALPHADB_ROLE=live`) — owns writes to `ALPHADB_LIVE_ACCOUNT_KEYS` (typically `paper,schwab-sim`), runs all background loops (auto-exit, loss monitor, watchdog, nightly scheduler, reconciliation).
- `alphaserver-replay` (`:8081`, `ALPHADB_ROLE=replay`) — owns writes to every other account (replay/sim), sets per-request `asOfBoundary` via `/v1/admin/as-of-date`, **no background loops**.

Writes to the wrong role return `400 WRONG_ROLE`. The replay process never sees background loop ticks, so the `asOfBoundary` it sets cannot bleed into live work. See `oms_replay.md` for the full invariant + routing summary.

## OMS repository pattern

OMS repos (`internal/oms/repository/postgres/`) follow one shape to avoid the silent "cannot run inside a transaction" bugs that repeat when the shape drifts. Enforced by the `oms-split-pool-db` lint rule.

**Single field, two interfaces.** A repo holds either `db DBTX` (narrow: Exec/Query/QueryRow — most repos) or `db Beginner` (DBTX + Begin — repos that need their own atomicity, currently SpreadRepo and AccountRepo). Never split into `pool *pgxpool.Pool + db DBTX`.

**`WithTx` takes concrete `pgx.Tx`.** Not `DBTX`. Passing a pool into `WithTx` is nonsense and the type system should forbid it.

**Transactional methods type-switch on `r.db`.** Repos that need multi-statement atomicity use:

```go
func (r *SpreadRepo) Create(ctx context.Context, spread *domain.SpreadOrder) error {
    if tx, ok := r.db.(pgx.Tx); ok {
        return r.insertSpread(ctx, tx, spread) // trust the outer tx
    }
    tx, err := r.db.Begin(ctx)
    if err != nil { return fmt.Errorf("begin: %w", err) }
    defer func() { _ = tx.Rollback(ctx) }()
    if err := r.insertSpread(ctx, tx, spread); err != nil { return err }
    return tx.Commit(ctx)
}
```

**No repo-level savepoints.** OMS callers always bail on the first error. The one place savepoints are used is `TxRunner.runInSavepoint`, for statement-error containment when a `RunTxCtx` closure invokes nested `RunTx` calls.

**Administrative bulk operations refuse to run inside an outer tx.** `AccountRepo.ResetAccount` returns `ErrResetAccountNested` if called on a tx-scoped copy. Holding row locks across ~10 tables for the duration of a business tx causes lock queue pileups, idle-in-transaction bloat, and autovacuum starvation.

**Cross-service transaction boundaries** use `TxRunner.RunTxCtx(ctx, fn)` to wrap ctx with a tx. Services inside the closure call their injected top-level repos; each repo's `conn(ctx)` helper (via `ConnFromContext`) picks up the tx from the context. `GroupService.adjustGroupAtomic` is the load-bearing example.

## Database conventions

- Migrations in `/migrations/` — use `IF NOT EXISTS`, `ON CONFLICT DO NOTHING`.
- `REAL` for prices/Greeks, `NUMERIC(8,2)` for strikes.
- PK order: `(symbol, time)` — optimized for `WHERE symbol = X AND time >= ...`.
- `pgx.CopyFrom()` for bulk inserts (>10k rows), `pgx.Batch` for incremental.
- Hypertable chunks: 7 days, compression after 7 days.

## Technology stack

Go 1.26+, TimescaleDB v2.25 (PostgreSQL 18), River v0.30 (job queue), Gin, pgx/v5, gorilla/websocket, viper.
