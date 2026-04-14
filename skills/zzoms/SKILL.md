---
name: zzoms
description: Senior OMS (Order Management System) software architect and analyst. Covers the design, construction, and review of production OMS platforms supporting paper and live broker trading. Topics include package segregation and DI, order lifecycle and state machines, pre-trade risk controls (collateral, buying power, max-loss entry caps), paper fill engines and slippage models, broker adapter patterns (token refresh, circuit breakers, external order detection, idempotency by client order id), position bookkeeping (derived-from-fills, cost basis, uniqueness), spread structure detection (vertical/IC/strangle/calendar/diagonal chains), exit rule engines (fixed tie-breaking with kill switches first, per-position normalized delta/gamma, RTH gating, hot-path Greeks gates), Greeks plumbing (shared IV/TTE inputs, replay-vs-live drift), replay/simulation determinism (as-of boundary, ordered grouping, stable natural-key sort, background-loop gating, cross-run bitwise reproducibility), reconciliation (nightly broker diff, phantom position detection, drift alerts), corporate actions (splits, dividends, assignment, expiration), strategy groups (rolls, atomic multi-repo adjustments), observability and alerting, and the recurring operational traps that bite OMS teams. Use when designing an OMS from scratch, reviewing an order/fill/exit pipeline, debugging replay-vs-live drift, adding an exit rule, structuring a broker integration, hardening reconciliation, or answering "is this OMS architecture sound?". For AlphaDB-specific file paths, HTTP API, and MCP tool catalog, use `zzalpha` instead — this skill is product-agnostic design knowledge, not a project reference.
---

# zzoms — OMS Software Architect & Analyst

Durable, product-agnostic domain knowledge for building, reviewing, and operating production Order Management Systems that support both paper and live broker trading, single-leg and multi-leg options, and deterministic replay for backtesting. Project-specific file paths, build commands, and broker-specific credentials belong in each project's `CLAUDE.md`.

## When to use

Invoke whenever you are:

- Designing an OMS from scratch or adding a new capability to one
- Reviewing an order/fill/exit pipeline for soundness
- Debugging a replay-vs-live divergence
- Adding or tuning an exit rule on a closing-strategy spec
- Wiring a new broker adapter or hardening an existing one
- Hardening nightly reconciliation, phantom-position detection, or corporate-action handling
- Structuring the repository/transaction boundaries of the OMS data layer
- Answering "would an institutional trading team ship this OMS?"

For questions tied to a specific project's file layout, HTTP surface, or MCP tool catalog (e.g. AlphaDB), use the project-specific skill (`zzalpha`) — this skill deliberately avoids product-level details.

## Scope and explicit non-goals

This skill covers the **engineering** of an OMS: state machines, data layer, replay determinism, exit-rule engines, broker integration, reconciliation, observability.

**Explicitly out of scope** — named here so they are not silently omitted. A real production OMS needs all of these; this skill does not teach them:

