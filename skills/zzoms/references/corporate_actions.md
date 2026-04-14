# Corporate Actions — Splits, Dividends, Assignment, Expiration

Corporate actions are the category of events where the broker silently changes your position. The OMS must detect, apply, and audit every one — or its position book drifts from reality.

## Categories

| Event | Effect on OMS positions | Frequency |
|---|---|---|
| Stock split | Quantity ×N, basis ÷N, strikes ÷N (for options) | Occasional |
| Reverse split | Quantity ÷N, basis ×N, strikes ×N | Rare |
| Cash dividend (equity) | No position change; cash posts to account | Frequent |
| Cash dividend (option) | Typically no adjustment (OCC rule); rare exceptions | Frequent |
| Special dividend (equity) | Strike adjustment on options (OCC memo required) | Rare |
| Merger cash | Position closed at merger price; cash posts | Occasional |
| Merger stock-for-stock | Position converted to acquirer shares at ratio | Occasional |
| Spinoff | New symbol appears; basis allocated per IRS rules | Occasional |
| Symbol change | Same position, new symbol | Occasional |
| Early assignment | Option position closed; underlying position appears | Frequent in options |
| Exercise | Same as early assignment at expiry | Frequent in options |
| Expiration (worthless) | Option position closed at 0 | Very frequent |
| Expiration (ITM, auto-exercise) | Same as assignment | Frequent |

The frequent ones are the ones most commonly mishandled — assignment, exercise, expiration. A split that happens once a year gets manual attention; an early assignment that happens three times a week is where bugs hide.

## The CorporateActionChecker interface

The OMS depends on a narrow injected interface:

```
type CorporateActionChecker interface {
    ActionsForSymbol(ctx, symbol, since, until) ([]Action, error)
    ApplyAction(ctx, action, position) (adjustedPosition, error)
}
```

The checker is called:

- **Nightly, as part of reconciliation.** All open positions have actions applied for the day that just ended.
- **On fill.** A fill on a symbol with a recent corporate action applies the action to the post-fill position state.
- **During replay.** Replay consults the checker with the as-of date, so historical corporate actions are applied as of the historical moment, not "now."

## Stock splits

The canonical hard case. A 3:1 split turns 1 share at $300 into 3 shares at $100. For options, the **standard OCC adjustment for whole-number forward splits** is:

- **Multiply contract count by the split ratio and divide the strike by the same ratio.** One call at $300 strike becomes three calls at $100 strike, each still on 100 shares. The deliverable (100 shares per contract) is **unchanged**.

**Deliverable adjustments** — where the deliverable becomes something other than 100 standard shares — are reserved for **odd ratios** (e.g. 3-for-2), **special dividends**, and **spinoffs**. For a 3-for-2 split, a single call at strike K typically becomes one call on a non-standard 150-share deliverable at strike K × 2/3, not 1.5 contracts (options contracts are integer).

Which adjustment applies depends on the OCC memo for the specific event. The OMS must read the memo (via the `CorporateActionChecker`), not guess from ratios. Getting the direction wrong is one of the most common real-world mis-books on split day.

**Gotchas:**

- **Basis must adjust.** A position opened at basis $300 for 1 share, after a 3:1 split, has basis $100 for 3 shares. If basis doesn't adjust, realized P&L is wrong by 3x.
- **Exit rule thresholds don't adjust automatically.** A `stop_loss_price = $250` on a pre-split position is nonsense post-split. The OMS must either adjust the threshold (divide by split ratio) or flag it for user review. Silent application is dangerous — the user set $250 based on pre-split prices.
- **Non-standard deliverables persist.** After an odd split or special dividend, an option's deliverable can remain non-standard for the life of every previously-listed strike (often 90+ days of live trading). Exit rule code that keys on `strike` as an underlying-price proxy will mis-fire on those strikes — key on the deliverable or the adjusted strike, not the raw ticker strike.
- **Mini and weekly options have non-100 multipliers.** Even without corporate action adjustments, mini options are 10-share deliverables, and some ETF products have different multipliers. Never hardcode 100.

## Dividends

Cash dividends on equity positions require classifying the distribution before deciding how to book it. **Different distribution types have different effects on basis and P&L** — this is the area where silent incorrect handling quietly corrupts books over time.

| Distribution type | Effect on basis | Effect on realized P&L | Typical source |
|---|---|---|---|
| **Ordinary / qualified dividend** | None | Post to dividend income | Most C-corps |
| **Return of capital (ROC)** | **Reduces basis** | None until basis exhausted; then capital gain | Many REITs, MLPs, closed-end funds, some ETFs |
| **Capital-gain distribution** | None | Booked as long-term capital gain | Mutual funds, some ETFs |
| **Nondividend distribution** (rare) | Reduces basis like ROC | None until basis exhausted | Corporate events |

**The blanket rule "dividends don't touch basis" is wrong.** REITs, MLPs, many ETFs, and any fund publishing a 1099-DIV with ROC components will silently corrupt basis if the OMS treats every distribution as a non-basis event. Classify the distribution using the broker's or issuer's published 1099-DIV box mapping (or a data-provider feed) before applying.

**Option adjustments from special (ordinary) dividends** are rare but material. The OCC issues a memo when a cash distribution exceeds the published threshold; strike adjustments flow through the same `CorporateActionChecker` path as splits.

