# OMS Architecture — Segregation, Services, DI

The OMS is a self-contained subsystem that handles order lifecycle, fills, positions, spreads, strategy groups, exit rules, reconciliation, and replay. It must be cleanly segregated from the rest of the platform so it can be versioned, tested, and replaced independently.

## Package segregation

**No package outside the OMS imports from the OMS.** Coupling flows inward only, via a small set of injected interfaces:

| Interface | Purpose | Implemented by |
|---|---|---|
| `PriceProvider` | Spot/last/bar quotes for underlyings | Market-data layer |
| `OptionsPricer` | Option quote by symbol/strike/expiry; BS fallback; **returns price and Greeks together** via `QuoteAtUnderlying(...) LegQuote{Price, Delta, Gamma, ...}` so shared IV/TTE/dividend inputs stay coherent (see `greeks_plumbing.md`) | Options service |
| `GreeksCalculator` | Standalone Greeks computation for paths where only Greeks are needed (e.g. portfolio-level risk dashboard); implementations often share inputs with `OptionsPricer` to avoid drift | Analytics layer |
| `CorporateActionChecker` | Splits, dividends, assignments affecting open positions | Corporate-actions service |
| `Clock` | `Now()` — single injection point, replaceable by an as-of clock for replay | Runtime |
| `Notifier` | Pushover/Slack/webhook for alerts | Notifications layer |

Any additional dependency goes through the same pattern: a narrow interface, implemented outside, injected at construction. If the OMS needs to import a package from outside, that's a sign the interface is wrong.

## Services

The OMS is organized as a small set of services, each owning one concern:

- **`OrderService`** — order submission, validation, fills, cancels, amendments, simulation. Owns the order state machine.
- **`PortfolioService`** — aggregate Greeks, buying-power dashboard, position P&L roll-up.
- **`GroupService`** — multi-position strategy groups, rolls, adjustments. Uses cross-repo transactions.
- **`ReconciliationService`** — nightly broker ↔ OMS reconciliation. Detects phantom positions, missing fills, drift.
- **`PositionMonitorService`** — continuous per-tick exit-rule evaluation against live or replay marks.
- **`SimulationService`** — replay of a trading day against historical data under an as-of boundary. Deterministic, bitwise-reproducible.

Each service takes its collaborators by constructor injection — no service locators, no package-level state.

## Separate database

The OMS owns its own database, distinct from any market-data or user-facing application schema. This is load-bearing for:

- **Blast radius:** administrative bulk operations (`ResetAccount`, `TruncateFills`) don't touch market data.
- **Backup/restore:** independent RPO/RTO for trading state vs analytics.
- **Replay cleanliness:** a replay run can start from a known-good trading state without coordinating with market-data migrations.
- **Schema ownership:** OMS migrations don't interleave with analytics migrations.

The OMS disables itself gracefully if its database URL is not configured — the rest of the platform keeps running as a market-data gateway.

## Dependency injection patterns

- **Constructor injection** for required dependencies: `NewOrderService(repo, pricer, greeks, clock, notifier)`.
- **Functional options** for optional dependencies: `NewSpreadRepo(db, WithTracer(t), WithMetrics(m))`.
- **Late binding via a `WorkerDependencies` pointer** to break circular deps when a background worker needs services that are themselves constructed with worker references. The pointer is populated after service init; accesses are read-only.

## Service wiring example

```
main.go
  ├── build pricer, greeks, corpActions, clock, notifier (outside-OMS deps)
  ├── open OMS pool (separate database)
  ├── build repos (OrderRepo, PositionRepo, SpreadRepo, AccountRepo, FillRepo, GroupRepo)
  ├── build services in dependency order:
  │     OrderService ← repos + pricer + greeks + clock
  │     PortfolioService ← repos + greeks
  │     GroupService ← repos + TxRunner
  │     ReconciliationService ← OrderService + BrokerAdapter + notifier
  │     PositionMonitorService ← OrderService + exit rule registry + clock
  │     SimulationService ← all of the above + asOfClock
  ├── wire background loops (PositionMonitor, LossMonitor, ReconScheduler)
  └── expose HTTP/RPC handlers that thin-delegate to services
```

The HTTP/RPC layer is always a thin delegate — no business logic lives there. All decisions (validation, routing, tie-breaking, atomicity) live inside services.

## What does not belong in the OMS

- **Market data storage.** Bars, quotes, chains come from the market-data layer via `PriceProvider` / `OptionsPricer`. The OMS never caches bars.
- **Strategy logic.** Strategies emit orders; the OMS executes them. "Should I enter this position?" is a strategy decision, not an OMS decision.
- **Analytics computation.** IV rank, skew, sentiment — those are inputs to strategies, not OMS concerns.
- **UI state.** The OMS exposes read APIs; the UI builds its own state on top.

If you find yourself adding one of these inside the OMS, the boundary is wrong. Push it out.
