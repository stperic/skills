# Risk Controls — Pre-Trade Validation and Continuous Guardrails

Risk controls in an OMS split cleanly into two phases: **pre-trade** (does this order get to enter?) and **continuous** (does this open position get to keep living?). Conflating them is the most common design bug in junior OMS codebases.

## The two phases

| Phase | When | Examples | What fires |
|---|---|---|---|
| **Pre-trade** | Before an order transitions `NEW → SUBMITTING` | `max_loss` cap, collateral, buying power, structure sanity | Validation returns an error; order goes to `REJECTED` |
| **Continuous** | Every tick while the position is open | `max_dollar_loss`, `max_dollar_profit`, `delta_exit`, `gamma_exit`, stop loss | Exit rule fires; position is closed via a normal exit fill |

**`max_loss` is pre-trade only.** It is a **theoretical max loss cap on the position at entry** — "if this trade goes to its worst case, the loss would be $X; if $X > threshold, block entry." It is **not evaluated during the trade**. Continuous $ guardrails during a live trade are `max_dollar_loss` and `max_dollar_profit`. See `exit_rules.md`.

This distinction matters because:

- **`max_loss` and `max_dollar_loss` have different semantics.** `max_loss` is the *defined* worst case (known at entry — e.g. width minus credit for a credit spread). `max_dollar_loss` is the *observed* unrealized loss (known per tick).
- **Both can be set on the same strategy.** A strategy might say "don't enter if theoretical max loss > $500, and kill the trade if unrealized loss exceeds $400." Those are two different thresholds, both valid, both enforced at different phases.
- **Conflating them corrupts trade entry.** A pre-trade gate that uses `max_dollar_loss` semantics (current unrealized) makes no sense on an order that hasn't filled yet.

## Pre-trade validation pipeline

Ordered cheapest first so common failures reject without external calls:

1. **Structural validity** — leg count matches structure, strikes monotone where required, expiry in the future, quantity > 0, no duplicate legs.
2. **Instrument resolution** — can the OMS resolve each leg to a tradable instrument? (expired options, unknown symbols, wrong exchange).
3. **Entry guardrails** — `max_loss` theoretical cap, concentration limits (no more than N% of account in one underlying, no more than M positions in one strategy).
4. **Collateral / buying power** — does the account have the margin/cash for this structure?
5. **Broker-level checks** — PDT restrictions, halted symbols, locked-down accounts, day-trade count.

Failures write `REJECTED` with a reason code. Do not silently drop — every rejection is a row for audit and backfill.

## Theoretical max loss computation

For common structures:

| Structure | Max loss |
|---|---|
| Long call / put | Premium paid |
| Short call (naked) | Unlimited (use a buying-power proxy instead) |
| Short put (cash-secured) | `(strike × multiplier) − premium received` — this is the **actually observable** max loss at underlying = 0 (bankruptcy), not a theoretical ceiling. Broken-symbol overnight events make this real, not hypothetical. |
| Vertical credit | (Width × multiplier) − premium received |
| Vertical debit | Premium paid |
| Iron condor | max(call_width, put_width) × multiplier − total_premium |
| Covered call | Stock basis × shares − premium received (downside unbounded) |
| Straddle / strangle (long) | Total premium paid |
| Straddle / strangle (short) | Unlimited (use a proxy: 2× or 3× credit, or fixed dollar, or defined-risk conversion) |

**Short undefined-risk strategies** (naked calls, short straddles/strangles) have no finite theoretical max loss. Strategies that trade them must use one of:

- A **buying-power proxy** (broker-computed initial requirement × safety factor).
- A **defined-risk conversion** (add a protective wing to bound the risk).
- A **fixed dollar cap** as a policy decision ("treat this as if max loss is $10,000").

The OMS should support all three and make the choice explicit per-strategy. Silently defaulting to "unlimited → no check" is how unbounded risk enters production.

## Collateral and buying power

Collateral computation is broker-specific, but the pattern is:

```
required_collateral = compute_per_structure(legs, strikes, quantity)
available           = account.cash + account.marginable_equity - locked_collateral
if required_collateral > available:
    reject(INSUFFICIENT_BUYING_POWER)
```

Per-structure formulas follow broker conventions (Reg T, portfolio margin, etc.). The OMS does not need to replicate the broker's formula exactly — it just needs to be at least as strict, so the OMS rejection happens before the broker rejection. If the OMS accepts and the broker rejects, you've wasted a round-trip and the user sees a confusing error.

**Locked collateral** is the sum of collateral requirements across all open positions in the account. When a position closes, its collateral is released. When a position's risk changes (e.g. underlying moved through a strike), recomputation is deferred until the next order submission — intra-day collateral adjustments are a broker concern, not an OMS concern, for most use cases.

## Entry guardrails beyond max_loss

- **Concentration limits.** No more than N% of account value in one underlying; no more than M% in one sector; no more than K positions per strategy. These are soft limits the user can override with an explicit flag, but the default is to block.
- **Duplicate prevention.** If a strategy already has an open position with the same `(underlying, expiry, strategy)` and the new order would double it, require an explicit "roll" or "add" flag rather than silently stacking.
- **Cooldown.** After a stop-out in a given underlying, block new entries for N minutes/hours. Common for mean-reversion strategies that otherwise re-enter the same losing trade.

## Kill switches (continuous)

Continuous guardrails run on every tick. Canonical set (see `exit_rules.md` for the full priority list):

- **`max_dollar_loss`** — per-tick unrealized loss cap. Fires before `loss_pct` so a concentrated position doesn't need a percentage move to trigger a dollar stop.
- **`max_dollar_profit`** — per-tick unrealized profit target. Useful for ratio spreads with near-zero entry credit where `profit_pct` is meaningless.
- **`delta_exit`** — position is closer to ATM/ITM than risk tolerates.
- **`gamma_exit`** — position is exposed to too much convexity, typically near expiry.
- **Account-level kill switch** — all positions in an account close if the account's daily drawdown exceeds a threshold. This is the emergency brake.

Account-level kills must be explicit and separate from per-position rules. Implementing them as a single "loop over positions and check each one's stop" hides the portfolio-level view.

## Testing risk controls

Every risk control needs three tests:

1. **Boundary test.** Value = threshold → decision X; value = threshold ± epsilon → decision Y.
2. **Interaction test.** Two controls that could fire together → the correct one wins (see `exit_rules.md` tie-breaking).
3. **Replay test.** The same state produces the same decision under replay.

Risk controls are the most safety-critical code in the OMS. Coverage must be exhaustive, not illustrative.
