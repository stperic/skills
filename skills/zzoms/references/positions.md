# Positions — Two-Tier Bookkeeping, Uniqueness, Cost Basis

Position bookkeeping is the most frequently-corrupted layer in an OMS. The pattern below minimizes the surface area for corruption by making positions **derived** from fills rather than written directly.

## Two tables, two purposes

| Table | Database | Purpose | Who writes |
|---|---|---|---|
| `positions` | Main app DB | Lightweight user-facing portfolio notes (tags, strategy label, notes) | Users / UI |
| `trading_positions` | OMS DB | Auto-derived from the fill stream: cost basis, quantity, realized/unrealized P&L, structure | OMS only, via `ApplyFill` |

The user-facing `positions` table is purely informational — deleting a row doesn't delete the underlying trading activity. The OMS table is the source of truth for P&L and risk.

**Rule: no code writes `trading_positions` outside the fill pipeline.** If you need to correct a position, insert a corrective fill. If you need to reset, reset via the administrative `ResetAccount` path (see `repository_patterns.md`).

## Uniqueness

`trading_positions` has a composite unique constraint:

```
UNIQUE (account_key, symbol, instrument_type, strike, expiry, position_side, strategy)
```

Every column is load-bearing:

- **`account_key`** — multi-account support; the same leg in two accounts is two rows.
- **`symbol`** — underlying ticker.
- **`instrument_type`** — `EQUITY`, `CALL`, `PUT`, `FUTURE`, etc.
- **`strike`** — NULL for equities, NOT NULL for options.
- **`expiry`** — NULL for equities, NOT NULL for options.
- **`position_side`** — `LONG` or `SHORT`. Critical because a long call and a short call at the same strike/expiry are distinct positions for cost basis and exit-rule purposes.
- **`strategy`** — a strategy label that lets multiple strategies share the same leg without cross-contaminating cost basis. A user running both a short-strangle strategy and a covered-call strategy on the same underlying tracks them separately.

**Never add a UUID column to this uniqueness.** The natural key is what makes position sorts stable across runs (see `replay_determinism.md`).

## Cost basis computation — internal running average

Cost basis in `trading_positions` is a **running average** for internal P&L display and exit-rule evaluation. **It is not a tax-lot engine.** Tax-lot accounting (specified-lot, FIFO/LIFO/HIFO, wash sale, Section 1256) is a separate ledger layer that sits on top of the fill stream. The OMS must preserve enough information for that layer — open/close tagging, per-fill price and timestamp, lot-selection hooks — but the running average in `trading_positions` is for internal display, not for tax reporting.

Running-average rule on every fill:

```
on BUY to open or add:
    new_qty     = qty + fill_qty
    new_basis   = (qty * basis + fill_qty * fill_price) / new_qty
    qty, basis  = new_qty, new_basis

on SELL to close or reduce:
    realized_pnl += (fill_price - basis) * fill_qty * multiplier
    qty          -= fill_qty
    basis         unchanged  (basis is for remaining shares/contracts)

on position flip (SELL through zero):
    close-out realized P&L for the original side
    remainder opens the opposite side at fill_price
    this requires TWO trading_positions rows (one closing, one opening)
```

Multiplier is 100 for standard equity options (but see post-split non-standard multipliers and mini/weekly variants in `corporate_actions.md`), 1 for equities, varies for futures.

**Gotchas:**

- **Short fills reverse the sign.** A sell-to-open at price P has cost basis P (credit received); a buy-to-close at price Q realizes `(P - Q) * qty * multiplier` — notice the reversed sign vs the long case.
- **Fee attribution.** If the OMS tracks fees, include them in realized P&L but not in cost basis. A common bug is adding fees to basis, which silently shifts the unrealized P&L.
- **Partial closes don't re-average.** Basis only changes on adds.

## Spread grouping

Positions are grouped into spread structures (vertical, iron condor, strangle, calendar, etc.) by `GroupPositionsBySpread`. Two rules that recur in production:

1. **Return an ordered slice, never a map.** Map iteration is randomized per process; any map at this layer corrupts replay determinism. See `replay_determinism.md`.
2. **Grouping key is `(account_key, underlying, strategy, expiry_bucket)`** — positions that share this tuple are candidates for spread grouping. The actual structure detection (vertical vs IC vs strangle) happens inside each group via the detector chain (see `structure_detection.md`).

## Position updates are transactional with fills

`ApplyFill` is a single transaction touching:

1. `fills` — insert the new fill.
2. `orders` — advance the order state.
3. `trading_positions` — upsert the position row.
4. `accounts` — decrement collateral / update realized P&L.

Partial success is never observable. If step 3 fails (e.g. unique constraint violation), the whole transaction rolls back and the fill is not applied. The next retry (or reconcile) will re-apply cleanly.

## Closed positions

Closed positions (`quantity = 0`) are **not deleted**. They remain in `trading_positions` for:

- Realized P&L reporting.
- Strategy attribution and backtesting.
- Feeding the external tax-lot ledger (the OMS does not compute tax lots itself — see the scope note in SKILL.md — but it preserves the raw data).

The default query for "open positions" filters `quantity != 0`. A common bug is forgetting the filter in a new handler, which inflates position counts by 10-100x.

## Position repo default sort

The default sort for position queries must be stable across runs. Canonical sort:

```
ORDER BY opened_at DESC,
         account_key, symbol, instrument_type, strike, expiry, position_side, strategy,
         open_seq ASC
```

The natural-key columns are the unique constraint (minus `opened_at`). `open_seq` is a **monotonic, deterministically-assigned** sequence derived from the fill stream — not a UUID. It disambiguates the case where the same logical position is reopened at the same millisecond (e.g. bulk re-open after a reset, or two legs of a roll that land on the same tick).

**Never append `id ASC` where `id` is a UUID.** UUIDs are per-run random and break replay determinism. The natural key plus a monotonic seq is the right tiebreaker. See `replay_determinism.md`.