The OMS should:

- Track dividends in a separate `dividends` table linked to the position, with the distribution type classified.
- Apply ROC distributions as basis reductions in `trading_positions`.
- Include ordinary and qualified dividends in realized P&L (total return, not just capital gain).
- Feed the external tax-lot ledger the raw distributions for its own qualified/non-qualified determination.

## When to expect early assignment

Early assignment is not random. A few patterns dominate, and an OMS that plans for them avoids most of the overnight surprises:

- **ITM short calls on the day before ex-dividend.** If the dividend exceeds the remaining time value of the call, the optimal exercise boundary is crossed and the call holder will exercise to capture the dividend. The OMS should scan every short call against the next-day ex-div schedule and flag positions at risk. The common operational remedy: close or roll the short call before the close on the day before ex-div.
- **Deep-ITM shorts trading at or below parity.** An option trading below its intrinsic value has no remaining optionality; the long side has a free arbitrage by exercising. Shorts of any deep-ITM option near parity should be treated as about to be assigned.
- **Hard-to-borrow shorts with high borrow fees.** For short puts on HTB names, the put holder may exercise early to shed the borrow cost. Less common than the ex-div case but material during squeezes.
- **Pin risk at expiration.** A short option with the underlying closing within a few cents of the strike on expiration day has an ambiguous settlement: the option may or may not be auto-exercised depending on whether it closes ITM by ≥ $0.01, and the long holder may contest or file a contrary exercise instruction. The OMS should flag positions within a pin-risk band (e.g. ±0.5% of strike) at the close and surface them to the operator overnight. Short straddles at the strike on expiration are the canonical example.
- **Overnight cash / buying-power implications.** An ITM short call assigned overnight creates a short equity position at the strike — which needs locate, borrow, and margin the OMS may not have. Surface the expected cash requirement on flagged positions before the close.

The `CorporateActionChecker` (or a sibling `AssignmentRiskChecker`) should expose these scans. The output feeds both operator dashboards and the risk control layer.

## Early assignment

Short option positions can be assigned by the holder at any time up to expiration. The broker notifies the OMS via an exec report that:

- Closes the short option position.
- Opens (or adjusts) the underlying position at the strike price.
- Posts cash.

The OMS receives this as fills on two distinct positions and applies them through the normal `ApplyFill` pipeline. No special code path is needed — as long as the fills carry correct attribution.

**What goes wrong:**

- The broker may send the two fills out of order. The OMS must handle "underlying fill before short option close" gracefully (it's the same end state either way).
- The short option's realized P&L must account for the credit received at entry, not just the fill at strike.
- Exit rules on the short option should not fire between assignment and the OMS processing the fills — use the `group.frozen` or `position.pendingAdjustment` pattern to pause evaluation briefly.

## Exercise (by the OMS user)

The OMS may initiate exercise on long ITM options. This is usually a user-invoked action, not automatic. The OMS:

1. Validates the exercise is possible (ITM, not yet expired, account can take the underlying).
2. Submits an exercise request to the broker.
3. Waits for the broker's confirmation (often a fill pair: option close + underlying open).
4. Applies the fills through the normal pipeline.

**Auto-exercise at expiry** is handled by the broker per OCC rules: any option $0.01 ITM at close is auto-exercised unless the account specifies otherwise. The OMS receives the resulting fills the next morning and applies them.

## Expiration

Options expire worthless (OTM) or are auto-exercised (ITM). The OMS must:

- **Close expired OTM options** at $0 realized P&L. Entry credit becomes full profit; entry debit becomes full loss.
- **Route expired ITM options** through auto-exercise, producing underlying fills the next day.
- **Never use wall clock for the expiration check during replay.** See `replay_determinism.md` — premature expiration resolution is a recurring bug.

Timing: an expiring option is still tradable until 16:00 ET on expiration day (for standard expirations). The OMS's expiration sweep runs *after* the close, not during regular hours. Running it during regular hours can close positions that the user intended to manage manually in the final minutes.

## Symbol changes and mergers

A symbol change is mechanical: update every row in `trading_positions`, `orders`, and `fills` with the new symbol, preserve basis and history, link old and new via a `symbol_history` table. Nothing else changes.

A stock-for-stock merger is harder: the target symbol is replaced with a quantity of the acquirer symbol at a ratio. The OMS must:

1. Close the target position at the merger price (realized at the price, not the market).
2. Open the acquirer position at the ratio-derived quantity and basis.
3. Preserve the link so tax-lot reporting can trace the chain.

Cash mergers are simpler: close at the cash price, book realized P&L, done.

## Audit and history

Corporate actions are rare enough that manual review is viable. Persist every action to a `corporate_actions_applied` table with:

- Action type and details (split ratio, merger price, etc.).
- Positions affected and their pre/post state.
- Timestamp and source (broker feed, manual entry, `CorporateActionChecker`).

At quarter-end or year-end, review the log to catch missed actions. The OCC memos and SEC filings are the ground truth; the OMS log is the applied-to-me view. Gaps mean missed events.

## Implementation status awareness

Corporate actions are the category most commonly "partially implemented" in a production OMS. Splits and simple symbol changes usually work; mergers, spinoffs, and odd splits are often hand-patched as they occur. Track open gaps in a known-issues list so oncall knows "splits land Tuesday, watch the account manually."
