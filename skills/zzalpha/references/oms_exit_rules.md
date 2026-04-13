# OMS — Closing Strategy & Exit Rules

Exit rules close positions based on P&L, time, Greeks, or absolute $ thresholds. They live under a generic `ExitRule[C]` registry pattern with Kit wrappers per instrument type.

## Architecture

```
ExitRule[C]            ← generic rule interface (C = context type)
ExitEvaluator[C]       ← evaluates a rule set against a context
Kit wrappers           ← EquityExitKit, OptionExitKit, SpreadExitKit
```

Files: `exit_rule.go`, `exit_kit.go`, `exit_rules_*.go` under `internal/oms/`.

## Closing strategy fields

All optional. Stored on `domain.ClosingStrategySpec`. Fields:

| Field | Type | Semantics |
|---|---|---|
| `stop_loss_price` | absolute underlying price | Hard stop when underlying ≤ threshold (long) / ≥ threshold (short) |
| `take_profit_price` | absolute underlying price | Target when underlying reaches threshold |
| `profit_pct` | fraction | P&L% target (0.50 = 50% of max profit) |
| `loss_pct` | fraction | P&L% stop |
| `intraday_profit_pct` | fraction | RTH-only profit target; see RTH gate below |
| `max_loss` | absolute $ | **Pre-trade entry validation cap only.** NOT evaluated during the trade |
| `max_dollar_loss` | absolute $ | Continuous guardrail. Fires per-tick when unrealized loss ≥ threshold |
| `max_dollar_profit` | absolute $ | Continuous profit target. Fires per-tick when profit ≥ threshold. Useful for ratio spreads with near-zero entry credit where `profit_pct` is meaningless |
| `max_hold_days` | days | Close after N calendar days from entry |
| `dte_exit` | days | Close when DTE ≤ threshold |
| `delta_exit` | float | Close when `|net_delta|` ≥ threshold (per-position normalized) |
| `gamma_exit` | float | Close when `|net_gamma|` ≥ threshold (per-position normalized) |
| `trailing_stop_pct` | float | 5.0 = 5% trailing from P&L% HWM |
| `eod_exit_time` | "HH:MM" ET | Force close at this ET time |

## Tie-breaking order

When multiple rules fire on the same tick, priority is fixed:

```
Expiration > GammaExit > DeltaExit > MaxDollarLoss($) > StopLoss
> LossPct > EODExitTime > DTEExit > MaxHold > IntradayProfitPct
> MaxDollarProfit($) > TakeProfit > ProfitPct > TrailingStop
```

**Why kill switches fire first:** gamma, delta, and absolute-$ kills fire before the P&L stop so a position exits before the P&L stop absorbs the full move. Tune thresholds with this ordering in mind.

## RTH gate on `intraday_profit_pct`

`intraday_profit_pct` is RTH-gated via `isBarRTH()`. Range is `[09:30, 16:00)` ET — **exclusive upper bound** because minute bars are left-labeled: a 16:00 bar is the first after-hours print, not the last RTH minute. `isBarRTH` is calendar-aware for early-close days (commit `37245d6`).

## Per-position normalization of delta/gamma thresholds

```
netDelta = Σ(bs_delta × side_sign × quantity) / totalContracts
```

With `side_sign` = +1 long / −1 short. Contract multiplier is deliberately omitted. The `/ totalContracts` division makes thresholds **size-invariant**: a 1-contract IC and a 10-contract IC use the exact same `delta_exit` value.

## Greeks plumbing

`delta_exit` / `gamma_exit` need per-tick `net_delta` / `net_gamma` computed from Black-Scholes.

### Replay path

`repriceSpread` + `computeSpreadGreeks` in `simulation_service.go` use `OptionsPricerAtUnderlying.QuoteAtUnderlying`, which returns `LegQuote{Price, Delta, Gamma}` in one call with **shared IV/TTE/div-yield inputs** for internal consistency.

### Live auto-exit path

`auto_exit.go:computeSpreadNetGreeks` uses the injected `GreeksCalculator`.

Both paths compute `netDelta` with the per-position normalization above.

### The hot-path gate

`SpreadExitKit.NeedsGreeks` is the gate. When it's `false`, Greeks are **never computed** — zero cost for spreads without `delta_exit` / `gamma_exit` rules. Always check this gate before adding per-tick computation.

### Error handling

Per-leg Greeks failures return errors (not silent zeros). Callers warn-once per position per day to prevent log floods while keeping failures visible for post-replay auditing.

### Known replay/live drift

Replay uses cached daily ATM IV for every leg, so OTM gamma is systematically overstated vs market-implied skew. **Tune `delta_exit` / `gamma_exit` thresholds against replay numbers, not live desk Greeks.** Documented in `ReplayDay` doc comment.

## Adding a new exit rule

1. Add the field to `domain.ClosingStrategySpec` with the correct type.
2. Implement `ExitRule[C]` in `exit_rules_*.go` for each instrument type it applies to.
3. Register in the correct Kit (`EquityExitKit`, `OptionExitKit`, `SpreadExitKit`).
4. Insert into the tie-breaking priority list at the correct position.
5. If it requires Greeks, flip `NeedsGreeks = true` in the Kit when the rule is set.
6. Wire both the replay path (`simulation_service.go`) and the live path (`auto_exit.go`).
7. Add unit tests for each firing condition + tie-breaking against adjacent rules.
