# LLM-in-the-Loop Trading Operations

## Scope

This file covers **operating** an LLM as part of a systematic trading loop — nightly strategy review, intraday trigger evaluation, pretrade qualitative filters, regime narrative, parameter adjustment. For LLM use in *research and signal discovery*, see `llm_alpha.md`. The two modes are fundamentally different: research is open-ended exploration; operations is a production system with uptime, latency, cost, and safety requirements.

## The central operational problem

An LLM in a trading loop is a non-deterministic, latency-variable, cost-variable, hallucination-prone dependency on the critical path. The entire design challenge is to put it *near* the critical path — informing decisions, narrating state, adjusting parameters — while never letting it *block* or *override* deterministic risk logic.

**Rule 1**: an LLM failure must be survivable. If the LLM is down, the system trades (or halts) to safe defaults.
**Rule 2**: an LLM output must be validated. Never trust the JSON. Ever.
**Rule 3**: an LLM decision must be auditable. Prompt, context, raw response, parsed verdict, applied effect — all logged.

## Patterns for LLM integration

### A) Nightly strategy review (low-frequency, high-latitude)

Run once per day (pre-market). LLM reviews yesterday's P&L, regime classification, open positions, strategy parameters. Output: KEEP / ADJUST / PAUSE + specific parameter deltas.

- **Latency budget**: seconds to minutes — generous
- **Cost**: one call per strategy per day — cheap
- **Blast radius**: next day's parameters
- **Safety mechanism**: bounds clamping on all proposed parameters (see `set_param()` patterns)

This is the safest LLM pattern: infrequent, generous timing, deterministic bounds on the output, full context available.

### B) Intraday trigger evaluation (medium-frequency, medium-latitude)

LLM wakes up on specific triggers (drawdown threshold, N consecutive stops, VIX regime shift, signal drought). Reviews current state, proposes adjustments via a short-lived config patch.

- **Latency budget**: tens of seconds
- **Cost**: bounded by trigger count per day
- **Blast radius**: remainder of trading day; patches auto-expire at next daily reset
- **Safety mechanism**: config patches have guardrails (allow-list of parameters, bounds, auto-expiry)

Critical: triggers must be pre-defined and discrete. LLMs should not choose *when* to wake up — they should respond when a deterministic trigger fires.

### C) Pretrade qualitative filter (high-frequency, low-latitude)

Every potential trade passes through an LLM that returns APPROVE / VETO + reason. The LLM sees the trade rationale, current positions, recent news, regime state.

- **Latency budget**: <2 seconds — tight
- **Cost**: per-signal — dominant
- **Blast radius**: one trade
- **Safety mechanism**: VETO can only block, never force entries; a blocked trade is simply skipped

**Warning**: the pretrade filter is seductive and dangerous. If it vetoes 50% of trades, you are running the LLM as the strategy, not as a filter. Monitor veto rate; a filter that vetoes more than ~15–20% of signals is effectively taking over and should be rethought.

### D) Regime narrative overlay

LLM writes prose describing the current regime and its implications. Output is *for humans* (reports) and as *context* for later LLM calls. Not a decision mechanism.

- Cheap, high-value, low-risk
- Always safe to add; always valuable for auditability

### E) Portfolio review (rare, high-latitude)

Weekly or monthly: LLM reviews strategy allocations, proposes rebalancing. Output: KEEP / REBALANCE + target weights. Same safety pattern as nightly review — bounds, audit, review before apply.

### F) Outcome attribution (purely analytical)

After a trade or a regime, LLM narrates what happened, why the P&L landed where it did, and what to update in the mental model. Feeds back into next review cycle as context.

**This is the feedback loop that makes LLM-in-the-loop actually improve.** Without outcome attribution, the LLM is blind to whether its advice worked. See "Outcome tracking" below.

## JSON contracts (not free-form prose)

Always make the LLM return structured JSON. Free-form prose is a parse-fail trap.

```json
{
  "verdict": "ADJUST",
  "adjustments": [
    {"parameter": "atr_stop_multiplier", "value": 2.5, "reason": "..."}
  ],
  "rationale": "...",
  "confidence": "medium"
}
```

- **Schema first**: define the contract before the prompt
- **Validate**: reject responses that don't match schema → log and halt, don't patch
- **Versioning**: include schema version in the prompt so you can evolve
- **Enum fields**: verdict should be a small enum, not free text
- **Bounded numerics**: clamp to bounds regardless of what the LLM says
- **Reason strings**: keep them but never trust them for logic — logging only

## Context assembly

The LLM is only as good as its context. Rules:

