# sf-decision-stack — SPEC

**Status:** draft v0, awaiting Patrick review (no code written yet).
**Date:** 2026-04-28.
**Author:** Claude (autonomous research session, while Patrick was sleeping).

## 1. Mission

Build two small, locally-runnable, low-latency LLMs that consume SF (simplefunctions)
internal state and produce *trading decisions*:

- **Analyst** — produces alpha. Reads market state + news + indicators + user intent;
  emits structured trade *ideas* (ticker, direction, conviction, time horizon, reason).
  Does not place orders.
- **Trader** — executes alpha. Reads an analyst idea + current orderbook +
  position + risk gates; emits structured *intents* (limit/market, size, expiry).
  Optimizes for fill quality and risk compliance.

Both models inherit common-sense and language fluency from a pretrained base
(no from-scratch training). They specialize via SFT on SF's own decision logs
plus GRPO on realized outcomes.

The mental model: **analyst is a quant + portfolio manager hybrid; trader
is a sell-side execution trader**. Analyst thinks slow (1-3s, runs every
5-15 min). Trader thinks fast (target <200 ms, runs per signal change).

## 2. Why this project, why now

Patrick's instruction (2026-04-28, post-`ttt-conv-memory` failure):

> 训练一个接近realtime response的小模型 ... user intent，(different scale and time
> horizon) market data (orderbook, history etc, correlated markets, index, beta etc),
> news (from all sources on all scales). 然后要有常识 所以不是从头训练 ... 然后
> 输出的就是决策。 然后两类模型 一类是research analyst（生产alpha的）另一类是
> trader（执行alpha的）

Three implicit constraints from prior context:
- **Realtime** — at least the trader; analyst can be slower.
- **Common sense built-in** — start from a pretrained LLM; never train from scratch.
- **Bridges quant ML and human-trader intuition** — text-native reasoning + numerical features.

We already have prior art: `portfolio_automation v2` (deployed 2026-04-14)
runs Gemini 2.5 Flash 1M-context as the production decision engine, fed by 13+
internal data sources. **That is our teacher.** The two new models are
distilled, faster, cheaper students of that pipeline plus the human-edited
trade ideas it has been producing.

## 3. The two models in detail

### 3.1 Analyst (alpha generation)

| | |
|---|---|
| Role | Read state, propose ideas |
| Cadence | Every 5-15 min and on news event |
| Latency budget | 1-3 s |
| Base model | Qwen3-7B (or Qwen3-3B for v0) |
| Adaptation | LoRA + numerical adapter (see §5) |
| Output schema | JSON: `{ticker, venue, direction, conviction (0-1), time_horizon (mins/hours/days), reason (markdown), supporting_indicators (list), risk_caveats (list)}` |
| SFT supervision | (a) past `trade_ideas` rows (~30/day, daily cron) (b) `thesis_state.causalTree` + `thesis_evaluations` (4×/hour) (c) Gemini portfolio-LLM logs (`portfolio_llm_context`) where applicable |
| GRPO reward | (a) realized PnL of executed ideas (from `portfolio_trades`) (b) thesis-confidence drift in correct direction (c) format adherence |

### 3.2 Trader (alpha execution)

| | |
|---|---|
| Role | Turn ideas into orders |
| Cadence | Per analyst-output change + per orderbook tick (or 5s heartbeat) |
| Latency budget | <200 ms (vLLM with grammar-constrained decoding) |
| Base model | Qwen3-1.7B or Qwen3-0.5B (v0); SmolLM2-1.7B as fallback |
| Adaptation | LoRA + numerical adapter |
| Output schema | JSON: `{action: place\|cancel\|hold, venue, marketId, side, type: market\|limit, price, size, time_in_force, reason}` |
| SFT supervision | (a) `intents` + `intentFills` table (the action log of past trader behavior) (b) Gemini-flash decisions converted into trader form |
| GRPO reward | (a) effective fill price vs midpoint (slippage) (b) fill rate (c) risk-gate compliance from `portfolio_state` |

## 4. Inputs (taxonomy)

