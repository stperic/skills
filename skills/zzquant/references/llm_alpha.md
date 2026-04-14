# LLM-Powered Alpha Research

## Premise

Use large language models to process unstructured data (news, filings, earnings calls, social media) and generate or refine alpha signals. Two distinct use modes:

1. **LLMs as feature extractors** — sentiment scoring, entity extraction, event detection from text
2. **LLMs as alpha researchers** — propose novel alpha expressions, iterate with backtest feedback, debate with other agents

Both are real, useful, and mostly fresh (2023–2026 literature).

## Use cases

### Sentiment analysis
NLP on financial news and filings for directional signals. Standard today: embed text with a finance-tuned model (FinBERT, FinGPT, or general-purpose LLMs) → score sentiment → feed to downstream alpha model.

Pitfalls:
- **Ticker disambiguation** (AAPL vs apple-the-fruit)
- **Sarcasm, negation, forward-looking statements** that reverse meaning
- **Latency**: if by the time you read the news it's already priced in, the signal is worthless

### Earnings call analysis
Extract forward-looking statements, management tone changes, guidance revisions. Nontrivial because:
- Guidance is often hedged
- Tone shifts year-over-year are the signal, not absolute tone
- Q&A section is more informative than prepared remarks (management off-script)

### Alternative data processing
Parse SEC filings (10-K, 10-Q, 8-K), patent data, regulatory documents. LLMs reduce the cost of custom parsers dramatically.

### Alpha factor generation
LLMs propose novel alpha expressions from financial hypotheses. Examples:
- **Alpha-GPT** (EMNLP 2025 Demos): user inputs an idea → LLM generates valid alpha expression → backtested → refined via human-in-the-loop
- **AlphaAgent** (KDD 2025): regularized LLM alpha mining with explicit anti-decay controls

## Multi-agent trading frameworks

Multiple LLM agents with different roles collaborate or debate before signaling:

- Market analyst
- Risk manager
- Contrarian / devil's advocate
- Macro strategist
- Technical analyst

The agents debate, converge (or explicitly disagree), and a consensus mechanism produces the final signal. Examples in recent literature: TradingAgents, FinAgent, Alpha-GPT's multi-agent variants. Often ensembles of 10+ agents with varying risk preferences.

Real value is unclear — many framework papers show backtests with mild OOS Sharpe improvements but heavy compute cost. Use cautiously; validate rigorously.

## Alpha-GPT pipeline (representative)

```
1. Knowledge compilation: user idea + external memory of similar expressions
2. LLM generates alpha expression + config
3. Alpha Search: backtest candidate expression
4. Thoughts Decompiler: interpret result in natural language
5. Iterative refinement: human-in-the-loop feedback
```

The insight: LLMs are better at *proposing* and *explaining* than at *optimizing*. Use them to widen the search space, not to pick winners.

## Originality enforcement (the AlphaAgent contribution)

Alpha decay is faster when LLM-generated signals resemble existing factors. AlphaAgent adds:

- **AST similarity check**: represent alpha as abstract syntax tree; reject candidates too similar to known factors
- **Hypothesis alignment**: require each candidate to cite an economic hypothesis, not just pattern-match historical data
- **Complexity control**: penalize over-long expressions

These controls address the fundamental risk: LLMs are trained on public finance literature and will naturally regurgitate known factors unless actively constrained.

## Cautions

1. **Plausibility ≠ profitability**: LLMs produce plausible-sounding reasoning for overfit signals. They are very good at post-hoc rationalization.
2. **Out-of-sample discipline**: stricter than for human-designed strategies because the LLM can memorize factor zoo papers verbatim.
3. **Data contamination**: public LLMs saw the financial literature in training. Any backtest on public data potentially leaks through the model.
4. **Decay acceleration**: if everyone uses the same LLMs for alpha, factor crowding accelerates.
5. **Cost vs alpha**: LLM inference is expensive; signals must clear this hurdle.
6. **Prompt stability**: the same prompt can produce different answers across model versions. Version-pin and log.

## Practical integration patterns

### Pattern A: feature extraction pipeline
1. Nightly: ingest news + filings → LLM sentiment / entity extraction
2. Store structured features in the research DB
3. Downstream ML alpha models consume those features
- Use LLM as a *preprocessor*, not a trader

### Pattern B: human-in-the-loop research assistant
1. Researcher proposes a hypothesis
2. LLM suggests alpha expressions and sanity checks the math
3. Researcher runs backtest, refines, iterates
- Use LLM as a *productivity multiplier*, not an autonomous system

### Pattern C: fully autonomous agent (experimental)
1. Agent loop: propose → backtest → critique → refine
2. Multi-agent debate for sign-off
3. Deploy signal to paper trading only
- Treat with maximum skepticism; great for research, not production

## Python libraries

- `langchain`, `llama-index` — LLM application frameworks
- `transformers` — FinBERT, FinGPT, sentence embeddings
- `openai`, `anthropic` — API clients
- `haystack` — retrieval-augmented pipelines

## References

- Alpha-GPT (EMNLP 2025 Demos) — human-AI interactive alpha mining
- AlphaAgent (KDD 2025) — decay-resistant regularized LLM alpha mining
- "From Deep Learning to LLMs: A Survey of AI in Quantitative Investment" (2025)
- FinBERT (Araci 2019) — financial sentiment pretrained model
- FinGPT (Yang et al. 2023) — open-source financial LLM
- TradingAgents, FinAgent — multi-agent framework papers (2024–2025)
