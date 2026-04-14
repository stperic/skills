# Exit Rules — Generic Registry, Tie-Breaking, Normalization

Exit rules close positions based on P&L, time, Greeks, or absolute dollar thresholds. The design goal is a single engine that evaluates any rule set against any instrument type, with a fixed tie-breaking order so behavior is predictable across runs.

## Architecture

```
ExitRule[C]            ← generic rule interface, C = context type (Equity | Option | Spread)
ExitEvaluator[C]       ← evaluates a rule set against a context, returns the first firing rule per tie-breaking order
Kit wrappers           ← EquityExitKit, OptionExitKit, SpreadExitKit — collect rules applicable to that instrument type
```

A Kit holds the rules that apply to its instrument type plus computed flags like `NeedsGreeks`. The evaluator is parameterized on the context type so the same engine handles an equity (`EquityContext`) and a spread (`SpreadContext`) without reflection.

## Closing strategy fields

The closing strategy spec is a plain data struct; every field is optional. Canonical fields:

| Field | Type | Semantics |
|---|---|---|
| `stop_loss_price` | underlying price | Hard stop: underlying ≤ threshold (long) / ≥ threshold (short) |
| `take_profit_price` | underlying price | Target: underlying reaches threshold |
| `profit_pct` | fraction | P&L% target (0.50 = 50% of max profit) |
| `loss_pct` | fraction | P&L% stop |
| `intraday_profit_pct` | fraction | RTH-only profit target |
| `max_loss` | absolute $ | **Pre-trade entry validation cap only** — NOT evaluated during the trade |
| `max_dollar_loss` | absolute $ | Continuous per-tick unrealized loss kill |
| `max_dollar_profit` | absolute $ | Continuous per-tick unrealized profit target (useful for ratios with near-zero entry credit) |
| `max_hold_days` | days | Close after N days from entry. **Specify calendar vs trading days explicitly** — a Friday entry with `max_hold_days=3` on calendar semantics closes Monday's pre-open; on trading semantics it closes Wednesday. The skill recommends trading days as the default and storing the choice on the spec itself (`max_hold_days_unit = trading|calendar`). Do not pick silently. |
| `dte_exit` | days | Close when DTE ≤ threshold |
| `delta_exit` | float | Close when `\|net_delta\|` ≥ threshold (per-position normalized) |
| `gamma_exit` | float | Close when `\|net_gamma\|` ≥ threshold (per-position normalized) |
| `trailing_stop_pct` | float | % trailing from P&L% HWM |
| `eod_exit_time` | "HH:MM" ET | Force close at this ET time |

`max_loss` vs `max_dollar_loss` is the most commonly conflated pair in OMS design. `max_loss` blocks entry; `max_dollar_loss` kills during the trade. If a strategy wants both — e.g. "don't enter if theoretical max loss > $500, and kill the trade if unrealized loss exceeds $400" — both fields are set, and they mean different things.

## Tie-breaking order

When multiple rules fire on the same tick, priority is **fixed**:

```
Expiration > GammaExit > DeltaExit > MaxDollarLoss($) > StopLoss
> LossPct > EODExitTime > DTEExit > MaxHold > IntradayProfitPct
> MaxDollarProfit($) > TakeProfit > ProfitPct > TrailingStop
```

**Kill switches fire first.** The rationale: gamma, delta, and absolute-$ kills must fire *before* the P&L stop so a position exits before the P&L stop absorbs the full move. If `loss_pct` fires at the same tick as `delta_exit`, the delta kill wins — otherwise the position would have already taken a 50% loss by the time you notice the delta breach.

**Expiration fires before all.** A position at expiration has no remaining optionality; any other rule that would fire is moot.

**TrailingStop fires last.** Trailing stops are the last line of defense — they only kick in if nothing else has already closed the position.

**Adding a new rule means inserting it into this order explicitly.** Appending is a bug — the new rule will never beat a profit-target rule and you'll wonder why it never fires.

### The time-based vs P&L ordering — a deliberate choice, not an accident

The order places `MaxDollarLoss > StopLoss > LossPct > EODExitTime > DTEExit > MaxHold`. A reader will notice that **P&L thresholds rank above time-based hard cutoffs** (EOD, DTE, MaxHold). The reverse order — "deadlines beat preferences" — is a defensible alternative and is what many desks run.

The rationale for putting P&L above time:

1. A position that has already breached its loss threshold **should exit now**, not wait for the EOD bar. If a position is down 15% at 14:30 ET and EODExitTime is 15:55 ET, firing on `LossPct` at 14:30 is the intended behavior.
2. A position that has already breached its profit target **should book the profit now**, for the same reason. The profit-target rules (`TakeProfit`, `ProfitPct`) sit below the time cutoffs, so a profit target and an EOD both pending will close at EOD — which is consistent with "let profits run until the deadline."
3. Time-based rules are the **last line of defense against forgetting**, not the preferred exit. If a position reaches its MaxHold without any P&L rule firing, something unusual is happening and a time-based close is correct.

