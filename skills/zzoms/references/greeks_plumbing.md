# Greeks Plumbing — Shared Inputs, Hot-Path Gate, Drift

Greeks-based exit rules (`delta_exit`, `gamma_exit`) are the most expensive computation in the position-monitor hot path. Getting the plumbing right is the difference between a monitor loop that scales to thousands of positions and one that melts at 20.

## The two paths

Greeks are computed in two distinct pipelines:

| Pipeline | Purpose | Inputs | Pricer |
|---|---|---|---|
| **Replay** | Backtest and simulation | Cached daily ATM IV, historical quote cache, as-of clock | `OptionsPricerAtUnderlying.QuoteAtUnderlying` (one call returns `LegQuote{Price, Delta, Gamma}`) |
| **Live auto-exit** | Per-tick monitoring during live trading | Current market IV per strike, live quote stream, wall clock | Injected `GreeksCalculator` |

Both pipelines compute `netDelta` with the same per-position normalization (see `exit_rules.md`). The divergence in inputs is intentional and documented — see "Replay/live drift" below.

## Shared inputs inside one call

In the replay path, repricing and Greeks computation happen in **one function call** (`QuoteAtUnderlying`) that returns `{Price, Delta, Gamma}` together. The reason: the inputs (IV, TTE, dividend yield, risk-free rate) must be consistent across price and Greeks for the numbers to be internally coherent.

A common bug: compute price from one pricer, compute Greeks from another, with slightly different IV or TTE inputs. The per-position P&L then disagrees with the per-position delta — e.g. the position shows unrealized gain but the delta says it should have lost money. The one-call pattern eliminates the possibility.

The live path uses the injected `GreeksCalculator` but the same discipline applies: fetch all inputs once, compute price and Greeks from the same inputs, never mix.

## The NeedsGreeks gate

`SpreadExitKit.NeedsGreeks` (or equivalent per-position flag) gates Greeks computation entirely:

```
for position in positions:
    if position.kit.NeedsGreeks:
        greeks = computeGreeks(position)   # the expensive call
    else:
        greeks = nil                        # zero cost
    evaluateExitRules(position, marks, greeks)
```

When `NeedsGreeks` is `false`, the Greeks call is **never made**. This is the difference between O(N * legs * pricing_cost) and O(N) per tick.

The gate is set when the Kit is constructed from the closing strategy spec: if `delta_exit` or `gamma_exit` is configured, `NeedsGreeks = true`; otherwise `false`. The gate is per-position, not global — a strategy with 100 positions where only 10 use delta exits pays Greeks cost for 10, not 100.

**Always check this gate before adding per-tick computation.** Any new Greeks-dependent rule must flip `NeedsGreeks` on the Kit when set, and the new computation must sit behind the gate.

## Per-leg error handling

A Greeks computation can fail per-leg (missing IV, stale quote, numerical issue in BS). The rule:

- **Never return silent zeros on failure.** A zero-delta position looks fine to the evaluator; the exit rule never fires; a bad position sits open.
- **Propagate per-leg errors up to the position monitor.**
- **Warn-once per position per day.** Log once on first failure; suppress subsequent failures for the same position for the rest of the session. This prevents log floods while keeping the failure visible.
- **The position is not closed on failure.** There's no basis for a decision. The next tick retries.

Post-replay auditing: persist Greeks failures to a `greeks_errors` table (position_id, timestamp, reason) so a weekly review can identify chronically-failing positions (usually ultra-OTM options with missing quotes or expired strikes the market-data layer hasn't cleaned up).

## Replay/live drift

**Replay uses cached daily ATM IV for every leg.** Live uses the market-implied IV per strike. The consequence:

- **OTM gamma is systematically overstated in replay.** The cached ATM IV is lower than what the OTM strike actually trades at (skew premium is missing). Lower IV → higher gamma (gamma is inversely proportional to vega, roughly).
- **OTM delta is less affected** but still drifts; the direction depends on whether the OTM is a call or put and where the skew lies.
- **ATM Greeks match closely** — that's where the cached IV is anchored.

### Implications for threshold tuning

Thresholds calibrated against live desk Greeks will fire too often in replay. Thresholds calibrated against replay Greeks will fire less often in live. You cannot have both.

**The rule: tune against replay.** Rationale:

- Replay is the only path where you can systematically test threshold choices across thousands of historical days.
- The drift is bounded and documented.
- A threshold that's "slightly too loose" in live is better than one that's "chronically too tight" in replay — the former means a strategy occasionally sits on a losing position a tick longer; the latter means your backtest under-reports every strategy that uses Greeks-based exits.

Document the choice in the `ReplayDay` doc comment (or equivalent) so future maintainers don't try to "fix" the cached-IV replay to match live. That fix would break cross-run determinism (every replay run would depend on which market-data snapshot was used to compute IV).

## Dividend yield

Black-Scholes Greeks require a dividend yield. Options on dividend-paying underlyings have materially different deltas and early-assignment probabilities. The rule:

- **Dividend yield is an input, not a constant.** Inject a `DividendYieldProvider` (or include it in the `OptionsPricer` interface).
- **In replay, use the historical yield as of the as-of date.** Not the current yield. This is load-bearing for determinism and for correctness — a strategy running in 2020 should see 2020 yields, not 2026 yields.
- **Cache per symbol, per day.** Dividend yield doesn't change intraday; a daily cache is sufficient.

## TTE (time to expiration)

TTE is computed as (expiry_date_close - current_time) in year-fractions. Two gotchas:

- **Day count convention.** For US-listed equity options, the canonical convention is **`ACT/365 fixed`** — this is what OCC/CBOE reference pricing uses and what desk traders compare against. `ACT/252` (trading days) is a common research convention but will produce Greeks that disagree with every live desk quote by a visible amount; using it on a production OMS is a recurring source of "your delta is wrong" support tickets. Pick `ACT/365 fixed` unless there's a specific reason to deviate, and document the choice where it's set.
- **Consistency.** Whatever convention you pick, price and Greeks must use the same one. Mixing conventions between the two is a silent source of drift.
- **Intraday TTE decay.** A 1 DTE position at market open has TTE ≈ 1/365; at close it's ≈ 0. Per-tick recomputation matters for gamma scalping and 0 DTE strategies. For multi-day holds, once-per-day TTE is fine.

In replay, use the as-of clock for "current time." Wall clock will advance during the replay and desynchronize from the historical market state.
