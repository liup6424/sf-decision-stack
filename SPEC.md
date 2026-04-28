# sf-decision-stack — SPEC v1

**Status:** v1 (Option A: outcome-only training). Frozen pre-execution.
**Date:** 2026-04-28.
**Repo:** https://github.com/spfunctions/sf-decision-stack (private)
**Local path:** /Users/liuyizhou/spfunctions/sf-decision-stack
**Author:** Claude (autonomous research session for Patrick).
**Predecessor:** `spfunctions/ttt-conv-memory` (closed, NEGATIVE verdict).

---

## 0. Document conventions

This SPEC is intended to be self-contained and reproducible by a future
session of Claude or any engineer with Modal + R2 access. It includes:

- Full input/output schemas (not just descriptions).
- Full prompt templates (literal text).
- Mathematical reward definitions.
- Exact hyperparameters, with provenance.
- Acceptance criteria per phase, with abort conditions.
- A risk register with quantified probabilities.

Throughout: **"the model"** means the analyst or trader being trained; **"the
base"** means the pretrained Qwen3 weights we start from. **"Outcome"** means
the realized future return, never anyone's opinion of what *should* have
happened.

---

## 1. Executive summary

We train two LoRA-adapted small Qwen3 models — **analyst** (Qwen3-3B-Instruct,
produces alpha) and **trader** (Qwen3-1.7B-Instruct, executes alpha) — using
**outcome-only supervision**. We do not distill from any LLM-generated decisions,
including but not limited to: Patrick's portfolio-v2 Gemini-flash logs,
SF `trade_ideas`, SF `thesis_state` reasoning. Those have demonstrated
quality issues (autopm position-trapped, evaluations format-only, cumulative
P&L −$310 at start of session). The only training signal trusted as ground
truth is **realized future return** computed from price history.

Two stages:

1. **Format SFT (~$2).** Tiny supervised pass that teaches both models to
   emit valid JSON conforming to their respective schemas. Outputs are
   programmatically generated stubs (no decision content), so no opinion is
   distilled.
2. **GRPO (~$30-60 each model).** Group Relative Policy Optimization
   per Fin-R1 / DeepSeek-R1 style. For each historical state, base policy
   samples K=8 candidate decisions, each is scored by realized future return
   (analyst) or simulated execution quality (trader), advantages are computed
   group-relatively, KL-constrained policy update.

The model's common-sense and language fluency come entirely from Qwen3's
pretraining; the training signal in this project teaches *only* "in this
domain, this kind of reasoning leads to good outcomes."

**Hard cost cap: $100.** Abort on cumulative spend ≥ $80.

**Predicted total spend: $70-90.** Predicted wall-time: 2-3 weeks.

**Decoupling:** read-only data access. No writes to live SF tables, no API
endpoints exposed, no integration with portfolio v2, no live execution.
Pure offline experiment with eval-only deliverable.

---

## 2. Mission & scope

### 2.1 In-scope

- Train an **analyst** Qwen3-3B that, given a market state snapshot from SF's
  databases, emits a structured trade idea (ticker, direction, conviction,
  time horizon, reason). One forward pass per market per cadence.
- Train a **trader** Qwen3-1.7B that, given an analyst output and a current
  orderbook + position snapshot, emits a structured intent (action, size,
  price, time-in-force, reason). One forward pass per signal change.
- Train both with outcome-only supervision (no opinion distillation).
- Evaluate both against a counterfactual over `portfolio_trades`: would the
  new stack have outperformed Gemini-portfolio-v2 on the same trades?

### 2.2 Out-of-scope (explicit)

- Live execution. The trained models sit as artifacts only.
- Multi-tenant access. Single-user (Patrick's data, Patrick's models).
- Tick-level orderbook ingestion (top-of-book is what we have).
- Integration with `simplefunctions-landing` API or `portfolio_automation_v2`.
- Joint base+TTT training, distillation from Gemini, and any LLM-generated
  reasoning as supervision. Closed by D-008 (§16).
- Equities / FX / crypto / non-prediction-market assets.
- Any RL algorithm except GRPO. PPO with critic, DPO, RLHF-PPO are out.

### 2.3 Success criteria (final, post-Phase 7)

A model release passes if **all** of the following hold on the held-out
test set:

1. **Analyst.** On the test-set states, the analyst's predicted-direction
   alpha (mean realized return of "buy YES" trades minus mean realized return
   of "sell YES" / no-action trades over the chosen horizon) is **statistically
   positive at 95% CL** (Wilcoxon signed-rank vs. zero, two-sided).
2. **Analyst.** Counterfactual alpha vs. Gemini-portfolio-v2: when both
   stacks have an active opinion on the same trade window, the analyst's
   horizon-weighted return is **at least non-inferior** (one-sided, p < 0.10)
   to Gemini's. We do not require strict outperformance; we require that the
   smaller, faster, cheaper, decoupled stack does no worse.