1. **Numbers before narrative** — lead with quantified state (P&L, drawdown, Sharpe, regime state probability, parameter values). Prose second.
2. **Point-in-time everywhere** — no accidental future data. If providing an annotation "[regime=fear, default=2.5, previous=2.3]", the "previous" value must be from *before* the current decision.
3. **Structured sections** — headers per concern (POSITIONS, RECENT P&L, REGIME, PARAMS) not a narrative paragraph.
4. **Deterministic formatting** — same data → same formatted context. No time.time() leakage, no nondeterministic ordering.
5. **Include failure attribution** — show the LLM its previous decision and its observed outcome ("last week you lowered the stop multiplier to 2.3; that change contributed −$340 to P&L"). This is where feedback loops are built.
6. **Size discipline** — trim aggressively. More context ≠ better decisions; it dilutes signal and raises cost and latency.

## Outcome tracking (the feedback loop)

Each LLM decision must be paired with an observed outcome so future decisions can see it. Standard pattern:

```python
@dataclass
class Decision:
    timestamp: datetime
    prompt_hash: str
    context_snapshot: dict
    verdict: str
    adjustments: list
    rationale: str
    # populated later
    outcome: Literal["IMPROVED", "DEGRADED", "NEUTRAL"] | None = None
    outcome_metric: float | None = None
```

Outcome computation is *pure* — a function of (decision, observed_metric_after). No state, no side effects. This makes it testable, replayable, and deterministic.

Feed recent outcomes into the next decision's context: "your last 5 adjustments: IMPROVED, NEUTRAL, DEGRADED, IMPROVED, NEUTRAL."

## Safety patterns

### Bounds clamping
Any numeric parameter the LLM can propose must pass through a validator that clamps to hard min/max. LLM saying `atr_stop_multiplier = 15.0` does *not* result in a stop 15× ATR away; it results in a clamp to the max and an audit log entry.

### Allow-list, not block-list
Define which parameters the LLM is *allowed* to touch. Anything else is silently ignored. This prevents an LLM from "helpfully" adjusting a parameter you never intended it to control.

### Single validation point
One function, one place, for validating LLM output. Scattering validation across callers guarantees inconsistency. Centralize.

### Deterministic fallback
If the LLM call fails (timeout, API error, schema fail, bounds-violated), the system continues with the previous day's parameters or a safe default. The LLM is never required for the system to run.

### Kill switch
One flag that disables all LLM calls and falls back to deterministic rules. Flip it when the LLM is misbehaving; fix offline.

### Replay equivalence
Any LLM-in-the-loop system must be runnable in replay mode with the same code path. Either record LLM responses and replay them, or make the replay mode use a deterministic stub. Replay-live equivalence is essential for debugging and validation.

## Prompt engineering for systematic trading

1. **System prompt is a contract**, not a persona. State: inputs, outputs, safety rules, what NOT to do.
2. **Example JSON in the prompt** — show the exact schema with a worked example.
3. **Refusal clauses** — tell it to return a specific "INSUFFICIENT_CONTEXT" verdict rather than guess.
4. **No apologies / no hedging** — waste of tokens, muddies logic.
5. **Temperature near 0** for operations; higher only for regime narrative.
6. **Pin the model version** — don't silently upgrade; test each version before rollout.
7. **Prompt cache** where available — nightly review with stable system prompt is a natural cache candidate; saves 50–90% on cost.

## Cost and latency

- **Token counting**: log tokens per call, monitor drift as context grows
- **Cache discipline**: use provider prompt caching for stable portions
- **Batching**: nightly reviews across strategies can be batched into one call with multiple outputs
- **Async**: LLM calls should be async if they're on any critical path; let deterministic logic proceed in parallel
- **Budget**: set a daily LLM spend cap and a circuit breaker

## Anti-patterns

1. **Free-form prose decisions** — "the model said we should be defensive" is not a decision
2. **LLM chooses entries directly** — treat as a research experiment, not production
3. **Unbounded parameter delegation** — LLM can touch any parameter, any magnitude
4. **No outcome tracking** — LLM can't learn from its own advice
5. **Prompt changes without versioning** — yesterday's decisions become unreproducible
6. **Production on bleeding-edge model** — test on prior-generation model first
7. **Synchronous LLM in execution hot path** — blocks everything; a single slow call stalls trading

## References

This pattern is ahead of most published literature, which focuses on LLM *research* use. Useful adjacent sources:

- Alpha-GPT (EMNLP 2025 Demos) — interactive LLM alpha mining (research mode, but the JSON contract pattern is transferable)
- AlphaAgent (KDD 2025) — regularization as anti-decay control
- TradingAgents / FinAgent (2024–2025) — multi-agent debate frameworks; operationally unsafe as-is but useful as design reference
- Anthropic engineering blog — prompt caching, tool use, structured outputs
- OpenAI function calling / structured outputs — JSON schema enforcement patterns
- Hamel Husain — blog posts on LLM evaluation and production monitoring