If your project disagrees (e.g. a tax-sensitive strategy that **must** close before session end regardless of P&L), override the order in that project's CLAUDE.md. The canonical order above is the skill default; document the override where you deviate.

## Per-position normalization of delta/gamma

```
netDelta = Σ(bs_delta × side_sign × quantity) / totalContracts
```

Where `side_sign` is `+1` for long legs and `-1` for short legs.

- **Contract multiplier is deliberately omitted.** `delta_exit` is a per-contract threshold, not a dollar threshold.
- **`/ totalContracts` makes the threshold size-invariant.** A 1-contract iron condor and a 10-contract iron condor use the exact same `delta_exit` value. This is crucial for portfolio-level strategies — you don't want a 10-lot to fire on a threshold calibrated for a 1-lot.

Gamma uses the same normalization.

## RTH gating on intraday rules

`intraday_profit_pct` fires only during regular trading hours. The canonical check for US equities and US-listed options:

```
isBarRTH(bar_time) = (bar_time in [09:30, 16:00) ET)  AND  not an early-close day
```

**Exclusive upper bound.** Minute bars are left-labeled: a bar stamped `16:00` is the first after-hours print, not the last RTH minute. Including 16:00 would fire the rule on the first after-hours print, which is usually the worst quote of the day.

**Calendar-aware for early-close days.** Thanksgiving, Christmas Eve, and similar days close at 13:00 ET. The RTH check must consult a market calendar to apply the early close.

**RTH is per-venue.** The 09:30/16:00 ET window is NYSE/Nasdaq. LSE, TSE, HKEX, and futures markets (near-24-hour sessions) have their own windows and their own early closes. An OMS that trades multiple venues must consult a per-venue calendar service, not hardcode US equity hours.

## Hot-path gate: NeedsGreeks

`SpreadExitKit.NeedsGreeks` (and equivalents per instrument type) is `false` unless a Greeks-dependent rule (`delta_exit`, `gamma_exit`) is in the rule set.

- **When false:** Greeks are **never computed**. The position monitor loop skips the Greeks call entirely.
- **When true:** Greeks are computed per-tick for that position.

This gate is load-bearing. Greeks computation is the most expensive thing in the exit-rule hot path; without the gate, every position pays the cost even if no Greeks-dependent rule is configured. Always check this flag before adding per-tick computation.

## Error handling for Greeks

Per-leg Greeks failures return errors (not silent zeros). Silent zeros are the worst possible default: a leg with zero delta looks fine to the evaluator, the rule doesn't fire, and a bad position sits open.

The pattern:

- Per-leg failure → propagate the error up to the position monitor.
- Position monitor catches it and logs a **warn-once per position per day**. This prevents log floods while keeping failures visible for post-mortem.
- The position is **not closed** on a Greeks failure — there's no basis for the decision. But the rule is also not considered "passed." The next tick retries.

## Replay/live drift for Greeks-based rules

Replay uses cached daily ATM IV for every leg; live uses the current market-implied IV per-strike. OTM gamma is systematically **overstated** under replay because the cached ATM IV is lower than the OTM strike's actual IV (the skew premium is missing).

**Tune `delta_exit` / `gamma_exit` thresholds against replay numbers, not live desk Greeks.** A threshold calibrated against a live desk's Greeks will fire too often in replay and mask the strategy's true historical performance.

Document this drift in the `ReplayDay` or equivalent doc comment so future maintainers don't try to "fix" the replay Greeks to match live.

## Adding a new exit rule

1. Add the field to the closing strategy spec with the correct type.
2. Implement `ExitRule[C]` for each instrument context type (equity / option / spread) it applies to.
3. Register in the appropriate Kit(s).
4. **Insert into the tie-breaking priority list at the correct position** — not append.
5. If it requires Greeks, flip `NeedsGreeks = true` in the Kit when the rule is set.
6. Wire both the replay path and the live auto-exit path — they share the rule but may compute inputs differently (see `greeks_plumbing.md`).
7. Add unit tests for:
   - Each firing condition (positive, negative, boundary).
   - Tie-breaking against adjacent rules in the priority list.
   - No-op when the field is unset.
   - RTH gating if time-dependent.

## Common exit rule bugs

- **`profit_pct` on a debit structure with zero max profit.** Division by zero. Use `max_dollar_profit` for ratio spreads and similar near-zero-entry-credit structures.
- **`stop_loss_price` on a spread.** The semantics are ambiguous — is "price" the underlying, one leg, or the net? Default to the underlying and document it.
- **Time-based rules in UTC.** `eod_exit_time` must be in market time (ET), not UTC, or daylight saving transitions silently shift the exit by an hour.
- **`dte_exit` vs `max_hold_days` confusion.** DTE is counted to expiry; max hold is counted from entry. A position opened 45 DTE with `max_hold_days = 30` closes at 15 DTE; with `dte_exit = 21` closes at 21 DTE. They are not interchangeable.