3. **Trader.** On simulated fills against historical top-of-book + spread,
   slippage is **bounded above by the midpoint-cross spread** in 95% of
   trades (i.e. trader doesn't cross more than the visible spread).
4. **Latency.** Trader p95 ≤ 200 ms with grammar-constrained vLLM on a
   single A10G; analyst p95 ≤ 3 s on A100-40G. Measured with a representative
   prompt distribution.
5. **No-regression.** The grammar constraint is honored on 100% of the test
   prompts (zero malformed JSON).

If 1, 3, 4, 5 hold but not 2, the project is a **partial success** (we have
a smaller-cheaper stack but no provable alpha advantage). If 1 fails, the
project is a **negative result**, documented and shipped with a CLOSURE.md
in the same shape as `ttt-conv-memory`.

---

## 3. Why this project; relationship to predecessor

### 3.1 Background (from session memory)

- 2026-04-14: SF deploys portfolio-automation-v2 with Gemini 2.5 Flash 1M
  context as the prod decision engine.
- 2026-04-19: prompt audit (`project_prompt_audit_2026-04-19.md`) finds
  4/6 alleged claims confirmed: evaluateEvents is format-only, autopm is
  position-trapped, thesis_id 0/35 written, autopm tick p99 = 285s.
- 2026-04-27: 2nd portfolio user (Cameron Whitt) found unauthorized.
- 2026-04-28 morning: cumulative portfolio P&L = −$310.42 per heartbeat.
- 2026-04-28: `ttt-conv-memory` ran iteration-2, conclusively showed past_w
  encodes nothing at any α — abandoned at $16 spend.
- 2026-04-28 evening: this project kicks off.

### 3.2 Why outcome-only is the right answer here

If we distilled the existing portfolio-v2 decisions, we would inherit its
failure modes (the position trap, format-only evals, etc.). Patrick's
2026-04-28 19:30 message was explicit: *"我们那个portfolio 就是判断的不咋滴啊"*.
Outcome-only training breaks this contamination — the realized return is
the only signal that doesn't carry someone else's bias.

### 3.3 What "outcome-only" precisely means

For every training pair `(s_t, a_t, o_{t+H})` where `s_t` is state at time
`t`, `a_t` is a candidate action sampled from the policy, and `o_{t+H}` is
the realized outcome by horizon `t+H`, the loss is computed only against
`o_{t+H}` — never against any human or LLM-generated `a_t*`. We never
provide a "correct answer" for the policy to imitate. The policy discovers
correct answers via reward-shaped sampling (GRPO).

---

## 4. Models — full I/O specification

### 4.1 Analyst

**Role.** For a market `m` at time `t`, decide whether to take a position
(YES or NO) and at what conviction over what horizon, plus a free-form
reasoning trace.

**Cadence.** Once per market per ~15-min cadence; or on-demand.
**Latency target.** P95 ≤ 3 s on A100-40G.
**Base model.** `Qwen/Qwen3-3B-Instruct`, bf16, sdpa attention. Loaded via
`transformers` `AutoModelForCausalLM` with our LoRA weights merged at
inference.

**Inputs.**
- `system`: literal system prompt (§5.1.1).
- `user`: structured prompt containing
  - `intent_block`: free-form text representing user intent. For training,
    we use a fixed canonical intent: "Find positive-edge trades with
    horizon-appropriate conviction. Default risk: small."
  - `market_block`: market metadata
    - canonical_market_id, ticker, venue, title, outcomes
    - current best bid / ask / midpoint / spread
    - 24h volume, open interest
  - `history_block`: 48-hour price history at 15-min resolution (text-formatted
    series; first row is oldest, last row is most recent).
  - `indicator_block`: 9 indicators (IY_yes, IY_no, CRI, EE, LAS, CVR, RV, VR,
    IAR) at the current snapshot, each with its 48h history rolled up to a
    one-line summary (mean, last, percentile vs. own history).
  - `regime_block`: latest `market_regime_snapshots` row for this market.
  - `cross_venue_block`: if `crossVenuePairs` has an entry, the paired
    venue + price gap.
  - `news_block`: last 7 days of relevant `events` rows whose
    `legislation_market_pairs.canonical_market_id` matches OR whose vector
    similarity to the market title exceeds 0.7. Up to 5 items.
  - `<MARKET_STATE_TOKENS>`: literal placeholder. At runtime, replaced by 32
    soft tokens emitted by the numerical adapter (§6.3).

**Output (JSON; exactly this schema, enforced at inference via grammar).**

```json
{
  "thinking": "string (free-form; the model's reasoning trace; <= 800 chars)",
  "decision": {
    "ticker": "string (canonical market ticker)",
    "venue": "kalshi | polymarket",
    "side": "yes | no | abstain",
    "conviction": "number in [0, 1] (1 = highest conviction)",
    "time_horizon_hours": "number in [0.25, 168] (15 min to 7 days)",
    "stop_thesis": "string (one-line: what would invalidate this; <= 200 chars)"
  },
  "supporting_indicators": ["EE", "CVR", "..."]
}
```

The `thinking` field is the model's chain-of-thought. We deliberately do
not constrain its content; GRPO shapes it indirectly via the decision's
realized return. (Per Fin-R1, this `<think>` plus accuracy reward pattern
emergently produces high-quality reasoning.)

**Grammar (lark, simplified).** See `docs/grammars/analyst.lark` (created
in Phase 1.5). Output is enforced via vLLM's `guided_decoding` parameter
or via `outlines` package.

### 4.2 Trader

**Role.** Given an analyst decision and a current top-of-book + position +
risk-state snapshot, emit a single intent (place / cancel / hold) with
size, price, time-in-force.

**Cadence.** On every analyst output change, plus a 5s heartbeat (rate-limited).
**Latency target.** P95 ≤ 200 ms on A10G with vLLM grammar-constrained decoding.
**Base model.** `Qwen/Qwen3-1.7B-Instruct`, bf16, sdpa.

**Inputs.**
- `system`: literal system prompt (§5.1.2).
- `user`:
  - `analyst_block`: the analyst's last decision (decision JSON).
  - `state_block`:
    - current best bid/ask/midpoint/spread
    - position (size, average entry price, unrealized PnL)
    - portfolio cash, exposure, headroom
    - active risk gates (from `portfolio_state`: 8 risk gates per memory).
  - `<MARKET_TOKENS>`: 16 soft tokens emitted by the trader's smaller
    numerical adapter.

**Output (JSON; grammar-enforced).**

```json
{
  "thinking": "string (<= 200 chars)",
  "intent": {
    "action": "place | cancel_all | hold",
    "type": "market | limit",
    "side": "buy | sell",
    "price_cents": "number in [1, 99] (0 if action != place)",
    "size_units": "number >= 0",
    "time_in_force": "ioc | gtc | gtd",
    "expiry_seconds": "number >= 0"
  }
}
```

**Coarse rule-based fallback.** A pure rule-based "shadow trader" is
implemented in `src/trader/rule_baseline.py` for sanity comparison.
The LoRA trader must beat this on slippage to be considered viable.

---

## 5. Prompt templates (literal)

### 5.1.1 Analyst system prompt

```text
You are SF-Analyst, a small expert model trained to find positive-expected-edge
trades on US prediction markets (Kalshi and Polymarket). For each market,
read the supplied state and emit a single JSON decision.

Rules:
- Output exactly one JSON object matching the schema. No prose outside JSON.
- The `thinking` field captures your reasoning. Be concrete. Cite specific
  indicators or news items. State the strongest counter-argument.
- Pick `side = abstain` when no edge is present. Abstaining is a real choice;
  do not feel obligated to recommend a position.
- Do not leak future information. Reason only from the supplied snapshot.
- Conviction is calibrated: 0.5 means "edge ≈ 1.5x spread"; 1.0 means
  "edge ≈ 5x spread".
- Time horizon is the period over which you expect the edge to realize.
  If you cannot specify a horizon, use 24.0.

Output shape:
{
  "thinking": "...",
  "decision": {
    "ticker": "...",
    "venue": "...",
    "side": "yes" | "no" | "abstain",
    "conviction": 0.0,
    "time_horizon_hours": 0.0,
    "stop_thesis": "..."
  },
  "supporting_indicators": ["..."]
}
```

### 5.1.2 Trader system prompt

```text
You are SF-Trader, a small expert model trained to execute alpha signals on
US prediction markets. Given an analyst decision and current market+portfolio
state, emit one JSON intent.

Rules:
- Output exactly one JSON object matching the schema. No prose outside JSON.
- `action = hold` if the analyst output is `abstain`, the position already
  matches, or risk gates block. Otherwise prefer `place`.
- Prefer `type = limit` at midpoint or one tick inside spread. Use `market`
  only when the spread is < 2 cents.
- Size respects portfolio headroom. Never exceed the analyst's implied
  position size, scaled by conviction.
- Reasoning ≤ 200 chars. The model is rewarded on execution quality, not
  prose.

Output shape:
{
  "thinking": "...",
  "intent": {
    "action": "place" | "cancel_all" | "hold",
    "type": "market" | "limit",
    "side": "buy" | "sell",
    "price_cents": 0,
    "size_units": 0,
    "time_in_force": "ioc" | "gtc" | "gtd",
    "expiry_seconds": 0
  }
}
```

---

## 6. Architecture

### 6.1 Base model versions

| | analyst | trader |
|---|---|---|
| HF model id | `Qwen/Qwen3-3B-Instruct` | `Qwen/Qwen3-1.7B-Instruct` |
| Params | 3.0 B | 1.7 B |
| Attention | SDPA (no flash-attn — saves 30 min cold-build, see ttt-conv-memory D-007) | same |
| Dtype | bfloat16 | bfloat16 |
| Context | up to 32k for training; 8k effective (we don't need more) | 8k |

If `Qwen3-3B-Instruct` is not yet released or unstable as of session date,
fall back to `Qwen2.5-3B-Instruct` and document. Same for trader `1.7B`.

### 6.2 LoRA configuration

Same for both models (literature consensus):

```python
LoraConfig(
    r=16,
    lora_alpha=32,
    lora_dropout=0.05,
    target_modules=["q_proj", "k_proj", "v_proj", "o_proj",
                    "gate_proj", "up_proj", "down_proj"],
    bias="none",
    task_type="CAUSAL_LM",
)
```

Trainable parameter count: ~30M (analyst-3B) / ~17M (trader-1.7B). Storage
~120MB / ~70MB respectively.

### 6.3 Numerical adapter

A small MLP encodes the 48-hour rolling history of 9 indicators into K
soft tokens that are prepended to the input embeddings.

- Analyst adapter: input shape `(T=192 timesteps, F=9 indicators)` →
  flattened to 1728 → MLP `[1728 → 1024 → 1024 → 32 × hidden_dim]` where
  `hidden_dim = 2048` for Qwen3-3B → output K=32 soft tokens, each of
  dim 2048.
- Trader adapter: input shape `(T=64, F=9)` → flattened to 576 → MLP
  `[576 → 512 → 512 → 16 × hidden_dim]` where `hidden_dim = 1536` for
  Qwen3-1.7B → K=16 soft tokens.

Activation: SiLU between layers. LayerNorm before the projection. Init:
`xavier_uniform_` on linears, zeros on biases. The adapter is trained
*jointly* with the LoRA (its parameters are added to the trainable set).

The soft tokens replace a literal `<MARKET_STATE_TOKENS>` placeholder in
the prompt at the embedding layer. The placeholder is one literal token in
the input ids; we replace its embedding with the K soft tokens via a
hook on `model.get_input_embeddings()`.

If T=192 is too many for memory, fallback: T=96, F=9, K=24. Adapter shape
adjusts. Decided in Phase 1 based on memory probe.

### 6.4 Output grammar

Both schemas are translated to lark grammars and compiled by `outlines` or
provided to vLLM's `guided_decoding`. See `docs/grammars/analyst.lark` and
`docs/grammars/trader.lark`. The grammars are *strict*: any field type
mismatch makes the production unparseable and forces the decoder to backtrack.

---

## 7. Data pipeline

### 7.1 Source tables (read-only, via R2 archive dumps)

All paths under `r2://sf-data-dump/{table}/{date}.jsonl` per
`project_r2_dump_architecture.md`. Tables we read:

| table | use |
|---|---|
| `external_markets` | market metadata + 5-min price snapshots |
| `market_price_snapshots` | 15-min wide-format history |
| `market_indicators` | 4×/hr indicator snapshot |
| `market_indicator_history` | 48h rolling indicator history |
| `orderbookSnapshots` | top-of-book + spread + depth ratio |
| `market_regime_snapshots` | 6h regime label |
| `crossVenuePairs` | daily pair detection |
| `marketGroupBetas` | weekly correlations |
| `events` | news / catalyst items |
| `legislation_market_pairs` | bill → market mapping |
| `government_news` | bill text |
| `canonical_markets` + `canonical_market_venues` | cross-venue ID mapping |
| `portfolio_trades` | RESERVED for counterfactual eval only — not for training |

We do **not** read: `portfolio_llm_context`, `trade_ideas`, `thesis_state`,
`thesis_evaluations`, `intents`, `intentFills`, `research_notes`. These
contain LLM-generated content that would contaminate Option A.

### 7.2 Time alignment & label horizons

For a training sample at time `t`:
- **State**: all columns / rows of source tables with `as_of <= t`. Strict
  inequality for any timestamped column.
- **Label**: the realized midpoint price at `t + H` for the relevant
  market, where `H ∈ {0.25, 1, 4, 24, 72, 168}` hours. We sample the
  horizon randomly per state during training (so the model learns to
  handle multi-horizon prompts).
- **Filter**: drop samples where the market resolved before `t + H` —
  realized price is then the resolution outcome (0 or 100 cents).

### 7.3 Train / val / test split

Strict time-based split, no leakage:

| split | period | N states (target) |
|---|---|---|
| train | from earliest available to T_end − 14d | ~8 000 |
| val | T_end − 14d to T_end − 7d | ~1 000 |
| test | T_end − 7d to T_end | ~1 000 |

`T_end` is the latest timestamp in `market_price_snapshots` at Phase 1
extraction. The test set is **reserved** until §10 evaluation.

We further hold out 30 markets ("contagion guard": same canonical market
across train/test causes leakage via slow-moving betas). Train markets and
test markets are disjoint at the canonical-market level.

### 7.4 Sampling & stratification

Train pool stratification:
- 40% high-edge states (states where `|EE| > p75`).
- 30% high-volume states (states where `volume24H > p75`).
- 20% pre-event states (within 24h of a `legislation_market_pairs` event).
- 10% random.

Val and test follow natural distribution (no stratification) so eval
mirrors deployment.

### 7.5 Negative example handling

Halted positions, errored intents, gate-blocked attempts: NOT used in
training. The trader doesn't learn from "what the prod stack failed to
do." Counterfactual eval (§10.4) does compare against `portfolio_trades`,
including the failures, but only at eval time.

### 7.6 R2 archive access

Read-only access via R2 access key + secret + endpoint. These are stored
in `~/spfunctions/simplefunctions-landing/.env.local` as
`R2_ACCESS_KEY_ID`, `R2_SECRET_ACCESS_KEY`, and the bucket name
`sf-data-dump` (or whatever is canonical, to be confirmed in Phase 1.0).

We use `boto3` with the R2 endpoint URL `https://<account_id>.r2.cloudflarestorage.com`.

---

## 8. Training Stage 1 — Format SFT

### 8.1 Goal

Teach both models to emit valid JSON conforming to their schemas. **Nothing
more.** No decision content is supervised. This stabilizes GRPO (a base
model with no schema prior often samples K candidates that are 0/8 valid).

### 8.2 Data generation procedure

For analyst: ~500 (state, output) pairs where:
- `state` is a real market state pulled from train period.
- `output` is a *programmatically generated* JSON object:
  - `thinking`: a short canned string from a 20-template pool.
  - `decision.ticker`: the actual ticker from the state.
  - `decision.side`: random uniform in {yes, no, abstain}.
  - `decision.conviction`: random uniform in [0, 1].
  - `decision.time_horizon_hours`: random from {0.25, 1, 4, 24, 72, 168}.
  - `decision.stop_thesis`: random from a 30-template pool.
  - `supporting_indicators`: random subset of the 9 indicators, size 0-4.

For trader: same shape, ~500 pairs.

The outputs are deliberately noise — what we want the model to learn is "I
emit a JSON of this shape." We then trust GRPO + outcome reward to teach
*content*. The format SFT prevents GRPO from collapsing on malformed
outputs.

### 8.3 Hyperparameters

| | value |
|---|---|
| Optimizer | AdamW (β1=0.9, β2=0.999, eps=1e-8, weight_decay=0.01) |
| Learning rate | 1e-4 (LoRA scale) |
| LR schedule | cosine, warmup 50 steps, decay to 1e-6 over 1500 total steps |
| Batch size | 1 |
| Grad accum steps | 16 (effective batch 16) |
| Epochs | 3 (so ~1500 steps for 500 examples × 3 epochs) |
| Grad clip | 1.0 |
| Seq length cap | 4096 |
| Mixed precision | bf16 |
| LoRA targets | as in §6.2 |
| Numerical adapter | jointly trained, same LR |
| Random seed | 42 |

### 8.4 Acceptance criteria (gate to Stage 2)

Run the trained model on 100 held-out val states with grammar enforcement
**off**. Pass criteria:

- ≥ 95% of outputs parse as valid JSON.
- ≥ 90% of outputs match the full schema (all required fields present,
  types correct).
- Schema-diversity: `decision.side` distribution spans all three values;
  conviction has variance > 0.05.

If any criterion fails, fix data generation or hyperparameters and retry
(within Stage 1 budget). Do not proceed to GRPO with broken format.

### 8.5 Cost & wall-time estimate

Stage 1 analyst: ~$1.5 (1500 steps × 0.5s/step on A100 + load).
Stage 1 trader: ~$0.5 (smaller model).

---

## 9. Training Stage 2 — GRPO

### 9.1 Algorithm summary

Group Relative Policy Optimization (Shao et al. 2024; used by Fin-R1, DeepSeek-R1).

For each prompt `q` (= one historical state):
1. Sample K=8 candidate completions `{a_1, ..., a_K}` from the policy
   `π_θ` (the LoRA-adapted model). Temperature 1.0, no top-k, no top-p
   so we get diverse samples.
2. Compute reward `r_k = R(q, a_k)` for each (§9.2).
3. Compute group-relative advantage:
   ```
   A_k = (r_k - mean(r_1..K)) / max(std(r_1..K), 1e-4)
   ```
4. PPO-style clipped objective:
   ```
   L_PPO(θ) = -E_q,k [ min(rho_k * A_k, clip(rho_k, 1-ε, 1+ε) * A_k) ]
   ```
   where `rho_k = π_θ(a_k|q) / π_θ_old(a_k|q)`.
5. KL constraint to a frozen reference policy `π_ref`:
   ```
   L_KL(θ) = β * KL(π_θ || π_ref)
   ```
6. Total loss: `L = L_PPO + L_KL`.

Reference policy `π_ref` = the Stage-1 SFT'd model, frozen. β = 0.04
initially; tuned in Phase 5.5 based on KL drift.

### 9.2 Reward functions

#### 9.2.1 Analyst reward

For sample `(q_t, a_t)` with `a_t.decision = (ticker, side, conviction, H)`:

```
midpoint_t = midpoint at time t for ticker
midpoint_{t+H} = midpoint at time t+H for ticker (or resolution price if resolved earlier)

if side == "abstain":
    raw_pnl = 0
elif side == "yes":
    raw_pnl = (midpoint_{t+H} - midpoint_t) / 100   # cents → dollars per unit
elif side == "no":
    raw_pnl = (midpoint_t - midpoint_{t+H}) / 100

# Spread normalization (so a 5-cent gap on a wide market and a 1-cent gap
# on a tight market are scaled comparably):
spread_t = ask_t - bid_t in cents
norm_pnl = raw_pnl / max(spread_t / 100, 0.005)

# Conviction-weighted: model is incentivized to be confident only when right
edge_reward = norm_pnl * conviction

# Format reward (binary): JSON parses + schema valid
format_reward = 1.0 if (parses and validates) else -0.5

# Final
R = clip(edge_reward, -3, 3) + format_reward
```

The format reward is small relative to edge_reward (which can be ±3 in
absolute value when realized return is 3 spreads). If the model abstains
on every prompt, `edge_reward = 0` and total reward is `0 + format_reward
= 1`. This is OK — it discourages malformed output without forcing
participation. The variance comes from non-abstain bets that work or don't.

If the market resolved before `t+H`, we use the resolution price and
`H_eff = t_resolve - t`. The conviction term is unchanged.

#### 9.2.2 Trader reward

For sample `(q_t, a_t)` where `q_t` includes an analyst signal and `a_t`
is the trader's intent:

```
# Format reward (binary)
format_reward = 1.0 if (parses and validates) else -0.5

# Action correctness:
if analyst.side == "abstain":
    correct_action = 1.0 if intent.action == "hold" else -1.0
elif intent.action == "place" and intent.side matches analyst.side:
    correct_action = 0.5
elif intent.action == "hold":
    correct_action = -0.2  # trader should usually trade when analyst signals
elif intent.action mismatches analyst.side:
    correct_action = -1.0

# Execution quality (only for `place`):
if intent.action == "place":
    # Simulate fill against historical top-of-book + spread:
    if intent.type == "market":
        fill_price = ask if buy else bid
    elif intent.type == "limit":
        # Fill iff within visible quote
        if (buy and intent.price_cents >= ask) or (sell and intent.price_cents <= bid):
            fill_price = intent.price_cents
        else:
            # Partial / no fill — use midpoint as estimator with slippage penalty
            fill_price = midpoint  # simplified
    midpoint_cost = abs(fill_price - midpoint) / 100
    exec_reward = -midpoint_cost  # in dollars per unit; negative = good
else:
    exec_reward = 0

# Risk compliance:
risk_reward = -1.0 if intent violates portfolio risk gates else 0.0

R = format_reward + correct_action + exec_reward + risk_reward
```

#### 9.2.3 Reward shaping notes

- We deliberately use a *coarse* reward for trader because `intentFills`
  data is noisy. Future iteration can refine.
- The analyst reward is variance-bounded by `clip(., -3, 3)`. Without this,
  resolution events (e.g., a market settling at 0 from a 30-cent prior)
  produce huge outliers that destabilize GRPO.

### 9.3 Group sampling

K=8 per prompt. Temperature 1.0. Max new tokens 512 (analyst) / 128
(trader). Grammar **enforced** during sampling so all candidates are
parseable; format_reward = 1.0 for all if grammar works. Format reward
becomes a constant; group-relative advantage zeroes it out — but we keep
the term for the rare case where grammar enforcement fails on edge tokens
(e.g., max_new_tokens is hit before JSON closes).

### 9.4 KL constraint & clip

| | value | provenance |
|---|---|---|
| KL coef β | 0.04 (initial), tune in 0.01–0.1 | DeepSeek-R1 default |
| Clip ε | 0.2 | PPO standard |
| Reference policy | Stage-1 SFT'd model, frozen | per Fin-R1 |

If at step 500 the policy KL > 1.0 nats/token, halve β and rewind 100
steps. If < 0.05 nats/token, the policy isn't moving — double β. Logged
to `wandb` (free tier) with project name `sf-decision-stack`.

### 9.5 Other hyperparameters

| | analyst | trader |
|---|---|---|
| Total GRPO steps | 1500 | 1500 |
| Learning rate | 5e-6 (LoRA + adapter) | 5e-6 |
| LR schedule | cosine, warmup 100, decay to 5e-7 | same |
| Batch size (prompts) | 4 | 8 |
| K | 8 | 8 |
| Grad accum | 4 (effective batch 16 prompts) | 4 |
| Grad clip | 1.0 | 1.0 |
| Seed | 42 | 42 |

### 9.6 Logging metrics (each step)

- `loss/ppo`, `loss/kl`, `loss/total`
- `reward/mean`, `reward/std`, `reward/min`, `reward/max`
- `reward/format_only_count` (number of K candidates that scored only on
  format — i.e. parsed but had `edge_reward = 0`)
- `kl/policy_to_ref` in nats/token
- `kl/policy_to_old` (the rho denominator's inverse)
- `entropy/policy` in nats
- `decisions/abstain_rate`, `decisions/yes_rate`, `decisions/no_rate`
- `convictions/mean`, `convictions/std`
- `outputs/length_mean`

### 9.7 Acceptance criteria (gate to Phase 7 eval)

Stage 2 passes if **all** hold over the last 100 logged steps:

1. `reward/mean` is positive AND increasing over the run (Mann-Kendall
   trend test, p < 0.05).
2. `kl/policy_to_ref < 2.0` nats/token (avoid policy collapse).
3. `decisions/abstain_rate < 0.95` (avoid trivial-abstain collapse).
4. `entropy/policy > 0.5` nats (avoid mode-collapse).
5. Held-out val reward (run model on 200 val prompts, compute mean
   reward) > 0.5 (analyst) / > 0.3 (trader).

If any fail, document the failure mode and stop. Do not eval (eval costs
money) until training looks healthy.

### 9.8 Cost & wall-time

Per prompt: 1 forward (sample 8 × 256 tokens) + 1 forward (logits for ratio)
+ 1 backward = ~3s on A100 for analyst-3B. 1500 steps × 4 prompts × 3s
= 18k s = 5h ≈ $5.50. Trader-1.7B: ~half of that, ~$3.

Plus reward-evaluation cost (looking up future midpoints, computing
returns): trivial since data is local.

Conservative estimate: analyst Stage 2 = **$15-30**, trader Stage 2 =
**$15-25**. Total Stage 2: **$30-55**.

---

## 10. Evaluation methodology

### 10.1 Held-out test set

The reserved 7-day test split (§7.3). ~1000 states, ~30 disjoint canonical
markets (the held-out 30 from train). Used **only after** Stage 2 passes
acceptance.

### 10.2 Per-decision metrics

For each test state:
- `predicted_alpha`: analyst's `conviction × (1 if side=yes else -1 if side=no else 0)`.
- `realized_return_H`: realized return at `t + H_predicted`, in spread-units.
- `signed_return`: `predicted_alpha × realized_return_H`.

Aggregated:
- mean(`signed_return`) — should be positive.
- median(`signed_return`)
- hit rate (fraction with `signed_return > 0` among non-abstain)
- abstain rate
- mean conviction on hits vs. misses (calibration check)

### 10.3 Aggregate metrics (treat all test trades as a strategy)

- **Sharpe**: `sqrt(N) * mean(returns) / std(returns)` (annualized
  with N = test trades / 7 days × 365)
- **Max drawdown**: max(running_pnl − running_max).
- **Hit rate** above
- **Bet sizing efficiency**: `corr(conviction, signed_return)` — should
  be positive.

### 10.4 Counterfactual on `portfolio_trades`

For each row in `portfolio_trades` whose entry timestamp is in the test
period, run the analyst at that timestamp on the same market. Compare:

| | Gemini-portfolio-v2 (actual) | sf-decision-stack (counterfactual) |
|---|---|---|
| Side chosen | from row | from analyst |
| Conviction | (proxy from `portfolio_trades.size_units`) | from analyst |
| Realized PnL | actual | computed from realized return |

Test 2 (success criterion §2.3.2): one-sided non-inferiority on
horizon-weighted return.

### 10.5 Latency benchmarks

vLLM serving on A10G (trader) and A100-40G (analyst). Run 1000 sample
prompts. Report p50, p95, p99 of end-to-end (including grammar overhead).
Pass: p95 ≤ 200ms / 3s respectively (§2.3.4).

### 10.6 Statistical significance

- Wilcoxon signed-rank for "alpha > 0" tests.
- Bootstrap (1000 resamples) for confidence intervals on Sharpe and hit
  rate.
- Bonferroni correction across the 4 success-criterion tests.

All eval code goes in `src/eval/`. Plots saved to `results/figures/`.

---

## 11. Inference deployment

For this project, "deployment" means **producing a callable artifact** in
the repo. We do **not** deploy to landing or any serving infrastructure.

Artifact:
- `inference/analyst.py` — loads LoRA + adapter + grammar; takes a state
  dict, returns a parsed decision.
- `inference/trader.py` — analogous.
- vLLM serving config files (`docs/serving/`) — show how to serve the
  models with grammar-constrained decoding, but actual `vllm serve`
  command is not run by this project.

Reproducible inference latency benchmark goes in `bench/latency.py`.

---

## 12. Cost gates & abort conditions

| condition | action |
|---|---|
| Cumulative Modal spend ≥ $80 | Stop. Save current state. Report. |
| Cumulative Modal spend ≥ $100 | HARD ABORT — must not happen. Phase ledger budgets summed = $73 max in worst case to avoid this. |
| Stage 1 acceptance fails twice | Stop. Document. Patrick decides whether to continue. |
| Stage 2 step 500 KL collapse (KL > 5 nats/tok) | Halve β, rewind. If two such events, stop. |
| Stage 2 reward never positive after 1000 steps | Stop. Document. |
| Test alpha statistically negative | Stop. Document. Ship as NEGATIVE result. |
| Latency target missed at p95 | Document. Project still ships if alpha is positive. |

The **$80 internal cap** is a 20% buffer below the **$100 hard cap** so a
single botched run doesn't blow the budget.

---

## 13. Phase ledger with deliverables

Each phase has explicit cost, wall-time, and acceptance criteria. Phases
are sequential; each must pass before the next starts.

### Phase 1.0 — environment & R2 access (target: $0, 30 min)

Deliverable:
- R2 credentials confirmed accessible.
- `src/data/r2_client.py` reads one sample file from each target table.
- Modal app skeleton (`modal_app.py`) with `image` + `vol`.

Acceptance: end-to-end smoke test reads a 10-row sample from
`market_indicators`.

### Phase 1.5 — data extraction & schema verification (target: $0.50, 2-3h)

Deliverable:
- `src/data/extract.py`: reads R2, writes per-table parquet to local
  `data/raw/`.
- Row counts (printed, recorded in STATE.md).
- Sanity histograms (lengths, time coverage).

Acceptance: each target table has ≥ 1000 rows in the train period and ≥
100 in the test period. If any table is empty, document and decide
whether to drop it as a feature.

### Phase 2.0 — feature engineering & adapter shape probe (target: $1, 2-3h)

Deliverable:
- `src/data/featurize.py`: produces `(state_dict, label_dict)` records,
  where `state_dict` includes everything for the prompt and the numerical
  adapter input, and `label_dict` includes future midpoints at all
  horizons.
- Adapter shape final pick (T, F, K) based on memory probe.
- Train/val/test parquet files written.

Acceptance: a 100-record sample formats into prompts that fit Qwen3 8k
context. Adapter probe: forward a (T, F) tensor through the MLP, check
output shape matches K × hidden_dim.

### Phase 3.0 — Stage 1 SFT analyst (target: $1.50, 1-2h)

Deliverable:
- `src/train/sft_analyst.py`.
- LoRA + adapter checkpoint at `data/ckpt/analyst_sft/`.
- Validation report: schema-validity rates as in §8.4.

Acceptance: §8.4 gates pass.

### Phase 4.0 — Stage 1 SFT trader (target: $0.50, 30-60 min)

Same shape as 3.0 for trader.

### Phase 4.5 — eval after Stage 1 (target: $0.20, 30 min)

Deliverable:
- Run both SFT-only models on 50 val prompts each. Generate a small
  `report_after_sft.md`. Just to verify shapes work end-to-end before
  committing GRPO budget.

Acceptance: outputs are valid JSON; the model says non-trivial things
(though often wrong — that's expected, GRPO is what learns content).

### Phase 5.0 — Stage 2 GRPO analyst (target: $30, 4-6h)

Deliverable:
- `src/train/grpo_analyst.py` using `verl` or a simple custom GRPO loop.
- Logged metrics (§9.6) to W&B.
- LoRA + adapter checkpoint at `data/ckpt/analyst_grpo/`.

Acceptance: §9.7 gates pass. Hard stop at step 1500 or `kl > 5`.

### Phase 5.5 — analyst eval (target: $1, 1h)

Deliverable:
- Run analyst on test set. Compute §10.2-§10.6 metrics.
- `results/analyst_eval.md` with tables and figures.

Decision gate: if §2.3.1 holds, proceed to trader. If not, document NEGATIVE,
optionally re-tune (within budget) once.

### Phase 6.0 — Stage 2 GRPO trader (target: $20, 3-4h)

Same shape as 5.0 for trader.

### Phase 6.5 — trader eval (target: $1, 1h)

Same shape as 5.5.

### Phase 7.0 — final report + counterfactual (target: $0.50, 2h)

Deliverable:
- `RESULTS.md` with verdict against §2.3 success criteria.
- `results/counterfactual.md` (§10.4).
- Latency benchmark (§10.5).
- Push everything.

### Phase 8.0 — closure (target: $0)

Deliverable: this repo's `RESULTS.md` is the deliverable. If POSITIVE:
keep open for further iteration. If NEGATIVE: add `CLOSURE.md` mirroring
ttt-conv-memory's pattern.

### Phase totals

| phase | $ |
|---|---|
| 1.0 | 0.00 |
| 1.5 | 0.50 |
| 2.0 | 1.00 |
| 3.0 | 1.50 |
| 4.0 | 0.50 |
| 4.5 | 0.20 |
| 5.0 | 30.00 |
| 5.5 | 1.00 |
| 6.0 | 20.00 |
| 6.5 | 1.00 |
| 7.0 | 0.50 |
| 8.0 | 0.00 |
| **subtotal** | **56.20** |
| reserve (re-runs, debug) | 15.00 |
| **plan total** | **71.20** |
| **HARD CAP** | **100.00** |

Headroom: ~$28 above the plan.

---

## 14. Reproducibility

### 14.1 Seeds

```python
TORCH_SEED = 42
NUMPY_SEED = 42
PYTHON_SEED = 42

torch.manual_seed(TORCH_SEED)
torch.cuda.manual_seed_all(TORCH_SEED)
numpy.random.seed(NUMPY_SEED)
random.seed(PYTHON_SEED)
torch.use_deterministic_algorithms(False)  # GRPO sampling needs nondeterminism
```

We accept GRPO is non-deterministic by design (sampling K candidates).
For SFT, seeds are honored.

### 14.2 Software pinning

`requirements.txt` will pin:
- torch==2.8.0
- transformers==4.57.3 (matched to ttt-conv-memory)
- peft==0.13.2 (LoRA)
- trl==0.11.x or verl latest (for GRPO; pick in Phase 5.0)
- outlines or vllm latest (for grammar)
- boto3 latest (R2 access)
- datasets, parquet, etc.
- bf16 supported on Modal A100/A10G.

### 14.3 Code & data versioning

- All code committed to `sf-decision-stack`.
- Data parquet hashed (sha256), recorded in STATE.md per phase.
- Checkpoint hashes recorded.

### 14.4 Modal entry points

`modal_app.py::phase_1_smoke`, `phase_1_5_extract`,
`phase_2_featurize_probe`, `phase_3_sft_analyst`, `phase_4_sft_trader`,
`phase_4_5_eval`, `phase_5_grpo_analyst`, `phase_5_5_eval_analyst`,
`phase_6_grpo_trader`, `phase_6_5_eval_trader`, `phase_7_report`. Each
starts by reading the previous phase's checkpoint from the volume and
ends by `vol.commit()`.

---

## 15. Risk register

| # | risk | prob | impact | mitigation |
|---|---|---|---|---|
| R1 | R2 data is malformed / inconsistent | 0.20 | high | Phase 1.5 catches; budget reserve $15 |
| R2 | Qwen3-3B not yet on HF / unstable | 0.10 | low | fallback Qwen2.5-3B |
| R3 | GRPO doesn't converge in 1500 steps | 0.30 | medium | tune β, restart once; Phase 5.0 acceptance gates |
| R4 | KL collapse → policy degenerates | 0.20 | medium | rewind logic; Phase 5.0 abort condition |
| R5 | Reward function is mis-specified | 0.30 | high | design review now (this SPEC); sanity in Phase 5.0 first 100 steps |
| R6 | All-abstain attractor in analyst | 0.40 | high | reward shape penalizes pure-abstain via group baseline; gate at acceptance §9.7.3 |
| R7 | Test alpha is statistically zero | 0.40 | high | This is the project failing. NEGATIVE shipped per §2.3.1. |
| R8 | Counterfactual is non-inferior but not better | 0.50 | low | Acceptable per §2.3.2 wording. |
| R9 | Trader latency > 200ms | 0.30 | medium | switch to smaller base or rule-based fallback |
| R10 | Modal cost overrun | 0.10 | catastrophic | abort triggers in §12 |
| R11 | Data leakage between train/test (e.g., overlapping markets) | 0.20 | high | strict canonical-market split per §7.3 |
| R12 | News data is too sparse to feed analyst | 0.40 | medium | analyst still has indicators + history; news_block can be empty |
| R13 | Trader reward is too coarse to learn | 0.50 | medium | accept; trader v0 is more about "shows the schema" than alpha |

The two highest-impact and highest-probability risks are R5 (reward
mis-specification) and R6 (all-abstain attractor). The acceptance gates
at §9.7 are designed to catch both within the first 500 GRPO steps.

---

## 16. Decisions log (snapshot at SPEC freeze)

| ID | Decision | Status | Why |
|---|---|---|---|
| D-001 | Repo name `sf-decision-stack` | Patrick OK | `2026-04-28T19:43Z`, 其他都无所谓 |
| D-002 | Two-stage SFT-then-GRPO | Patrick OK | Fin-R1 recipe |
| D-003 | LoRA + numerical adapter, not pure RL | Patrick OK | preserves common sense |
| D-004 | ~~Distill from Gemini-flash~~ | **REJECTED** | Patrick: "我们那个portfolio 就是判断的不咋滴啊" — bad teacher |
| D-005 | Tick-data ingestion: defer | Patrick OK | top-of-book is enough for v0 |
| D-006 | Trader = Qwen3-1.7B with grammar | Patrick OK | latency budget achievable |
| D-007 | Analyst = Qwen3-3B | Patrick OK | iteration speed > peak quality at v0 |
| D-008 | **OPTION A: outcome-only training, no opinion distillation** | Patrick OK | the realized return is the only un-biased signal |
| D-009 | Hard cost cap $100, abort at $80 | Patrick OK | "不要 go over 100 一定不行" |
| D-010 | Decoupled — no live integration, no API exposure | Patrick OK | "完全去耦合 ... 我们就是测试一下" |
| D-011 | R2 archive as data source (not direct DB) | Patrick OK | cleanest decoupling |
| D-012 | Stage-1 SFT outputs are programmatic stubs (not Gemini-derived) | autonomous | preserves D-008's intent |
| D-013 | Reward clip at ±3 spread-units to prevent outlier domination | autonomous | numerical stability |
| D-014 | Strict time-and-canonical-market split for train/test | autonomous | no leakage |

---

## 17. Open questions (require Patrick or Phase-1 data to resolve)

1. **R2 bucket name and credentials.** The exact bucket name + access keys
   need confirmation from `simplefunctions-landing/.env.local` or
   `wrangler.toml`. Phase 1.0 verifies.
2. **How much history is in R2?** If only the last 7-10 days, train pool
   is too small. Phase 1.5 measures.
3. **Are `Qwen3-3B-Instruct` and `Qwen3-1.7B-Instruct` released on HF?**
   If not as of Phase 3.0 start, fall back to `Qwen2.5-3B-Instruct` and
   `Qwen2.5-1.5B-Instruct`. Document.
4. **Polymarket pricing in R2 dumps?** If Polymarket prices are not
   archived (only Kalshi), restrict to Kalshi-only training. Phase 1.5
   verifies.
5. **News table population.** `events` may be sparse. If so, drop
   `news_block` from prompts in v0.

---

## 18. Glossary

- **Alpha** — risk-adjusted excess return.
- **Edge** — the analyst's predicted alpha; the EE indicator.
- **GRPO** — Group Relative Policy Optimization (Shao et al. 2024).
- **LoRA** — Low-Rank Adaptation (Hu et al. 2021).
- **Soft tokens** — embedding-space inputs to the LLM that are not derived
  from text tokenization but from a learned encoder over numerical features.
- **Canonical market** — SF's internal abstraction over a market that may
  exist on multiple venues.
- **Conviction** — analyst's calibrated bet sizing in [0, 1].
- **Spread-units** — reward normalization where 1 unit ≈ one spread of
  realized P&L.
- **Counterfactual** — eval where we ask "would our model have done
  better than the prod stack on the same trades."

---

## 19. References

1. Shao et al., 2024. "DeepSeekMath: Pushing the Limits of Mathematical
   Reasoning in Open Language Models." arXiv:2402.03300. (GRPO origin.)
2. Liu et al., 2025. "Fin-R1: A Large Language Model for Financial
   Reasoning through Reinforcement Learning." arXiv:2503.16252. (Recipe template.)
3. Hu et al., 2021. "LoRA: Low-Rank Adaptation of Large Language Models."
   arXiv:2106.09685.
4. Hu et al., 2025. "LLM4TS: Large Language & Foundation Models for Time
   Series." github.com/liaoyuhua/LLM4TS. (Numerical adapter pattern.)
5. SF internal memories (in
   `~/.claude/projects/-Users-liuyizhou-spfunctions-simplefunctions-landing/memory/`):
   - `project_portfolio_automation.md` (2026-04-14)
   - `project_r2_dump_architecture.md` (2026-04-18)
   - `project_prompt_audit_2026-04-19.md` (Gemini-portfolio quality issues)
   - `project_full_stack_audit.md` (2026-04-14)
   - `feedback_silent_failures.md` (data-quality landmines)
6. ttt-conv-memory CLOSURE.md (predecessor's verdict + reusable artifacts).

---

## 20. Document version log

| version | date | change |
|---|---|---|
| v0 | 2026-04-28T20:55Z | initial draft, distill-from-Gemini teacher |
| v1 | 2026-04-28T22:30Z | Patrick rejected Gemini distillation; pivoted to Option A (outcome-only); rewrote teaching strategy, expanded reward functions, added grammar+adapter detail, expanded risk register, doubled length |

---

End of SPEC v1. Frozen pending Patrick `/compact`.