All four are mentioned by Patrick. Mapping each to existing SF data plumbing:

| Input | What | Where it lives in SF | Granularity |
|---|---|---|---|
| **User intent** | thesis text, position-size prefs, mode (dry/live/halted), goals | `thesis_state` + `portfolio_config` + `portfolio_state` | per-tick / on edit |
| **Market data — orderbook** | top-of-book + spread + depth (top of book only) | `orderbookSnapshots` (sparse) | on-demand 15-min |
| **Market data — history** | per-15-min market_price_snapshots; 4×/hr indicator history | `market_price_snapshots` + `market_indicator_history` | 15-min |
| **Market data — correlated** | `crossVenuePairs` + `marketGroupBetas` | DB tables | daily / weekly |
| **Market data — index/regime** | `sfIndexSnapshots` (4 indices) + `market_regime_snapshots` | DB tables | 6h |
| **News — bills** | Congress.gov + OpenStates ingestion | `events` + `government_news` + `legislation_market_pairs` | weekly |
| **News — events** | scraped via Firecrawl `monitor-news` cron | `events` (sparsely populated) | background |
| **News — narratives** | `blog_posts`, `opinions`, `weekly_summaries`, `briefing` API | DB | daily/weekly |
| **News — Twitter/X** | xdk integration | **not stored** — query-only | — |

See §10 for the full data inventory with row counts and freshness.

## 5. Architecture choices

We considered six approaches; only one fits all four constraints (small,
low-latency, common-sense, decision-output). Comparison:

| Architecture | Latency | Common sense | Numeric handling | Notes |
|---|---|---|---|---|
| Pure pretrained LLM, prompt only | high | yes | weak | Patrick already tried this with Gemini 1M-flash; works but expensive at scale |
| LoRA-tuned small LLM, text only | low (<500ms) | yes | very weak | Misses orderbook signal — wouldn't learn microstructure |
| LoRA-tuned small LLM + numerical adapter (MLP encoder → soft tokens) | low (<500ms) | yes | yes | **Recommended** — mirrors LLM4TS, FinR1's approach |
| Pure tabular/sequence RL (Decision Transformer / PPO over state-action-reward) | very low (<10ms) | no | yes | Fast but no language reasoning, no auditability |
| Hybrid LLM-strategy + RL-execution | mixed | partial | yes | Two-system maintenance overhead, slower to iterate |
| Joint base+TTT training (the failed approach in `ttt-conv-memory`) | n/a | n/a | n/a | Negative result — out of scope |

**Recommended:** LoRA + numerical adapter on a pretrained Qwen3 base, separate
sizes for analyst (7B or 3B) and trader (1.7B or 0.5B). Train via two-stage
SFT-then-GRPO recipe, the same pipeline Fin-R1 used to reach 75.2 avg on
financial benchmarks with a 7B model.

### 5.1 Numerical adapter

```
[ orderbook tensor (T × F) ]      [ indicator tensor (T × N) ]
            │                              │
            └────► MLP encoder ───────► [K soft tokens]
                                              │
[ tokenized prompt + intent text ] ─────► [tokenized embeddings]
                                              │
                          concat → LLM (LoRA) ──► structured output
```

K = 16-32 soft tokens carry the numerical state. The encoder is small (<10M
params). This pattern matches `LLM4TS` (time-series alignment stage) and is
proven on financial data.

### 5.2 Inference deployment

- **Trader**: vLLM with grammar-constrained decoding (regex / lark grammar
  enforces the JSON schema). On a single A10G or RTX 4090: ~50-150 ms for
  Qwen3-1.7B at 200 tokens.
- **Analyst**: vLLM single-instance; periodic cron via Trigger.dev. Run on
  A100-40G or A10G; ~1-3s per call.
- Both can ship behind `/api/agent/analyst` and `/api/agent/trader` endpoints,
  or as Modal scheduled functions. Local `sf runtime` already has vLLM-style
  shape (per `project_fly_runtime_state_2026-04-27.md` memory).

## 6. Training plan (two stages, per Fin-R1)