- **Tax-lot accounting.** Specified-lot / FIFO / LIFO / HIFO selection, wash-sale disallowance (IRC §1091), Section 1256 60/40 mark-to-market, constructive sale (§1259), straddle rules (§1092). The OMS must *preserve* enough data to support a tax-lot layer (open/close tagging, lot-selection hooks, wash-sale window scans) but tax-lot computation itself is a separate ledger.
- **Short inventory and locate.** Reg SHO locate requirements, hard-to-borrow fees, recall/buy-in risk, `sell-short` vs `sell-to-close`. Any equity-shorting OMS needs this layer.
- **Margin mechanics.** Reg T initial vs maintenance, portfolio margin, house requirements, SMA, margin-call ingestion and resolution, PDT rules, day-trading buying power. The skill references collateral and buying power but does not teach the broker-specific formulas.
- **Trading halts, LULD, SSR, LOPR.** Halt handling (live risk with no price discovery), LULD bands silently rejecting limit prices, Reg SHO Rule 201 short-sale restrictions, Large Options Position Reporting compliance.
- **Settlement and cash management.** T+1 settlement, unsettled-cash tracking, good-faith violation, free-riding (Reg T §220.8), journal/sweep, multi-bucket cash ledger (cash, settled cash, SMA, buying power, maintenance excess).
- **Order-type taxonomy beyond the basics.** OPG (opening cross), MOC/LOC (closing auctions), bracket/OCO/OTO, reserve/iceberg, native stop-limit vs OMS-simulated, trailing stops at broker vs OMS, odd-lot semantics. The lifecycle file covers DAY/GTC/IOC/FOK only.
- **Non-US-equity instrument classes.** Futures (SPAN margin, variation margin posting, FCM distinctions, session rollover), crypto (24/7 *for real*, no settlement, fee-in-base-vs-quote), fractional shares (breaks `quantity integer`), fixed income, FX. The skill assumes US equities and US-listed options.
- **Post-trade allocation.** Block-and-allocate models (average-price, pro-rata, partial-fill allocation) for multi-account institutional flow. Single-account OMSs can skip this; institutional cannot.

When a task touches any of the above, flag it explicitly rather than improvising.

## How to navigate this skill

Each topic is a self-contained reference file. Open only the ones relevant to the current artifact.

| Topic | File | When to open |
|---|---|---|
| Package segregation, DI via injected interfaces, service breakdown, separate OMS database | `references/architecture.md` | Any structural change, new package, new service, DI wiring |
| Order lifecycle state machine, validation, fills, amendments, cancels, submitting status | `references/order_lifecycle.md` | Touching order service, fill pipeline, state transitions, idempotency |
| Two-tier position bookkeeping (user notes vs auto-derived), cost basis, unique constraints | `references/positions.md` | Position repo, P&L computation, position uniqueness, upserts |
| Spread structure detection chain (vertical → IC → strangle → ...), first-match precedence | `references/structure_detection.md` | Detecting multi-leg structures, adding a new structure, grouping legs |
| Paper fill engine, slippage precedence and direction, BS fallback, 24/7 availability | `references/paper_trading.md` | Paper fills, slippage configuration, deterministic paper replay |
| Generic exit rule registry, Kit wrappers, tie-breaking, normalization, RTH gating | `references/exit_rules.md` | Adding/editing exit rules, debugging rule priority, delta/gamma thresholds |
| Greeks plumbing, shared IV/TTE inputs, hot-path Greeks gate, replay/live drift | `references/greeks_plumbing.md` | Adding a Greeks-dependent exit rule, investigating replay-vs-live Greeks drift |
| Replay determinism contract: as-of boundary, ordered grouping, stable sort, bg-loop gating | `references/replay_determinism.md` | Replay path, as-of date, simulation service, any bitwise reproducibility concern |
| Broker adapter patterns: token refresh, circuit breaker, external order detection, idempotency | `references/broker_integration.md` | Live broker wiring, auth flows, reconnect logic, detecting externally-placed orders |
| Pre-trade validation (collateral, BP, max-loss entry cap), distinct from continuous $ guardrails | `references/risk_controls.md` | Entry gating, continuous $ kill switches, collateral math |
| Strategy groups: multi-leg tracking, rolls, adjustments, cross-repo atomicity | `references/strategy_groups.md` | Roll logic, adjustments, atomic multi-repo updates |
| Nightly broker reconciliation, phantom detection, drift alerts, fill replay vs broker book | `references/reconciliation.md` | Nightly recon, detecting missing/extra fills, drift remediation |
| Corporate actions: splits (OCC adjustment conventions), dividends (including ROC), assignment, expirations | `references/corporate_actions.md` | Any code touching fills/positions across a corporate event |
| Repository patterns: single-field DBTX vs Beginner, `WithTx(tx)`, cross-service tx runner, admin-only ops | `references/repository_patterns.md` | OMS data layer, transaction boundaries, administrative bulk ops |
| Observability: alert taxonomy, reconciliation drift, submitting stuck, circuit breaker state, deployment verification | `references/observability.md` | Designing a new alert, investigating a stuck `SUBMITTING` order, post-incident runbook writing |
| Operational traps: amend vs new, fill ordering under reconnect, DST in expiry, premature expiry, stale deployed binaries | `references/operational_traps.md` | Post-incident review, hardening checklist, "why does this bug keep coming back" |

