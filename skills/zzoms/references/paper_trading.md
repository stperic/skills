# Paper Trading — Fill Engine, Slippage, 24/7 Availability

Paper trading is not "live trading minus the broker call." It's a simulated fill engine with its own invariants. The goal is to produce fills that a strategy can reason about consistently across live, paper, and replay modes.

## Availability

**Paper fills run 24/7.** This is load-bearing for:

- Weekend backtests and strategy development.
- Auto-exit rules that fire on time-based triggers (`max_hold_days`, `dte_exit`).
- Replay of after-hours events.

Live fills only occur when the venue is open. A strategy running against both live and paper accounts must handle the asymmetry: paper positions may advance through closes and weekends while live positions sit frozen.

## Fill model

Paper fills use a simple model:

- **Equities:** fill at the current mid (or user-configured side of the spread), minus slippage.
- **Options with live bid/ask:** fill at mid or at the configured side (e.g. sell at bid, buy at ask for worst-case; mid for realistic).
- **Options without live bid/ask:** fall back to Black-Scholes theoretical price using the injected `OptionsPricer`. This is the 24/7 path — when the venue is closed and no quote is available, the BS price is the only honest answer.
- **Multi-leg structures:** each leg is priced independently, then summed for the net debit/credit.

The BS fallback requires inputs: underlying spot, strike, expiry, risk-free rate, and IV. The IV used in the fallback **matters for determinism** — see `replay_determinism.md` for the cached-daily-ATM-IV rule.

## Slippage precedence

Slippage is applied to every paper fill. The precedence chain is **first non-zero wins**:

```
1. slippage_Nleg       — leg-count-specific (e.g. slippage_4leg for an IC)
2. spread_slippage_pct — spread-wide default for multi-leg
3. slippage_pct        — global default
4. 0                   — no slippage (sanity-check / test mode)
```

Configuring all three is legal. The precedence chain is what makes paper replay stable across config migrations: if a team adds `slippage_4leg` for iron condors, existing strategies that rely on `spread_slippage_pct` still work — the new setting only applies where it's specific enough to match.

**Why this chain:**

- **Leg-count-specific first** because iron condors slip differently than verticals differently than single legs; the venue-specific microstructure is at the leg-count layer.
- **Spread-wide next** because spreads generally slip more than single legs (execution has to cross more spreads).
- **Global last** because it's the coarsest default.
- **0 is a sentinel** meaning "no setting applies," not "zero slippage intentionally." Use a tiny positive value (1e-6) if you truly want a zero-slippage test mode to distinguish from unset.

## Slippage direction

Slippage always moves the fill *against* the trader. For single legs:

```
buy_fill_price  = base_price * (1 + slippage_pct)   # pay more
sell_fill_price = base_price * (1 - slippage_pct)   # receive less
```

For spreads, apply to the **net** debit/credit, not per-leg, and be explicit about direction:

```
# debit structure (you pay the net)
net_debit_paid      = base_net_debit  * (1 + slippage_pct)

# credit structure (you receive the net)
net_credit_received = base_net_credit * (1 - slippage_pct)
```

A credit spread with base credit $2.00 and 5% slippage fills at $1.90 received (worse for you), not $2.10. Getting the sign wrong on credit structures is a common bug — strategies look artificially profitable in paper because the slippage was silently helping.

Closing a position uses the symmetric direction: closing a credit structure is a buy-to-close on the net, so `buy_fill_price = base * (1 + slippage_pct)` applies to the net debit you pay to close.

## Paper vs live order validation

Paper and live orders share the same validation pipeline (see `order_lifecycle.md`). The only difference is the fill engine: paper orders route to the paper fill engine; live orders route to the broker adapter. This symmetry is important — a strategy that passes validation in paper should pass in live (pending broker-level checks that the paper engine can't simulate, like halted symbols).

## Paper accounts and the credential gate

Paper accounts only require the base API key, not the live broker key. The gate is per-account, keyed off the account's `provider` field (`paper`, `sim`, `broker_live`). A paper order never reaches the broker adapter, so it never needs broker credentials. See `broker_integration.md` for the credential check pattern.

## Deterministic paper replay

When replay is active (`asOfBoundary` set), the paper fill engine:

1. Uses cached historical quotes from the market-data layer, not live quotes.
2. Uses cached daily ATM IV for BS fallback (see `replay_determinism.md`).
3. Applies slippage from the slippage precedence chain exactly as in live paper mode.
4. Uses the as-of clock (`Clock.Now()` returns the as-of time) for all timestamps.

The result: a paper replay of day D produces bitwise-identical fills across runs, and identical to the paper fills that happened on day D if the inputs (quote cache, IV cache) are unchanged.

## What paper can't simulate

- **Broker rejections** beyond basic entry guardrails (halted symbols, PDT violations, day-trade count).
- **Partial fills** at the microstructure level. The paper engine typically fills the entire quantity at once.
- **Queue position.** A limit order filled in paper would not necessarily fill in live if the market only traded through the price briefly.
- **Session transitions.** Paper fills work across the session boundary; a live order at 15:59:59 might not fill before close.

Strategies that depend on microstructure (queue position, partial fill timing) need a more sophisticated fill model — at which point you're building a backtesting engine, not a paper OMS.