### Stage A — SFT on teacher decisions

The teacher pool we have today:

| Source | Rows available | Quality | Use for |
|---|---|---|---|
| `trade_ideas` (daily cron output) | ~30/day × 90+ days = ~2700 | Mixed (LLM-generated, some human-edited) | analyst |
| `thesis_state` + `thesis_evaluations` | ~16 theses × 4×/hr × 30d = ~46k rows | High (with realized returns) | analyst |
| `portfolio_llm_context` (Gemini logs) | ~14 days × ~5/hr = ~1700 decisions | High (production decisions) | both |
| `intents` + `intentFills` | unknown (need to count) | High (real orders) | trader |
| `portfolio_trades` | ~50-100 known trades | High (with PnL outcome) | both (reward) |

Stage A targets ~5-10k SFT samples per model, balanced across categories.
LoRA rank 16-32, lr 1e-5, 2-4 epochs, ~$5-10 each on A100.

### Stage B — GRPO on outcomes

After Stage A, each prompt gets K=8 candidate decisions. We score with:

**Analyst rewards:**
- Format reward (binary): valid JSON conforming to schema
- Accuracy reward (continuous): correlation between predicted conviction and
  realized return over the time horizon (capped at ±1)
- Edge reward: agreement with sign of `market_indicators.ee` (expected edge)
  weighted by horizon

**Trader rewards:**
- Format reward (binary)
- Slippage reward (continuous): −|fill_price − midpoint| / spread
- Compliance reward (binary): risk gates pass
- Fill reward (continuous): fill_quantity / requested_quantity

GRPO step: ~2000-5000 steps on A100, ~$10-20.

### Stage C — eval

Before any production rollout, eval against:
- Held-out trade_ideas (analyst)
- Held-out intent fills (trader)
- Counterfactual on `portfolio_trades` (would we have placed the same? what's
  the alpha gap vs Gemini-flash teacher?)
- Latency benchmarks (P50/P95/P99 on the deployment hardware)

## 7. Phase ledger

- [ ] **Phase 0 — spec + state files** (this commit)
- [ ] **Phase 1 — feasibility check** (Patrick green-lights, then we…)
   - count actual rows in each source table
   - dry-run the SFT data formatter on a 100-row subset
   - measure inference latency of stock Qwen3-1.7B + Qwen3-7B on Modal A10G
   - decide on numerical adapter shape (T, F, K)
- [ ] **Phase 2 — SFT data pipeline** (offline, ~$0)
   - Snapshot teacher decisions into a flat parquet
   - Convert each row to `(prompt, completion)` pairs
   - Split train/val/test by time (last 7 days = test)
- [ ] **Phase 3 — Stage A SFT training** (analyst first, ~$10)
   - Modal A100 single GPU, LoRA rank 32, Qwen3-3B
   - Eval against held-out test set
   - Decide go/no-go on trader SFT
- [ ] **Phase 4 — Stage A SFT (trader)** (~$5)
- [ ] **Phase 5 — Stage B GRPO (analyst)** (~$15)
- [ ] **Phase 6 — Stage B GRPO (trader)** (~$15)
- [ ] **Phase 7 — eval + benchmark** (~$5)
- [ ] **Phase 8 — deployment shape** (vLLM endpoint or Modal function)
- [ ] **Phase 9 — A/B against Gemini teacher** (offline counterfactual)

Total predicted budget end-to-end: **~$50-100** on Modal.
Predicted wall-time: 2-3 weeks part-time.

## 8. Risks & open questions

1. **Tick-level orderbook depth is missing.** SF persists top-of-book only.
   The trader will struggle to learn microstructure (depth changes pre-fill,
   queue position, hidden liquidity). Mitigation: stick to swing-style
   execution; revisit when a tick-data ingestion pipeline ships.
2. **News data is sparse.** Twitter/X queried but never stored. News scraping
   cron exists but its row volume is unclear. Mitigation: focus analyst on the
   *narrative* tables we do have (blog/opinions/weekly_summaries) for v0;
   add news ingestion in a later phase.