If unsure which file to open, start with the one that matches the **concrete artifact** you are touching.

## Hard rules (do not violate)

These are the invariants that get broken most often. Each is one sentence; full context lives in the cited reference.

1. **OMS is architecturally segregated.** No code outside the OMS package imports from it; coupling is via a small set of injected interfaces. See `architecture.md`.
2. **OMS owns its own database.** Trading state never shares a schema with market-data or user-facing application data. See `architecture.md`.
3. **One DB handle per repo, tx-awareness at the field.** A repo holds a single database handle — never a pool plus a separate tx-scoped handle. Repos that need their own atomicity carry a handle type that can begin a transaction; `WithTx` accepts a concrete transaction, never a pool or an interface that could be a pool. See `repository_patterns.md` for the Go + pgx implementation.
4. **Administrative bulk operations refuse nested transactions.** `ResetAccount` and similar return a sentinel error if called on a tx-scoped repo copy. See `repository_patterns.md`.
5. **Replay is cross-run bitwise-reproducible under an active as-of boundary**, resting on three invariants: ordered spread grouping (slice, never map), stable natural-key sort plus a monotonic deterministic tiebreaker (never a UUID), and background loops that skip ticks while replay is active. See `replay_determinism.md`.
6. **Exit rule tie-breaking is fixed, and kill switches fire first**: `Expiration > GammaExit > DeltaExit > MaxDollarLoss > StopLoss > LossPct > EODExitTime > DTEExit > MaxHold > IntradayProfitPct > MaxDollarProfit > TakeProfit > ProfitPct > TrailingStop`. See `exit_rules.md` for the rationale and the time-vs-P&L caveat.
7. **Delta/gamma thresholds are per-position normalized and size-invariant.** Canonical formula lives in `exit_rules.md`; all other files cite it by reference.
8. **`max_loss` is a pre-trade entry cap only.** Continuous dollar guardrails are `max_dollar_loss` / `max_dollar_profit`. See `risk_controls.md`.
9. **Slippage precedence for paper fills** (first non-zero wins): leg-count-specific → spread-wide → global → 0. Direction: debits slip up, credits slip down. See `paper_trading.md`.
10. **Time and date correctness.** Use an `Eastern` day-boundary helper; never `t.Truncate(24h)` for midnight; never `t.Add(±24h)` to step dates (DST days are 23h or 25h). Order expiry checks, EOD, and `max_hold_days` are the usual victims.
11. **Greeks hot-path gate is non-negotiable.** A per-position `NeedsGreeks` flag (set only when a Greeks-dependent rule is configured) gates all per-tick Greeks computation. See `greeks_plumbing.md`.

Note: rule 10 duplicates zzalpha rule 8 by design — both skills must stay in sync.

## Cross-cutting principles

Worth internalizing before any OMS work:

1. **The OMS executes orders; strategies decide which orders.** Validation, routing, tie-breaking, atomicity belong in the OMS. "Should I enter this trade?" does not.
2. **Order state is a strict state machine.** `NEW → SUBMITTING → SUBMITTED → PARTIALLY_FILLED → FILLED | CANCELLED | REJECTED | EXPIRED`. Transitions are idempotent. Reconnect does not resurrect terminal orders.
3. **Fills are authoritative for positions.** Derived tables (e.g. `trading_positions`) are computed from the fill stream; never written directly by ad-hoc code. Corrections are corrective fills.
4. **Idempotency by client order id.** Every order carries a client-generated key; it is the OMS primary identifier and the broker dedup key.
5. **Reconcile is the source of truth for live accounts.** Nightly recon against the broker is the safety net for everything the OMS missed (external orders, reconnect drops, adjustments).
6. **Paper fills run 24/7; live fills run only when the venue is open.** Load-bearing for auto-exit, premium decay, weekend backtests.
7. **Structure detection is first-match, never scoring.** Precedence chain is fixed; adding a new structure means inserting at the right precedence, not building a score.
8. **Replay Greeks have known drift vs live** (cached daily ATM IV → OTM gamma overstated). Tune Greeks-based thresholds against replay numbers.
9. **"Amend" is a lie.** Most broker APIs implement it as cancel+new under the hood. Design the OMS around that reality.

