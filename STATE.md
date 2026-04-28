# STATE — sf-decision-stack live progress log

Each entry stamped with UTC timestamp. Updated continuously.

## Current phase

**Phase 0 — spec + state files written, awaiting Patrick review.**
No code yet. No data downloaded. No money spent.

## Phase ledger

- [x] Phase 0: spec + state (this commit)
- [ ] Phase 1: feasibility check (row counts, latency probe, adapter shape)
- [ ] Phase 2: SFT data pipeline (offline)
- [ ] Phase 3: SFT analyst (~$10)
- [ ] Phase 4: SFT trader (~$5)
- [ ] Phase 5: GRPO analyst (~$15)
- [ ] Phase 6: GRPO trader (~$15)
- [ ] Phase 7: eval + benchmark (~$5)
- [ ] Phase 8: deployment shape
- [ ] Phase 9: A/B vs Gemini teacher

## Detailed log

### 2026-04-28T20:30Z — kickoff (autonomous, Patrick asleep)

Patrick's instruction (paraphrased): build two small low-latency LLMs over SF
data — analyst that produces alpha, trader that executes. Common-sense from
pretrained base. Inputs: user intent, market data (orderbook/history/correlation/
index/beta), news. Output: decisions. New private repo. Spec + state first,
no code yet, nothing irreversible.

### 2026-04-28T20:35Z — ttt-conv-memory closed out

The previous repo (`spfunctions/ttt-conv-memory`) was pushed and abandoned with
a CLOSURE.md. Iteration-2 result: even at α=0.1 inference scaling (smallest
TTT perturbation), gold appears in prediction 0/259. Past_w encodes nothing
under the frozen-base + short-Q&A setup. To flip the verdict would require
joint base+TTT training at $20-30+ — beyond remaining free Modal credit.
Patrick chose to abandon.

### 2026-04-28T20:38Z — repo created

Created `spfunctions/sf-decision-stack` (private) via `gh repo create`. Cloned
to `/Users/liuyizhou/spfunctions/sf-decision-stack`. Empty.

### 2026-04-28T20:40Z — research dispatched in parallel

- **Explore** subagent: SF data inventory (DB schema, market data, news,
  R2 archives, API endpoints). Returned a 9-section ~250-line report with
  table-formatted findings. Headlines: 9 indicators 4×/hr, ~9.8k markets at
  5-min, top-of-book only orderbook (gap!), Twitter not stored (gap), R2
  daily archives of 33+ tables.
- **WebSearch** for "small LLM fine-tuning numerical adapter time series 2025-2026":
  surfaced LLM4TS framework, PEFT (LoRA/QLoRA/DoRA/Spectrum), Unsloth single-GPU
  recipes, Sebastian Raschka's State of LLMs 2025.
- **WebSearch** for "GRPO RLHF small LLM 1B 3B trading":
  surfaced Fin-R1 (Qwen2.5-7B + GRPO, 75.2 avg on FinBench), llm-stats post-training-2026
  overview, Unsloth RL guide, hybrid LLM-strategy + RL-execution pattern.
- **WebSearch** for "LLM trading agent sub-100ms latency":
  confirmed pure LLM HFT is hard; hybrid is the consensus pattern.
- **WebFetch** of Fin-R1 paper: confirmed two-stage SFT-then-GRPO recipe with
  binary format reward + accuracy reward; 60K SFT samples; results at 7B
  beating much larger models.

### 2026-04-28T20:55Z — SPEC.md written

12 sections, ~280 lines. Architecture choice: LoRA + numerical adapter on
Qwen3 base (analyst 3B/7B, trader 0.5B/1.7B), two-stage SFT-then-GRPO per
Fin-R1 recipe. SFT teachers: trade_ideas, thesis_state evals, Gemini
portfolio-LLM logs. GRPO rewards: format + realized PnL + slippage.

### 2026-04-28T21:00Z — STATE.md (this file) written

## Decisions log

| ID | Decision | Why | Risk |
|---|---|---|---|
| D-001 | Repo name = `sf-decision-stack` | Says what it is (analyst+trader stack). Patrick may rename. | Renaming is reversible. |
| D-002 | Two-stage SFT-then-GRPO per Fin-R1 | Proven recipe at 7B for financial domain. | Reward design hardest part. |
| D-003 | LoRA + numerical adapter, not pure RL | Patrick wants common-sense / intuition; pure RL discards language. | Adapter shape needs Phase 1 probe. |
| D-004 | Distill from Gemini 2.5 Flash teacher | Patrick's prod portfolio-v2 is the labeled-decision oracle. | Teacher access + log completeness need verifying. |
| D-005 | Defer tick-data ingestion to a later phase | Trader v0 can use top-of-book + spread + regime. | Microstructure strategies blocked. |
| D-006 | Trader = Qwen3-1.7B with grammar-constrained JSON, not RL agent | Latency 50-150 ms achievable; preserves intuition. | If <200 ms infeasible, switch to Decision Transformer. |
| D-007 | No code in Phase 0 | Patrick asked for spec + state first, "一点一点来 不要做不可逆的事情". | None — pure planning. |

## Cost tracker

| When | What | $ | Cumulative |
|---|---|---|---|
| 2026-04-28 | (nothing) | 0.00 | 0.00 |

Free credit on Modal: ~$13 remaining (after `ttt-conv-memory` consumed ~$17).
Project budget plan: ~$50-100 end-to-end → exceeds remaining free credit by
~$40-90, will need paid spillover or a fresh credit.

## Open questions for Patrick (mirroring SPEC §9)

1. Repo name OK?
2. Analyst v0 size: 3B vs 7B?
3. Trader: LLM-with-grammar vs RL policy?
4. Deployment target: Modal / Fly / landing endpoint?
5. Add tick-data ingestion before trader training, or after?
6. Gemini-flash teacher: primary, or one of several?

## What I'd do next (if Patrick green-lights without amendments)

Phase 1 — feasibility check, expected wall time ~1-2h, cost ~$1:

1. SQL count of rows in: `trade_ideas`, `thesis_state`, `intents`,
   `intentFills`, `portfolio_llm_context`, `portfolio_trades`. Verify
   we have the SFT volume claimed in §10 of SPEC.
2. Pull a 100-row sample from each, dry-run a prompt-formatter that turns
   each row into `(prompt, completion)`. Visual sanity check the output.
3. Modal scratch run: load Qwen3-1.7B + Qwen3-7B with vLLM, measure P50/P95/P99
   latency on a 200-token decode for the trader/analyst prompt shapes. Confirm
   <200 ms / 1-3 s targets achievable.
4. Sketch the numerical adapter: T (lookback steps), F (per-step features),
   K (soft tokens). Decide based on what `market_indicator_history` actually
   stores at 4×/hr × 48h retention = ~192 steps × 9 features.
5. **Stop. Report back to Patrick.** Do not start training without explicit
   green light on the data + adapter shape.

## Pointers

- Spec: `SPEC.md`
- This file: `STATE.md`
- Sister repo (closed): `https://github.com/spfunctions/ttt-conv-memory`
- SF data inventory (full): pending — will be added to `docs/data-inventory.md`
  in Phase 1.
- Memory file: `~/.claude/.../memory/project_sf_decision_stack.md` (created
  alongside this commit)
