# Design exploration — pre-SPEC-v2

**Status:** v0 of design exploration. Research notes + mini-experiments. Not yet a SPEC. This document precedes the v2 SPEC freeze and exists to surface design choices that v1 left as gut-feel decisions.

**Date:** 2026-04-28 evening (autonomous, Patrick reviewing pre-compact).

**Trigger:** Patrick: *"你说你会不会有点急 我觉得数据集和训练设计可能还能优化很多很多"*. Correct. v1 was first-instinct. This doc is the second pass.

**Method:** Read SF code (indicator computation, edge calculation), pull live samples from `/api/public/screen`, search 2025-2026 literature, validate v1 assumptions one by one.

---

## 0. Punch list — things that change vs. SPEC v1

In order of impact:

1. **EE must be excluded from training features.** It is LLM-tainted via `thesis/agents/formation.ts:394` (Claude Opus call). v1 included it implicitly. **§1.**
2. **Reward function v1 was a single-objective linear combination. Recent literature (2025-2026) converges on multi-objective composite rewards** with Sortino/Sharpe + drawdown + asymmetry components. v1 misses bet-sizing-aware reward and risk regularization. **§2.**
3. **Realistic upper bound is sobering.** Per Prediction Arena (arXiv 2604.07355v1), all six 2026-frontier LLMs lost 16-31% on Kalshi over 57 days with web search. A LoRA-tuned Qwen3-3B is unlikely to beat that. **Success criteria need to relax** from "positive alpha" to "non-inferior to abstain" or "calibrated conviction." **§3.**
4. **The market universe is mostly garbage.** Live probe of 200 markets from `/api/public/screen` (2026-04-29 ~05:38 UTC): median volume24h = $10, p25 = $0, 50% have price ≤5¢ or ≥95¢, only 1/200 in middle 30-70¢ band. Without explicit quality filtering, 90% of training states are noise. v1's stratification (40% high-edge / 30% high-volume / 20% pre-event / 10% random) is not enough — we need a *quality floor* before stratification. **§4.**
5. **Horizons should align with tau, not be one-size-fits-all.** v1's fixed `H ∈ {0.25, 1, 4, 24, 72, 168}` hours is wrong because most markets have tau=1-2 days. Horizons longer than tau are auto-resolution outcomes, not "trading P&L." **§5.**
6. **Numerical adapter v1 was a flat MLP.** Recent work (PatchTST, TsLLM) recommends patch-based encoder with **learned positional encoding** + **channel-independent** processing. Probably 2-5% absolute improvement, but free since the architecture cost is the same. **§6.**
7. **GRPO needs explicit handling of zero-variance prompts** (CameronWolfe GRPO++ tricks). v1 has no curriculum / no dynamic sampling. In a sparse-reward domain, ~80% of GRPO updates will get zero gradient unless we filter actively. **§7.**
8. **Polymarket should be over-represented in training**, not under. Per Prediction Arena, Polymarket has measurable inefficiency (LLM benchmark avg −1.1% vs. Kalshi −22.6%). v1 weighed venues by natural distribution; v2 should up-weight Polymarket. **§8.**
9. **Trader v0 viability is in serious doubt.** Reward signal for trader is coarse, top-of-book only, no real fill data. Honest answer: trader v0 should probably be **rule-based**, not LLM-trained. The LLM-trader idea ships when we have tick data. **§9.**
10. **No domain pretraining** in v1. Recent FinDaP / DACP papers show 2-5% gains from continual pretraining on domain corpus. We don't have a corpus, and our budget can't afford it. Document as future work, not v0. **§10.**

---

## 1. EE is LLM-tainted; exclude from features

### Evidence

`src/lib/indicators/compute.ts:265` defines EE as either `thesisEdgeCents` (LLM-derived) or `sfEdgeCents` (also LLM-derived via the regime path). Trace from the indicator audit:

```
src/lib/thesis/agents/formation.ts:394   generateObject() → Claude Opus
   "ASSUME the user's thesis is CORRECT" → edgeAnalysis.edges[].edgeSize
→ thesisState.edgeAnalysis
→ findEdgesForTicker() → marketRegimeSnapshots.sfEdgeCents
→ compute-indicators cron line 149-162
→ market_indicators.ee
```

Both paths are opinion, not data. **D-008 (Option A) forbids opinion supervision.** Therefore EE must be excluded from prompt features.

### Live data check

Pulled 200 markets at 2026-04-29 05:38 UTC:
- `iyYes`, `iyNo`, `cri`, `adjIy`: 200/200 = 100% coverage (clean)
- `rv`, `vr`, `iar`: 140/200 = 70% (clean, sparse early)
- `las`: 72/200 = 36% (clean, orderbook-sourced)
- **`ee`: 1/200 = 0.5% — basically empty** (and contaminated where present)
- `cvr`: 0/200 = 0%

EE was already practically absent. Excluding it is essentially free.

