# OMS — Order Lifecycle, Fills, Positions

The OMS (`internal/oms/`) handles paper and Schwab live trading. This file covers the order/fill/position pipeline. Exit rules live in `oms_exit_rules.md`; replay determinism lives in `oms_replay.md`.

## Services

- **`OrderService`** — order lifecycle, fills, positions, collateral, simulation.
- **`PortfolioService`** — Greeks, risk dashboard.
- **`GroupService`** — strategy groups, rolls, adjustments.
- **`ReconciliationService`** — nightly Schwab ↔ OMS reconciliation.
- **`PositionMonitorService`** — continuous exit-rule evaluation.

## Two position tables

| Table | DB | Purpose |
|---|---|---|
| `positions` | main (`alphadb`) | Lightweight portfolio notes, user-facing |
| `trading_positions` | OMS (`alphadb_oms`) | Auto-derived from fills: cost basis, P&L, strategy |

Unique constraint on `trading_positions`: `(account_key, symbol, instrument_type, strike, expiry, position_side, strategy)`. The `idx_trading_positions_unique` columns are load-bearing for replay determinism — see `oms_replay.md`.

## Accounts

Accounts carry a `provider` (`paper`, `schwab-sim`, `alpaca`, `schwab`). Only `schwab` triggers live trading through the Schwab broker; all others are simulated locally. The per-account check is what gates `ALPHA_OMS_KEY` enforcement — paper/sim only need `ALPHA_API_KEY`, schwab live needs both.

`AccountRepo.ResetAccount` is an administrative bulk operation that refuses to run inside an outer tx (returns `ErrResetAccountNested`). See `architecture.md`.

## Paper trading

Paper fills run 24/7. Options are filled via `OptionsPricer` — bid/ask when available, Black-Scholes fallback otherwise.

**Slippage precedence** (first non-zero wins):

1. `slippage_Nleg` (leg-count-specific)
2. `spread_slippage_pct` (spread-wide)
3. `slippage_pct` (global)
4. 0

OMS disables gracefully if `OMS_DB_URL` is not set — AlphaDB still runs as a market-data gateway.

## Structure detection

`AnalyzeStructure` uses a detector chain, in order:

```
vertical_credit → vertical_debit → iron_condor → covered_call
→ straddle → strangle → calendar → diagonal → single_leg → custom
```

First match wins. Each detector is a pure function over the leg set; adding a new one is a matter of inserting it in the chain at the correct precedence.

## Fill pipeline

1. Order submitted → validated (collateral, buying power, entry guardrails like `max_loss`).
2. On fill, `trading_positions` is updated (cost basis averaged, P&L recomputed).
3. `PositionMonitorService` evaluates exit rules per tick. See `oms_exit_rules.md`.
4. Exit fill closes/reduces the position; realized P&L is booked.
5. Nightly `ReconciliationService` reconciles against Schwab for live accounts.

## Spread grouping

`domain.GroupPositionsBySpread` returns an **ordered** `[]SpreadGroup`. This ordering is load-bearing for replay determinism — never reintroduce map iteration here. See `oms_replay.md` for the full determinism contract.

## Strategy groups

`GroupService` manages multi-position strategy groups (e.g., a short strangle with two legs tracked as one unit, or a rolled iron condor). Rolls and adjustments happen inside `GroupService.adjustGroupAtomic`, which uses `TxRunner.RunTxCtx` for cross-repo atomicity.

## Pre-trade validation

`max_loss` is a **pre-trade entry validation cap** — it blocks the order from entering if the theoretical max loss exceeds the threshold. It is **NOT evaluated during the trade**. Continuous $ guardrails during a live trade are `max_dollar_loss` and `max_dollar_profit` — see `oms_exit_rules.md`.

## Corporate actions

`CorporateActionChecker` is injected and called during fill/position updates. Splits are partially implemented — see the project memory for current gaps.
