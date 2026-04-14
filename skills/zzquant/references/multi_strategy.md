# Multi-Strategy Portfolio Orchestration

## Why this file exists

Running N strategies in one book is not the same as running N independent strategies. Correlation, shared risk factors, overlapping positions, and resource contention create problems that none of the individual strategy references address. This file covers the orchestration layer: capital allocation, risk budgeting, correlation management, strategy lifecycle, and the kill-switch discipline that prevents one failing strategy from taking down the book.

## The three goals of orchestration

1. **Diversification** — uncorrelated return streams reduce portfolio vol; a multi-strategy book should have Sharpe ≥ max(individual Sharpes), otherwise diversification isn't working
2. **Capital efficiency** — limited capital must be allocated across strategies by some principled rule
3. **Blast containment** — one strategy's failure must not cascade to others

All three are load-bearing. Drop any one and the multi-strategy book degrades to "a basket of strategies with correlation you didn't model."

## Capital allocation methods

### Equal weight
- Simplest baseline: `w_i = 1/N`
- No view on strategy quality; pure diversification
- Good default when you don't trust the estimates (every other method uses inputs that are themselves noisy)

### Inverse volatility
- `w_i = (1/σ_i) / Σ(1/σ_j)`
- Normalizes risk contribution across strategies
- Standard for multi-trend-following books
- Estimation error on σ is much smaller than on return

### Risk parity
- Each strategy contributes equal risk to the portfolio, accounting for correlations
- `risk_contribution_i = w_i · (Σw)_i`
- Iterative solver required (cvxpy or closed-form for specific covariance structures)
- Preferred over inverse-vol when strategy correlations are non-trivial

### Mean-variance optimization (MVO)
- Markowitz optimal: maximize Sharpe given return and covariance estimates
- In practice: noise amplifier. Small errors in expected returns produce extreme allocations.
- Usable only with heavy shrinkage (Ledoit-Wolf, Black-Litterman priors) and hard weight bounds
- Most practitioners prefer risk parity or inverse vol for robustness

### Kelly fraction
- `f_i = μ_i / σ_i²` for a single strategy
- Multi-strategy Kelly: `f = Σ⁻¹ · μ`
- Theoretically optimal growth; practically too aggressive — full Kelly is painful to live through
- "Fractional Kelly" (0.25–0.5× Kelly) is standard

### Regime-conditional allocation
- Allocate based on current regime (see `regime_detection.md`)
- Strategy weights change as regime changes; soft via state probability, hard via state assignment
- Requires robust regime detection; whipsaw risk

### Online learning / regret minimization
- Exponential weights (Hedge algorithm, Prod algorithm)
- Online Newton Step (Hazan et al.) for Sharpe-based allocation
- Theoretically robust (no statistical assumptions); practically slow to adapt

### Manager-of-managers heuristic
- Start with equal weight
- Over time, trim strategies with rolling drawdown > X or Sharpe < Y
- Add strategies that clear OOS tests
- Simple, robust, used by more multi-manager shops than will admit it

## Correlation management

Strategies that *look* uncorrelated in backtest can spike together in stress. Defenses:

### Rolling correlation monitor
- Track rolling N-day correlation of strategy return streams
- Alert when correlations exceed threshold (e.g., two strategies that should be uncorrelated showing ρ > 0.6 over 30 days)
- Common cause: both strategies became short-vol / long-momentum / long-duration without intending to

### Factor exposure aggregation
- Decompose each strategy's returns into factor exposures (MKT, SMB, HML, UMD, VIX, DV01, etc.)
- Aggregate at portfolio level
- A book with two "uncorrelated" strategies both long momentum and both short vol is a concentrated momentum+vol book pretending to be diversified

### Cross-strategy position overlap
- Do multiple strategies hold the same names on the same side?
- Equity stat arb + factor tilt often overlap on value names; a drawdown in value hits both
- Monitor: gross concentration at the symbol level, not just the strategy level

### Correlation spike protocol
- Define a trigger: "if any pair of strategies shows ρ > 0.8 over 20 days, alert; if > 0.9, halve both"
- Pre-committed response beats ad-hoc in stress

## Risk budgeting

Allocation by dollar weight is wrong when strategies have different vol. Allocate *risk*, not *capital*.

### Portfolio risk budget
```
target_vol = σ_portfolio_target  (e.g., 12% annualized)
```
Scale gross exposure so that the sum of risk contributions equals target_vol.

### Per-strategy risk caps
- **Max Var contribution**: no strategy > X% of total portfolio variance
- **Max gross exposure**: no strategy > X% of NAV
- **Max drawdown allocation**: if strategy is at worst historical drawdown, it can't be more than Y% of NAV

### Risk scaling
- Vol targeting: scale strategy size by `target_vol / realized_vol`
- Keeps strategy contribution stable as its vol changes
- Smooths with EWMA to avoid over-reaction to noise

## Kill switch discipline

Every multi-strategy system must have at least four levels of kill switch:

### Level 1: Per-trade
- A single trade cannot exceed `max_loss_per_trade`
- Enforced by OMS, not by strategy code
- Fail-safe: if validation is absent, order is rejected