### Decision (D-015)

Drop `EE` from analyst input. Drop `EEsource`, `hasThesis` flag too (correlated). Use the 8 clean indicators: `iyYes, iyNo, cri, las, cvr, rv, vr, iar`. Add `adjIy` as a derived feature (it's `iy * exp(-vol_penalty - friction_penalty)`, deterministic from clean inputs).

### Replacing EE's role

EE was meant to be the model's "edge" signal. We should replace it with something purely empirical:
- **Realized recent edge**: for each market, compute `(midpoint_t - midpoint_{t-H}) / spread_t` over multiple lookback Hs. Pure price-history math, no LLM.
- **Resolution-implied probability vs. base rate**: for markets with similar templates (e.g., all "Will X happen by date Y?"), compute the empirical base rate of resolutions and compare to current price. Cross-market statistical, not LLM.

These can be added to the numerical adapter input.

---

## 2. Reward function — multi-objective composite

### v1 was

`R = clip(spread_normalized_pnl × conviction, ±3) + format_reward`

This is a **single-objective linear** reward. Two known problems:

1. Equally rewards "high conviction, marginal edge, big bet" and "moderate conviction, big edge, moderate bet" if they have the same expected return. Real trading prefers the latter (Kelly-criterion-aware sizing).
2. No risk penalty. A 50% chance of +$10 / 50% chance of -$10 has the same reward as 50% chance of +$1 / 50% chance of -$1. The model can't learn to avoid high-variance trades.

### Literature consensus (2025-2026)

- Sun et al., 2025 (arXiv 2506.04358) propose a composite reward of (annualized return + downside risk + differential return + Treynor ratio). Each is differentiable.
- Multiple 2025 papers use Sortino + Sharpe + Calmar simultaneously to "avoid reward hacking" common with single metrics.
- Prediction Arena: best model's win profile was **+$63.89 when correct, −$3.23 when wrong** — pure asymmetry, *not* high hit rate. Reward should capture this.

### v2 proposal — composite reward for analyst

```
For (state s_t, action a_t = (side, conviction, horizon H)):

raw_pnl   = signed_return(side, midpoint_t, midpoint_{t+H})  in cents per unit
norm_pnl  = raw_pnl / max(spread_t, 2)                       # spread-units
abs_size  = conviction                                       # bet size proxy

# 1. Expected-return component
r_return  = clip(norm_pnl * abs_size, -2, 2)

# 2. Asymmetry bonus (reward big wins, lightly penalize losses)
r_asymm   = if norm_pnl > 0: norm_pnl^1.5 * abs_size
            else:           norm_pnl     * abs_size * 0.5
r_asymm   = clip(r_asymm, -1, 1)

# 3. Calibration penalty (conviction × outcome consistency)
# If high conviction but wrong, penalize harder
if abs_size > 0.5 and sign(predicted_alpha) != sign(realized_alpha):
    r_calib = -0.5 * abs_size
else:
    r_calib = 0

# 4. Sortino-flavored downside-risk penalty (per-trade, not portfolio)
# Approximate: penalize trades where intra-horizon drawdown > 2× spread
r_drawdown = -0.3 if observed_intra_horizon_drawdown > 2*spread_t else 0
# Computed by checking max(|midpoint_τ - entry_midpoint|) for τ ∈ [t, t+H]

# 5. Format reward
r_format  = +0.2 if grammar passes else -0.5

# 6. Abstain handling (separately rewarded)
if side == "abstain":
    r_total = 0.05  # tiny reward for refusing when uncertain (avoid "always trade")
else:
    r_total = r_return + 0.5*r_asymm + r_calib + 0.3*r_drawdown

R = r_total + r_format
```

Properties:
- Linear in expected return (so the model has a clear learning signal).
- Asymmetric (rewards big wins more than big losses, capturing the Prediction Arena finding).
- Penalizes overconfidence on losers.
- Includes intra-horizon drawdown — approximated, since we only have 15-min snapshots.
- Abstain has a small positive reward (0.05) — enough that always-abstain has ER = 0.05, but a confidently-correct trade has ER ≈ 1+. Avoids both all-abstain collapse and forced participation.

### v2 proposal — trader reward

Honestly, we should defer trader training (§9). For completeness, if we do train v0:

```
r_format         = +0.2 if grammar passes else -0.5
r_action_match   = +0.3 if intent matches analyst.side else (-0.5 if mismatched else 0)
r_simulated_fill = -|fill_price - midpoint_t| / spread_t    # in spread-units
r_risk           = -1.0 if violates portfolio gate else 0

R = r_format + r_action_match + r_simulated_fill + r_risk
```

Coarse on purpose. We don't have real fill data; pretending we do is worse than admitting we don't.

### Decision (D-016)

Replace v1's linear reward with the composite above. Specific weights (0.5, 0.3, 0.05) are starting points; tune in Phase 5.0 first 100 steps based on reward distribution diagnostics.

### Open question

The intra-horizon drawdown estimator is approximate. We have `market_price_snapshots` at 15-min cadence, so for `H ≥ 1h` we get reasonable density. For H = 15 min, drawdown is just one tick — drop the term. Document the H-dependent precision.

---

## 3. Realistic upper bound; relax success criteria

### Evidence: Prediction Arena (arXiv 2604.07355v1, ran Jan 12 - Mar 9, 2026)

Six frontier LLMs (Opus 4.5, GPT-5.2, Gemini 3 Pro, Grok 4.20, Grok 4.1 fast, GLM 4.7) traded $10k each on Kalshi + Polymarket with web search and 15-45 min cadence:

| | Kalshi 57d | Polymarket 28d |
|---|---|---|
| best | -16.0% (GLM 4.7) | +6.02% (Gemini 3.1 Pro, 3 days only) |
| median | ~-22% | -1.1% |
| worst | -30.8% (Grok 4.1 fast) | — |
| only profitable historical run | grok-4-20-checkpoint +10.9% (mystery-model-alpha, earlier; +$63.89 win vs −$3.23 loss asymmetry) |

**All six frontier models lost on Kalshi.** Weather markets dominated 71-97% of settlements (the benchmark became largely a weather-forecasting eval).

### Implication for our success criteria

A LoRA-tuned Qwen3-3B with no web search is **not going to beat Opus 4.5 with web search at this task** under realistic budget. v1's success criterion §2.3.1 ("positive alpha at 95% CL") is too aggressive.

### v2 success criteria proposal

| # | criterion | v1 | v2 (relaxed) | rationale |
|---|---|---|---|---|
| 1 | analyst alpha | positive at 95% CL | **non-negative at 90% CL** | matching the universe's ~−22% baseline is itself a win |
| 2 | counterfactual vs. portfolio-v2 | non-inferior at 90% | **non-inferior at 80%** | weak comparator, weaker requirement |
| 3 | trader slippage bound | 95% within visible spread | **drop trader from v0** (§9) | rule-based fallback |
| 4 | latency | p95 ≤ 200ms / 3s | unchanged | physically achievable, our promise |
| 5 | grammar | 100% schema-valid | unchanged | engineering correctness |
| **NEW 6** | **calibration** | — | **conviction-vs-realized correlation > 0.05 at 95% CL** | the model is at least *calibrated*, even if not profitable |
| **NEW 7** | **abstain quality** | — | **abstain rate in [0.4, 0.85]** | avoid both modes of collapse |

### Decision (D-017)

Adopt the relaxed v2 criteria. Add calibration and abstain-rate criteria. Treat "matching the abstain baseline" (≈ 0%) as a partial win.

### What "success" actually means in v2

- **Best case (10% probability):** model has small positive alpha on Polymarket, calibrated convictions. Ship as v0, iterate.
- **Likely case (50% probability):** model is near-zero alpha but calibrated, latency-OK. Useful as a probability estimator, not a money-printer. Ship as research artifact.
- **Worst case (40% probability):** model is negative alpha but calibrated. Document as NEGATIVE; the calibration alone is a finding.
- **Failure mode (rare):** model collapses to all-abstain or all-buy. Phase-5 acceptance gate catches this.

Honest framing for Patrick.

---

## 4. State filtering — quality floor before stratification

### Evidence: live probe of 200 markets (2026-04-29)

Without filtering:
- `volume24h`: median $10, p25 = $0, p90 = $2,288, max = $2.9M (2899021).
- `price`: median 9¢, only 1/200 in middle 30-70¢, 50% in extreme ≤5/≥95.
- `tauDays`: median 1, p90 = 3, max = 11.

The default screener sorts by IY descending, which biases toward extreme prices (high IY = high implied yield = mostly very low or very high price). Even without that bias, 21k of 35k total universe survive `excludeSports + excludeNoise` → still mostly noise.

### v2 quality filter (applied before stratification)

Inclusion criteria for a state (s_t, ticker, venue) to enter training:

```
include_state(s):
    s.price >= 8 and s.price <= 92                   # avoid resolved-near-extreme
    s.volume24h >= 200                               # actual liquidity (USD)
    s.tauDays >= 1.5                                 # at least 36h to expiry
    s.tauDays <= 60                                  # not perpetual; LLM context fits
    s.ind_coverage >= 6                              # at least 6 of 8 clean indicators populated
    not s.is_sports                                  # already in excludeSports
    not s.title_matches_noise_regex                  # SF's noise filter
```

After applying this, the state pool will likely be **2-5k markets at any given time** (rough estimate from the 200-sample probe; need Phase 1 to confirm). With 60-90 days of train history at 15-min cadence, that's 100k-300k *raw* state observations, but most are correlated (same market, adjacent times). Effective independent samples ≈ 5-10k. Adequate for SFT and feasible for GRPO.

### Stratification *after* the filter

Within the filtered pool:
- 50% **high-volume, mid-price, mid-tau** (the "tradeable" core)
- 20% **near-event** (within 24h of a `legislation_market_pairs` or `events` row)
- 15% **regime-shift** (states where SF Index v2 changed > 1 std from previous snapshot)
- 10% **cross-venue gap > 2 cents** (arb opportunities — feature, not target)
- 5% **random** (tail diversity)

Note that v1's "high-edge" stratum is gone — we excluded EE. Replace with empirical proxies (recent realized return absolute value, indicator deviation from rolling mean).

### Decision (D-018)

Apply the inclusion criteria above. Re-stratify per the new five buckets. Validate sample count in Phase 1.5; if filtered pool is <2k markets, relax thresholds.

### Open question

`s.ind_coverage >= 6` may be over-restrictive in early periods of the data. Phase 1.5 measures coverage as a function of timestamp.

---

## 5. Horizon design — align with tau

### v1 was

`H ∈ {0.25, 1, 4, 24, 72, 168}` hours, sampled randomly per state.

### Problem

If `tauDays = 1`, then horizons of 72h or 168h are auto-resolution outcomes (the market has already settled). The model would always "win" or "lose" by H=72h on a 1-day market — depending only on whether its initial side matched the resolution. That's not training to predict price movement; it's training to predict resolution outcome.

### Live data — most markets are short-tau

p90(tauDays) = 3. Median = 1. Vast majority of markets are 1-3 day markets.

### v2 proposal — `H_eff = min(H_predicted, tau_remaining * 0.8)`

Let the model predict `H_predicted` from `{15min, 1h, 4h, 24h, 72h}`. For the realized-return computation:
- Use `H_eff = min(H_predicted, 0.8 * tau_remaining_hours)` so we always evaluate before resolution.
- If `H_eff < H_predicted`, add a small penalty to the reward: `r_horizon_mismatch = -0.2 * (1 - H_eff/H_predicted)`. This discourages predicting horizons longer than tau.

Alternatively, and arguably cleaner:
- Train with **two separate prompts per state**: one short-horizon (15min-4h) and one long-horizon (24h-tau). The model emits a single decision matched to the prompt's horizon scope. Reward computed at exactly the matched horizon.

The second option is cleaner because the model doesn't need to learn the awkward "predict horizon and we'll cap it" rule. Default to the second option in v2.

### Decision (D-019)

Per state, generate two training prompts: one for `H_short ∈ [15min, 4h]` and one for `H_long ∈ [24h, 0.8·tau]`. Doubles dataset size; halves the effective unique states; OK in budget.

### Open question

When `tau < 30h`, `H_long` becomes degenerate (≤ 24h). For these states, just one prompt (short-horizon) per state. Phase 2.0 implements this branch.

---

## 6. Numerical adapter — patch-based with positional encoding

### v1 was

Flat MLP `(T=192, F=9) → 1728 → 1024 → 1024 → 32 × hidden_dim`. K=32 soft tokens. No time positional info.

### Problem

The MLP destroys temporal structure. The model can't tell whether the spike is at t=−5 or t=−180. For a time-series task, that's a known weak architecture.

### Literature recommendation (PatchTST, TsLLM, 2025)

- **Patch-based encoding**: divide T timesteps into P patches of size T/P. Apply a small linear projection per patch. Sum with **learned positional encoding** per patch position. Process through a 2-layer transformer. Output K soft tokens.
- **Channel-independent**: each indicator (F=8 in v2 since EE is dropped) gets its own patch sequence. The 8 sequences are processed in parallel with a shared transformer, then aggregated.
- TsLLM specifically uses a `<|ts|>` placeholder token whose embedding is replaced with the time-series encoding at runtime — same shape as v1's `<MARKET_STATE_TOKENS>`.

### v2 proposal

```
input: indicator history (T=96, F=8 indicators)
       T=96 means 96 × 15-min = 24h of context
       F=8 = the 8 clean indicators after dropping EE

Step 1: per-channel patch encoder
  For each of 8 channels:
    patches = reshape (96,) into (12 patches, 8 timesteps each)
    patch_embed[i] = Linear(8 → 128)(patch[i])
    position_embed[i] = LearnedEmbedding(i)  for i ∈ [0, 12)
    embed[i] = patch_embed[i] + position_embed[i]

Step 2: aggregate channels
  channel_token[c] = TransformerEncoder(channel c's 12 patch embeds, 2 layers)
                     [returns 1 vector per channel]

Step 3: cross-channel pooling → K soft tokens
  pooled = stack(channel_token[c] for c ∈ [0, 8))   shape (8, 128)
  K_tokens = Linear(8 → K=24)(pooled.T).T            # (K=24, 128)
  soft_tokens = Linear(128 → hidden_dim)(K_tokens)    # (24, hidden_dim)

Total params: ~5M (vs v1's ~3.5M MLP). Free given LoRA's 30M.
```

K=24 (down from 32). 12 patches × 8 timesteps = 96 timesteps lookback (24h at 15-min cadence). Channel-independent. Learned positional encoding. Shared transformer for all 8 indicators.

### Decision (D-020)

Adopt patch-based architecture. T=96 (24h, 15-min cadence). F=8 (clean indicators). Patch=8 timesteps (=2 hours). 12 patches per channel. K=24 soft tokens.

### Trader adapter

Same architecture, smaller: T=32 (8h), patch=4, 8 patches, F=8, K=12. Total ~2M params.

### Open question

Should we add raw price history as a separate channel? Currently price is encoded *inside* the indicators (IY and CRI both depend on price). Adding raw price gives the model first-derivative signal that indicators alone don't capture cleanly. Let's add it: F=9 = 8 indicators + 1 mid-price-normalized-to-spread series. K stays 24.

---

## 7. GRPO++ tricks for sparse-reward setting

### Findings (CameronWolfe, GRPO++ post)

1. **Dynamic sampling**: drop prompts where all K candidates score the same reward. In our domain, "all 8 candidates abstain" or "all 8 candidates buy YES" produces zero gradient. Detect and skip these batches. Refresh by oversampling other prompts.
2. **Asymmetric clip**: standard PPO uses ε=0.2 for both upper and lower bounds. GRPO++ uses ε_high=0.28 to allow exploration of low-prob tokens. In our domain, "abstain" is the dominant low-prob option from a base model; we want it explored.
3. **Length-aware reward**: soft penalty over an interval, not hard cliff. Already in our v2 reward (no length term yet — should add).
4. **Conservative or omitted KL**: many recent reasoning models drop KL entirely after initial steps, allowing the policy to drift further. In our 1500-step regime with format-SFT init, we keep KL but anneal: β starts 0.04, decays to 0.01 over 1000 steps.
5. **Curriculum / Pass@K filtering**: select prompts where the base policy has 2-6 of 8 correct answers (i.e., genuine learning signal). Easy and impossible prompts contribute nothing.

### v2 GRPO additions

```
At each step:
  1. Sample 4× the batch size of prompts (oversample)
  2. For each, sample K=8 with grammar enforcement
  3. Compute rewards
  4. Filter: keep only prompts where reward variance > 0.1
  5. Compute advantages on filtered set
  6. Compute KL with current β (annealing)
  7. Apply asymmetric clip: ε_low=0.2, ε_high=0.28
  8. Update policy
  9. Log: filter_keep_rate, reward variance histogram, abstain rate
```

If `filter_keep_rate < 0.3` for 50 consecutive steps, the prompt distribution is ill-formed (too many trivial cases). Stop and fix the data filter.

### Decision (D-021)

Add dynamic sampling, asymmetric clip, KL annealing to v2 GRPO. Add explicit acceptance gate on `filter_keep_rate ≥ 0.3` over rolling 50 steps.

### Hard truth

GRPO at ~1500 steps with K=8 is genuinely small for a sparse-reward task. Frontier reasoning models train for 10s of thousands of steps with K=64+ on much denser reward signals. We're aiming for "learn the schema and avoid mode collapse," not "achieve frontier reasoning." Document this explicitly so v0 isn't oversold.

---

## 8. Polymarket weighting

### Evidence

Prediction Arena: Kalshi −22.6% avg over 57d, Polymarket −1.1% avg over 28d (with `gemini-3.1-pro-preview` at +6.02% in 3 days). Polymarket is structurally more inefficient; LLMs find more edge.

### Live data

Of 200 markets sampled, Kalshi 154 (77%), Polymarket 46 (23%). Natural distribution skews toward Kalshi.

### v2 proposal

Up-weight Polymarket states 3× during training stratification:

```
target_distribution:
  kalshi:     (1/3) of train pool
  polymarket: (2/3) of train pool
```

This intentional bias helps the model learn the venue-specific dynamics where edge actually exists.

### Decision (D-022)

3× upweight Polymarket. Eval is reported by-venue separately so we can see which venue drives the alpha.

### Risk

Polymarket has weirder ticker formats (0x-hex conditional IDs) and sometimes degraded indicator coverage. Phase 1.5 measures and adjusts.

---

## 9. Trader v0 viability — defer

### Honest assessment

Given:
- We have only top-of-book (no depth, no queue position).
- We have no real fill data; "simulated fills against historical TOB" is fiction.
- The action space is small enough (action ∈ {place, cancel_all, hold} × type ∈ {market, limit} × side × price × size) that a rule-based policy can handle 95% of cases.
- Training a 1.7B LLM with grammar takes the same Modal budget as a 3B analyst and produces output of unverified value.

The trader-as-LLM plan was driven by Patrick's "human trader intuition" requirement. But intuition is *analyst's* job. Trader is mechanical execution.

### v2 proposal

**Drop trader LLM training from v0 scope.** Replace with `src/trader/rule_baseline.py`:

```python
def trade(analyst_decision, state):
    if analyst_decision.side == "abstain":
        return Intent("hold")
    
    spread = state.ask - state.bid
    midpoint = (state.ask + state.bid) / 2
    size = analyst_decision.conviction * MAX_POSITION
    
    if state.position_in_market.size >= size:
        return Intent("hold")  # already at target
    
    if spread <= 2:
        return Intent("place", "market", side, ask if buy else bid, size, "ioc")
    else:
        # Mid-or-better limit
        price = midpoint - 1 if buy else midpoint + 1
        return Intent("place", "limit", side, price, size, "gtc", expiry=300)
```

Frees ~$20-25 from trader Stage 2 GRPO. Reallocates to (a) more analyst SFT data, (b) longer analyst GRPO, (c) reserve.

### What we don't lose

The analyst still produces full structured decisions including conviction and horizon. The trader rule consumes those. End-to-end behavior is preserved. The only loss is "small LLM that thinks about execution" — which we couldn't credibly train anyway.

### Decision (D-023)

Drop trader LLM from v0. Implement rule-based baseline. Revisit when tick data arrives.

### Budget impact

| | v1 | v2 |
|---|---|---|
| Phase 4.0 trader SFT | $0.50 | $0 (rule-based) |
| Phase 6.0 trader GRPO | $20 | $0 |
| Phase 6.5 trader eval | $1 | $0.20 (rule-based bench) |
| **subtotal saved** | | **$21.30** |

Plan total: $56.20 → $34.90. Reserve becomes $35. Total budget $69.90 vs. cap $100.

---

## 10. Domain pretraining — defer (note as future work)

### Findings (FinDaP, DACP papers, 2025)

Continual pretraining on domain corpus before instruction tuning gives 2-5% absolute improvement on financial tasks at 1-3B scale. Standard recipe:
1. Crawl domain corpus (we'd need: prediction-market-specific texts — Kalshi market descriptions, Polymarket events, financial news, possibly forecasting blogs).
2. ~1B tokens of continual pretraining.
3. Then SFT, then RL.

### Why we skip this in v0

- Corpus collection is its own project ($X, several days).
- 1B tokens × Qwen3-3B continual pretraining ≈ $50-100 by itself. Eats our budget.
- Without confidence that v0 SFT+GRPO produces *anything*, paying for pretraining first is premature.

### v2 mitigation

- Use Qwen3-3B-Instruct (not base). Instruct version has substantial chat / reasoning capability that approximates "common sense" Patrick wanted.
- Include explicit market-context glossary in the system prompt (§5.1.1 of SPEC v2): definitions of IY, CRI, etc. This is cheap and gives the model anchor terms.
- If v0 retrieval + reasoning quality is poor, document as "needs domain pretraining" and the next iteration tries it.

### Decision (D-024)

No domain pretraining in v0. Document as future work in SPEC v2 §17.

---

## 11. Summary — concrete v2 deltas vs v1

| section in v1 | v1 | v2 |
|---|---|---|
| §1 Executive | 2 LLMs analyst+trader | 1 LLM analyst, rule-based trader |
| §2.3 Success | "alpha statistically positive at 95%" | non-negative at 90% + new calibration criterion |
| §4.1 Analyst inputs | includes EE | 8 clean indicators, no EE; add price-normalized-spread |
| §4.2 Trader | Qwen3-1.7B + grammar | rule-based, no LLM |
| §6.3 Numerical adapter | flat MLP (T=192, F=9, K=32) | patch-encoder + transformer (T=96, F=9 incl. price, K=24) |
| §6.4 Output grammar | analyst + trader | analyst only |
| §7.3 Train/val/test | stratified by EE+volume+events | quality-filtered first, then stratified by 5 new buckets |
| §7.4 Stratification | high-edge/high-volume/pre-event/random | 50% tradeable core, 20% near-event, 15% regime-shift, 10% cross-venue, 5% random |
| §7.4 Venue weights | natural | 3× upweight Polymarket |
| §8 Stage 1 SFT | both models | analyst only (~$1.50) |
| §9.2.1 Reward | linear, conviction-weighted, ±3 clip | composite (return + asymmetry + calibration + drawdown + format), abstain bonus |
| §9.3 GRPO sampling | K=8 vanilla | K=8 + dynamic sampling on reward variance |
| §9.4 KL/clip | β=0.04 const, ε=0.2 sym | β anneal 0.04→0.01, ε_low=0.2 ε_high=0.28 |
| §9.6 Logging | basic | + filter_keep_rate, abstain_rate trend |
| §9.7 Acceptance | 5 gates | 5 gates + filter_keep_rate ≥ 0.3 |
| §13 Phase ledger | 13 phases, $71 plan | 11 phases (drop trader 4/6/6.5), $35 plan |
| §15 Risk register | 13 rows | + R14 EE-contamination found and fixed; +R15 trader scope-cut bias |

### New decisions (D-015 through D-024)

- D-015: Drop EE from features (LLM-tainted).
- D-016: Composite reward (return + asymmetry + calibration + drawdown).
- D-017: Relaxed success criteria; add calibration metric.
- D-018: Quality-filter states before stratification.
- D-019: Per-state two-prompt horizon design (short + long).
- D-020: Patch-based numerical adapter, T=96, K=24, F=9 incl. price.
- D-021: GRPO++ tricks (dynamic sampling, asymmetric clip, KL annealing).
- D-022: 3× Polymarket up-weight in training pool.
- D-023: Drop trader LLM from v0; rule-based baseline.
- D-024: No domain pretraining in v0.

### Updated cost plan

| phase | $ |
|---|---|
| 1.0 env + R2 | 0.00 |
| 1.5 extract | 0.50 |
| 2.0 featurize + adapter probe | 1.00 |
| 3.0 SFT analyst | 1.50 |
| 4.5 post-SFT eval | 0.20 |
| 5.0 GRPO analyst | 30.00 |
| 5.5 analyst eval | 1.00 |
| 7.0 final report + counterfactual | 0.50 |
| 8.0 closure | 0.00 |
| **subtotal** | **34.70** |
| reserve | 35.00 |
| **plan total** | **69.70** |
| **HARD CAP** | **100.00** |

Headroom: ~$30 above plan.

---

## 11.5. Mini-experiment results (2026-04-29 ~05:50 UTC)

### Experiment §12.1 — indicator predictive power (preliminary)

Pulled `/api/public/market/{ticker}/history?hours=72` for 30 markets meeting a loose
quality filter (price ∈ [10, 90], volume24h > 0). 17 markets had ≥30 history rows;
4,372 samples total. Computed correlations between current state (price, CRI,
15-min delta) and forward returns at 4h and 24h.

**Pooled cross-market Pearson correlations (n=4,372):**

| | ret_4h | ret_24h |
|---|---|---|
| price_t | −0.18 | −0.32 |
| CRI | +0.08 | +0.06 |
| delta_15min | −0.17 | −0.12 |

**Per-market `price_t → ret_4h` correlation distribution (17 markets):**

| | value |
|---|---|
| median | −0.84 |
| mean | −0.62 |
| positive markets | 1 / 17 |
| top market: KXJOBLESSCLAIMS | n=15, corr=−0.979 |
| top weather: KXHIGHMIA | n=27, corr=−0.924 |
| KXLCATTLEW | n=82, corr=−0.888 |

**Mean return all samples (24h forward), by venue:**
- Kalshi: n=1,974, mean=+0.47¢, std=10.66¢
- Polymarket: n=1,693, mean=+0.18¢, std=14.45¢

### Reading

The per-market correlation of −0.84 is **enormous** for finance. Three possible
interpretations, in decreasing order of "tradeable":

1. **Real mean reversion within each market.** Prices oscillate around a slow-moving
   fair value; extreme deviations get faded.
2. **Resolution-convergence artifact.** Markets resolve at a fixed time. As tau→0,
   price→outcome ∈ {0, 100}. If outcome is, say, 50, then early prices at 80 will
   come down to 50; that registers as negative price→ret correlation but is *not*
   tradeable — the bias only manifests as resolution approaches.
3. **Sample-period artifact.** Most of the sampled markets are weather / 1-day
   commodity markets that resolved within the 72h history window. Selection bias.

The pooled cross-market correlation (−0.18) is much weaker than per-market median
(−0.84). This pattern is *consistent with* both real mean-reversion (each market
reverts around its own mean, but means differ across markets) and resolution
convergence (as different markets resolve at different price levels).

### What this means for the SPEC

- The signal is **detectable in the data**, but the test was confounded by
  resolution-convergence. A clean test requires controlling for time-to-resolution
  (computing ret over horizons that are uniformly < tau, not crossing resolution).
- The pooled cross-market correlation (−0.18 to −0.32 on price_t) is itself
  **non-trivial** and *would* be tradeable if it generalizes. Need to confirm with
  a larger out-of-resolution sample.
- **Per-market normalization is critical.** A model trained on raw prices across
  heterogeneous markets will see the −0.18 pooled signal and miss the per-market
  −0.84 signal. v2's adapter should normalize price by per-market rolling mean
  (e.g., price_z = (price_t − rolling_mean_24h) / rolling_std_24h), giving the
  model a clean mean-reversion feature.
- The simplest baseline strategy ("fade prices > rolling mean") may already
  capture most of the alpha. **The LLM has to add value beyond this baseline** to
  justify its existence.

### Decision (D-025)

Add per-market price normalization (`price_z`) as the 9th adapter input channel
(replacing or augmenting the raw mid-price). Baseline strategy (rule-based mean
reversion) is added to the eval suite as a comparator that the LLM must beat.

### Decision (D-026)

The mini-experiment is preliminary (1 market = 412 samples dominated, then 17
markets; weather + commodity heavy). Phase 1.5 must repeat at scale (≥ 200
markets, ≥ 30-day history per market, controlled for resolution timing) before
SPEC v2 final freeze. If the signal disappears under proper time-to-expiry
controls, **the project pivots or stops.**

### Caveats

- Sample is from 2026-04-27 18:00 UTC to 2026-04-29 05:50 UTC; one cross-section
  of "market state at one moment in time, looking back 72h."
- The 4h horizon assumes 16 indicator rows, but cadence is uneven. The actual
  H_eff varies. Approximation is fine for a punchline; not for SPEC v2 numbers.
- Public API only exposes `iy`, `cri`, `price`, `delta` in `indicatorHistory`.
  Cannot test predictive power of RV, VR, IAR, LAS, CVR via this endpoint.
  Phase 1.5 (DB or R2 access) probes the full 8-indicator set.

---

## 12. Open mini-experiments before SPEC v2 freeze

These are ≤ $0 / local-only validations to do *before* writing v2. Goal: catch any remaining false assumptions.

1. **Indicator predictive power.** Pull last 30 days of `market_indicator_history` + `market_price_snapshots` for 100 markets. For each (state, indicator deciles), compute realized 4h forward return. If the top deciles of CRI, RV, IAR, LAS, etc. don't show separation in realized return, **the indicators have no predictive power** and the project's premise is broken. **This is the most decisive single check.**
2. **Polymarket coverage.** Confirm Polymarket markets actually have indicator coverage in our public API. Live probe showed 23% Polymarket presence but coverage may be lower. If Polymarket indicators are sparse, the 3×-up-weight plan needs adjustment.
3. **News alignment density.** How many `events` rows are linked (via `legislation_market_pairs` or vector similarity) to test-pool markets? If <10% of markets ever have a linked event in the prior 7 days, news is feature noise; drop it from prompts.
4. **Inference latency ground truth.** Modal scratch run on Qwen3-3B with vLLM grammar-constrained decoding. Measure p95. If p95 > 5s, we have a problem (the v2 budget assumes 1-3s).
5. **Resolution-time distribution.** What fraction of training markets resolve mid-horizon? If high (>30% on H=24h), we need careful handling; my §5 H_eff cap helps.

(1) and (3) are local-only (just SQL/analysis). (2) is API-only. (4) is the only one that costs Modal $.

### Recommendation

Do (1), (2), (3), (5) locally before freezing SPEC v2. Defer (4) into Phase 1 itself (it's part of feasibility check).

If (1) shows zero indicator predictive power, **the project pivots to a different data shape entirely** (price-history-only, news-only, or different lookback) — or the verdict is NEGATIVE before training starts.

---

## 13. References

1. Sun et al., 2025. "A Risk-Aware Reinforcement Learning Reward for Financial Trading." arXiv:2506.04358. (Composite reward design.)
2. Yang et al., 2025. "Risk-Adjusted Deep Reinforcement Learning for Portfolio Optimization: A Multi-reward Approach." Springer. (Multi-objective rewards.)
3. Wolfe, 2026. "GRPO++: Tricks for Making RL Actually Work."
   cameronrwolfe.substack.com/p/grpo-tricks. (Dynamic sampling, asymmetric clip, length-aware penalty.)
4. Lin et al., 2026. "Prediction Arena: Benchmarking AI Models on Real-World Prediction Markets." arXiv:2604.07355v1. (Frontier-LLM trading benchmark; sobering upper bound.)
5. Liu et al., 2026. "TsLLM: Augmenting LLMs for General Time Series Understanding and Prediction." arXiv:2510.01111v2. (Patch-based time-series adapter.)
6. PatchTST: Nie et al., 2023. arXiv:2211.14730. (Channel-independent patch transformer.)
7. Anderson et al., 2025. "Demystifying Domain-adaptive Post-training for Financial LLMs (FinDaP)." arXiv:2501.04961. (Domain pretraining methodology.)
8. SF internal: indicator audit (this session, 2026-04-28). EE LLM-contamination identified at `src/lib/thesis/agents/formation.ts:394` → `market_indicators.ee`.
9. SF internal: live data probe via `/api/public/screen?limit=200` at 2026-04-29 ~05:38 UTC. Median volume24h = $10, 50% of markets at price ≤5 or ≥95.

---

## 14. Status & next step

- This doc is **v0 of design exploration**. Not yet binding.
- Next actions (in order):
  1. Do mini-experiments §12.1, §12.2, §12.3, §12.5 locally (~half day).
  2. Patch this doc with results.
  3. Write SPEC v2 incorporating §11 deltas + mini-experiment findings.
  4. Patrick reviews v2.
  5. /compact and start Phase 1.0.

---

End of design exploration v0. Awaiting review.
