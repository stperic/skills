---
name: zzalpha
description: AlphaDB + OMS domain knowledge. AlphaDB is an MCP server that gives LLM trading agents a unified interface to market data (Polygon, Alpha Vantage, FRED, tastytrade, Schwab, SearXNG) and computes derived analytics (IV rank/percentile, skew, IV/RV spread, sentiment, beta). The OMS layer handles paper + Schwab live trading with exit rules, Greeks plumbing, replay determinism, and strategy groups. Use when writing, reviewing, or debugging AlphaDB HTTP endpoints, the 76+ AlphaMCP tools, OMS order/fill/exit flow, closing strategy rules, replay/as-of-date, provider proxies, or the nightly sync DAG. Covers the HTTP API reference, MCP tool catalog, and OMS invariants.
---

# zzalpha — AlphaDB + OMS Domain Skill

Unified reference for AlphaDB (the MCP gateway + computed-insights database) and the segregated OMS (paper + Schwab live trading). Project-specific file paths and build commands live in `CLAUDE.md`; this skill is the durable domain knowledge that outlives refactors.

## When to use

Invoke whenever you are:

- Touching any `/v1/*` HTTP endpoint or adding/editing an AlphaMCP tool
- Working on OMS order lifecycle, fills, positions, spreads, or strategy groups
- Adding or tuning an exit rule (`stop_loss`, `delta_exit`, `gamma_exit`, `max_dollar_loss`, etc.)
- Debugging replay determinism, as-of-date, or `SimulationService.ReplayDay`
- Wiring a new provider proxy, cache layer, or auth flow
- Adding a nightly sync job, scheduler entry, or verifier
- Answering "what does tool X return?" or "which endpoint serves Y?"

## How to navigate

Each topic is a self-contained reference file. Open only what the task touches.

| Topic | File | When to open |
|---|---|---|
| Layers, proxy pattern, OMS segregation, nightly sync DAG, River job queue | `references/architecture.md` | Any structural change, new package, new job, DI wiring |
| Full HTTP API (routes, auth tiers, request/response shapes, error codes) | `references/http_api.md` | Editing a handler, adding an endpoint, client integration |
| All 76+ AlphaMCP tools (schemas, parameters, batch mode, sort/filter knobs) | `references/mcp_tools.md` | Adding or editing an MCP tool, scan filter, or tool handler |
| OMS order lifecycle, accounts, fills, positions, spread detection, paper fills | `references/oms_trading.md` | Order service, fill pipeline, position bookkeeping, structure detection |
| Closing strategy fields, tie-breaking order, Greeks plumbing, live vs replay drift | `references/oms_exit_rules.md` | Adding/editing exit rules, `ExitRule[C]` registry, `SpreadExitKit.NeedsGreeks` gate |
| `SimulationService.ReplayDay` determinism invariants, `asOfBoundary`, ordering rules | `references/oms_replay.md` | Replay path, as-of-date, background loop gating, deterministic ordering |
| IV rank/percentile, skew (25d risk reversal), IV/RV spread, sentiment, beta, correlation | `references/analytics.md` | Computed-insight endpoints under `/v1/alpha/*` |
| Auth key tiers, as-of-date admin, deployed-binary verification, LXC/SSH operations | `references/operations.md` | Deploy verification, prod DB checks, auth debugging |

If unsure which file to open, start with the one that matches the **concrete artifact** you're touching (editing an MCP tool → `mcp_tools.md`; tweaking `ExitEvaluator` → `oms_exit_rules.md`).

## Hard rules (do not violate)

These are the invariants that get broken most often. Full context in the per-topic files; the condensed rules belong here because violating them silently corrupts data or determinism.

1. **DB stores computed insights, not raw market data.** Bars/quotes/chains are served live from Polygon/Schwab/tastytrade via proxy. Do not add tables that mirror upstream OHLC.
2. **CLI goes through HTTP.** All CLI commands call the HTTP API via `internal/cli/client/`. Never direct DB access from the CLI.
3. **OMS is segregated.** No package outside `internal/oms/` imports from it. Coupling is via a small set of injected interfaces (`PriceProvider`, `OptionsPricer`, `GreeksCalculator`, `CorporateActionChecker`) and the OMS `*pgxpool.Pool`.
4. **OMS uses a separate database.** OMS data lives in `alphadb_oms`, not `alphadb`. Always use `OMS_DB_URL`.
5. **Replay determinism is load-bearing.** `SimulationService.ReplayDay` must be cross-run bitwise-reproducible under `asOfBoundary`. Never reintroduce map iteration in spread grouping, never add a UUID tiebreaker to the position repo default sort, and background loops must skip ticks while `asOfBoundary` is set. See `oms_replay.md`.
6. **Exit rule tie-breaking is fixed.** `Expiration > GammaExit > DeltaExit > MaxDollarLoss($) > StopLoss > LossPct > EODExitTime > DTEExit > MaxHold > IntradayProfitPct > MaxDollarProfit($) > TakeProfit > ProfitPct > TrailingStop`. Kill switches fire first so positions exit before the P&L stop absorbs the full move.
7. **Delta/gamma thresholds are per-position normalized.** `netDelta = Σ(bs_delta × side_sign × quantity) / totalContracts`. A 1-contract and 10-contract IC use the same `delta_exit` threshold.
8. **Time/date correctness.** Never `t.Truncate(24*time.Hour)` for midnight (epoch-based → DST/timezone bugs). Never `t.Add(±24*time.Hour)` to step dates (DST days are 23h/25h). Use `timeutil.StartOfDayEastern` / `t.AddDate(0,0,±1)`.
9. **Admin mutations need the Admin Key.** Read-only GETs can take the API Key. Trading endpoints check per-account: schwab live needs `ALPHA_OMS_KEY`, paper/sim only need `ALPHA_API_KEY`.
10. **MCP tool parameters must be updated in lockstep across three files** — HTTP handler, MCP tool schema, MCP tool handler. Missing any one causes silent failures. See the checklist in `mcp_tools.md`.
11. **Two-process role boundary.** AlphaDB runs as `alphaserver-live` (`:8080`) + `alphaserver-replay` (`:8081`) sharing one OMS DB. Writes are gated by `account_key` via `RoleGuard`; wrong port returns `400 WRONG_ROLE`. Replay-only endpoints (`replay-day`, `simulate-outcome`, `as-of-date` set/clear) must go to `:8081`. Live writes to `paper` / `schwab-sim` must go to `:8080`. Background loops only attach in the live process. See `oms_replay.md` §"Two-process role boundary".