### Level 2: Per-strategy
- A strategy that hits drawdown `X%` is halted (no new entries, existing positions managed to close only)
- Parameters: drawdown threshold, rolling window
- Auto-halt is the default; manual re-enable after review

### Level 3: Correlated group
- Groups of strategies with similar exposures (all short premium, all momentum) are scaled down together when group drawdown > Y%
- Prevents "the short-vol book just lost 20% but they're all still running full size"

### Level 4: Book-wide
- Total portfolio drawdown > Z% → halt all new entries, unwind by priority
- This is the "unplug everything" switch
- Tested quarterly; run a drill

## Strategy lifecycle

### Onboarding
- New strategy starts at 10% of target size or less
- Paper trade in parallel with live production first (parallel simulation)
- Promotion criteria: 30+ days of live trading with consistent behavior vs backtest, OOS Sharpe within tolerance of IS Sharpe
- Gradual ramp to target size over 60–90 days

### In-production monitoring
- Rolling live-vs-backtest deviation: if live Sharpe diverges from backtest Sharpe by > X, investigate
- Regime classification: is the strategy in its expected regime?
- Parameter drift: if the strategy uses LLM or adaptive parameters, is the drift bounded?

### Retirement
- Drawdown past threshold → halted
- Halted strategy reviewed: broken? decayed? regime unfavorable but still valid?
- Clear criteria for resurrecting vs retiring: N consecutive months of flat or negative live P&L without an attribution story = retire

## Resource contention

Multiple strategies competing for the same shared resources cause subtle bugs:

- **Same symbol, opposite sides** — two strategies want to buy and sell the same name same day. Net to one order or cross internally? Document the rule.
- **Same symbol, same side** — can overshoot position caps. Global position check before order.
- **Buying power** — one strategy consumes all available BP, another can't enter. Pre-allocate BP per strategy.
- **Margin** — options strategies can have margin changes on existing positions affecting available margin for new strategies.
- **Borrow** — for shorts, who gets the available borrow? First-come or pre-allocated?

Resolve via a **central risk/OMS layer** that sees all strategies and enforces global constraints. Strategies submit *intents*; the central layer approves, modifies, or rejects.

## Execution order

When multiple strategies fire signals at the same moment:
- **Priority**: document it (e.g., "stop-losses before new entries", "exits before adds")
- **Netting**: if two strategies offset at the same symbol, net them (or cross internally — saves spread + commission)
- **Sequencing**: if capital is limited, which strategy gets allocated first?

A common retail bug: two strategies submit orders in parallel, both assume they have full BP, both get partial fills, neither is at target size. The fix is a *serialized* order gate: one order-approval path for the whole book.

## Attribution

Multi-strategy books must support per-strategy P&L attribution:
- Tag every position with its originating strategy
- Compute per-strategy P&L (realized + unrealized)
- Reconcile against account-level P&L (they must sum)

Without attribution, you can't tell which strategy is dragging, you can't fire decisions, and you can't maintain the allocation algorithm. Attribution is non-negotiable.

## Scaling considerations

What changes as the book grows:

| Size | What matters most |
|---|---|
| < $100K | Defined-risk only, commissions dominate, per-strategy kill switches |
| $100K–$1M | Multi-strategy orchestration, correlation monitoring, basic risk parity |
| $1M–$10M | Factor-aware allocation, transaction cost modeling, proper TCA |
| $10M–$100M | Market impact matters, venue selection, prime brokerage relationships |
| > $100M | Capacity constraints per strategy, risk team, formal ops |

The architecture should be the same at any scale; the inputs and constraints tighten as capital grows.

## Risk controls summary

1. **Global BP tracker** — one source of truth for available capital
2. **Pre-trade global risk check** — orders validated against portfolio-level constraints, not just strategy-level
3. **Correlation monitor** — alerts on cross-strategy correlation spikes
4. **Kill switch hierarchy** — per-trade, per-strategy, per-group, book-wide
5. **Attribution** — daily, per-strategy, reconciled to account
6. **Parameter bounds** — enforced at a single point, not in each caller
7. **Drawdown de-risking** — automatic size reduction as drawdowns deepen
8. **Drill** — quarterly test of the kill switch hierarchy against a simulated blow-up
9. **Haircut-expansion buffer** — prime brokers raise haircuts on volatile positions by 2–3× during stress events, *exactly when you need capacity to manage the book*. Several short-vol funds blew up in March 2020 from forced de-grossing rather than realized losses. Maintain a liquid cash buffer sized to expected stress-haircut expansion (1.5× to 2× current margin); stress-test under expanded haircuts; diversify prime brokers at scale to survive single-broker policy changes.

## References

- Grinold & Kahn — *Active Portfolio Management* (risk budgeting, factor neutral)
- Litterman — *Modern Investment Management* (institutional multi-manager frameworks)
- Meucci — *Risk and Asset Allocation* (risk budgeting, Black-Litterman)
- Qian — "Risk Parity and Diversification" (Panagora)
- Clarke, de Silva, Thorley — "Minimum-Variance Portfolios in the U.S. Equity Market"
- Kelly (1956) — original Kelly criterion paper
- Thorp — "The Kelly Criterion in Blackjack, Sports Betting, and the Stock Market"
- Hazan — *Introduction to Online Convex Optimization* (regret-based allocators)