3. **Gemini-flash teacher is not free.** SFT on its decisions assumes the
   logs are clean and accessible. Need to verify `portfolio_llm_context` is
   complete (no truncation gaps).
4. **Latency target for trader is aggressive.** <200 ms with grammar-decoded
   JSON output on Qwen3-1.7B is achievable on A10G with vLLM, but Modal
   cold-starts may eat the budget. May need a warm-pool deployment or local
   GPU.
5. **Reward design is tricky.** Realized PnL is sparse and noisy (most
   trades take hours). Need a denser reward signal — likely thesis-confidence
   drift or a shorter-horizon synthetic outcome.
6. **Distribution shift.** Training on the last 90 days of decisions assumes
   the world distribution doesn't shift. Election period, regime change, or
   new venue would degrade quickly. Plan for re-training cadence (weekly?).

## 9. Decisions Patrick needs to make

1. **Repo name.** Currently `sf-decision-stack`. Alternatives:
   `sf-analyst-trader`, `realtime-decision-stack`, `sf-alpha-execution`.
2. **Model size for v0 analyst.** Qwen3-3B (cheaper, faster iteration) or
   Qwen3-7B (probably better, ~2x training cost)?
3. **Trader architecture.** LLM with grammar (Qwen3-1.7B) or non-LLM RL
   policy (Decision Transformer)?
4. **Where to deploy.** Modal scheduled functions, Fly.io agent (the existing
   `sf-runtime` shape), or behind an existing landing API endpoint?
5. **Whether to add tick-data ingestion** *before* trader training, or
   accept top-of-book only for v0.
6. **Distillation source.** Do we use Gemini 2.5 Flash logs as the primary
   teacher, or only as one of several SFT sources?

## 10. Data inventory (summary; full report in `docs/data-inventory.md` once we copy it)

This was assessed 2026-04-28 against the live SF codebase. Headlines:

- **9 indicators** at 4×/hour cadence — analyst-ready.
- **~9,800 markets** across Kalshi+Poly with 5-min refresh.
- **~30 trade ideas/day** generated by the daily cron — primary SFT target.
- **~16 active theses** with 4×/hour evaluation — secondary SFT target.
- **`portfolio_llm_context` ≈ 1700 production decisions** from Gemini flash
  teacher, ~14 days of operation.
- **R2 archives daily** of 33+ tables (per `project_r2_dump_architecture.md`)
  — historical training set in place.
- **Top-of-book only** for orderbook (no depth history).
- **Twitter/X queried but not stored.** News cron present but sparse.

Analyst readiness: **8/10.** Trader readiness: **6/10** (gated by orderbook
depth gap).

## 11. Out of scope (explicitly)

- Reinforcement learning from scratch (no language base).
- Joint base + TTT-fast-weight training (we tried, see `ttt-conv-memory`).
- Cross-venue execution against Polymarket (only Kalshi supported today).
- High-frequency / latency arbitrage strategies (data + venue gating).
- Multi-asset / equities / FX (project scope is prediction markets).

## 12. Prior art consulted

- **Fin-R1** (arXiv 2503.16252): Qwen2.5-7B + 60K SFT + GRPO; 75.2 avg on
  financial benchmarks. Recipe template for our two-stage training.
- **LLM4TS** (github.com/liaoyuhua/LLM4TS): time-series alignment stage for
  pretrained LLMs. Numerical adapter design reference.
- **FinMem** (github.com/pipiku915/FinMem-LLM-StockTrading): layered memory
  for LLM trading agents — relevant to analyst memory design later.
- **Language Model Guided RL in Quantitative Trading** (arXiv 2508.02366):
  hybrid LLM-strategy + RL-execution. Lives next door to our analyst+trader split.
- SF internal memory:
  `project_portfolio_automation.md`, `project_thesis_git_phase5_canary.md`,
  `project_full_stack_audit.md`, `project_r2_dump_architecture.md`,
  `project_fly_runtime_state_2026-04-27.md`, `project_external_metrics_2026-04-08.md`.

---

End of SPEC v0. Awaiting Patrick's read.