## Operating rules for this skill

1. **Topic-first.** Open the reference by the artifact you're touching, not by complexity tier.
2. **Don't inline a reference into SKILL.md.** If a topic file is relevant, read it; don't paraphrase from memory.
3. **Don't fabricate thresholds or broker quirks.** Slippage defaults, credit-stop multipliers, recon drift bps, circuit-breaker thresholds, and broker amend/partial-fill semantics all differ per venue. If a number isn't in a cited source, flag it as opinion and verify against the specific broker's API docs.
4. **Project CLAUDE.md overrides this skill.** If a project encodes a different rule (custom tie-breaking, custom slippage chain, custom recon schedule), the project wins.
5. **Distinguish invariant from convention.** An invariant is something whose violation corrupts state or determinism (the hard rules above). A convention is house style. Label them differently in review comments.
6. **Prefer structural fixes over patches.** If a bug keeps recurring in the same place, the repository shape, state machine, or interface boundary is probably wrong. Patching the symptom is how OMS codebases rot.
7. **If a task touches an explicitly-out-of-scope topic, flag it.** Tax lots, short inventory, margin calls, halts, settlement, non-basic order types, and non-US-equity instrument classes are not covered here — name the gap rather than improvising.

## Canonical sources

Books
- Larry Harris — *Trading and Exchanges: Market Microstructure for Practitioners*
- Kissell — *The Science of Algorithmic Trading and Portfolio Management*
- Sheldon Natenberg — *Option Volatility and Pricing* (Greeks, pricing, early-assignment logic)
- Euan Sinclair — *Option Trading* and *Volatility Trading*
- David Aronson — *Evidence-Based Technical Analysis* (backtest discipline)

Standards and protocols
- FIX protocol specification — https://www.fixtrading.org/standards/ (order state machine, exec reports, order-cancel-replace semantics, `ExecID` tag 17 dedup)
- FIX Orchestra — machine-readable order workflow descriptions
- OCC — corporate action adjustment rules and memos (https://www.theocc.com/)
- CBOE rules on early assignment, pin risk, automatic exercise thresholds
- SEC Rule 15c3-5 (market access rule) — pre-trade risk control obligations
- Reg SHO — short-sale locate and SSR (Rule 201)
- FINRA Rule 4210 — margin requirements

Papers
- Almgren & Chriss (2001) — *Optimal Execution of Portfolio Transactions*
- Avellaneda & Stoikov (2008) — *High-frequency Trading in a Limit Order Book*
- Kyle (1985) — *Continuous Auctions and Insider Trading* (adverse selection framing)

Reference implementations
- QuantConnect Lean — https://github.com/QuantConnect/Lean (full OMS + backtest + live, C#)
- Zipline — https://github.com/quantopian/zipline (archived; the order/fill/slippage model is canonical)
- Alpaca SDKs — https://alpaca.markets/docs/ (clean REST OMS semantics for reference)
- Interactive Brokers TWS API — grandfather of broker API quirks; study amend/cancel semantics before designing your own

Ongoing references
- FIX Trading Community — https://www.fixtrading.org/
- Nasdaq market structure research — https://www.nasdaq.com/solutions/nasdaq-economic-research
- tastytrade Learn Center — https://tastytrade.com/learn/ (options management rules and operational reality of short premium)
