# STATE — sf-decision-stack live progress log

Each entry stamped with UTC timestamp. Updated continuously.

## Current phase

**Phase 0 — SPEC v1 frozen (Option A: outcome-only training).**
No code yet. No data downloaded. $0 spent. Awaiting Patrick `/compact` and
green light to start Phase 1.0.

## Phase ledger (v1)

- [x] Phase 0: SPEC v1 + STATE v1 frozen 2026-04-28
- [ ] Phase 1.0: env + R2 access (target $0)
- [ ] Phase 1.5: data extraction + schema verification (target $0.50)
- [ ] Phase 2.0: feature engineering + adapter shape probe (target $1.00)
- [ ] Phase 3.0: Stage 1 SFT analyst (target $1.50)
- [ ] Phase 4.0: Stage 1 SFT trader (target $0.50)
- [ ] Phase 4.5: post-SFT shape eval (target $0.20)
- [ ] Phase 5.0: Stage 2 GRPO analyst (target $30)
- [ ] Phase 5.5: analyst eval (target $1.00)
- [ ] Phase 6.0: Stage 2 GRPO trader (target $20)
- [ ] Phase 6.5: trader eval (target $1.00)
- [ ] Phase 7.0: final report + counterfactual (target $0.50)
- [ ] Phase 8.0: closure (target $0)

**Plan total: $56.20.** **Reserve: $15.** **Budget: $71.20.** **Hard cap: $100.**

Each phase has explicit acceptance criteria in SPEC §13. Each phase MUST
pass before the next starts. Cumulative cost is tracked after each phase.
Abort triggers in SPEC §12.

---

## Detailed log

### 2026-04-28T13:30 PDT — kickoff

Patrick (post-`ttt-conv-memory` failure): build two small low-latency LLMs
over SF data — analyst (alpha) + trader (execution). Common-sense from
pretrained base. Output = decisions. Two model types. New private repo.
Spec + state first, no code yet, nothing irreversible.

### 2026-04-28T13:35 PDT — predecessor closed

`spfunctions/ttt-conv-memory` pushed and abandoned with CLOSURE.md.
Iter-2 verdict: past_w encodes nothing at any α. Patrick's response: "结
果是不是不好...那就放弃吧". $16 total spend, within $30 free credit.

### 2026-04-28T13:38 PDT — repo created

`gh repo create spfunctions/sf-decision-stack --private` → cloned to
`/Users/liuyizhou/spfunctions/sf-decision-stack`.

### 2026-04-28T13:40 PDT — research dispatched (parallel)

- Explore subagent: SF data inventory. Returned 9-section report. Headlines:
  9 indicators 4×/hr, ~9.8k markets at 5-min, top-of-book only orderbook
  (gap), Twitter not stored (gap), R2 daily archives of 33+ tables.
- WebSearch: "small LLM fine-tuning numerical adapter time series 2025-2026"
  → LLM4TS, PEFT (LoRA/QLoRA/DoRA), Unsloth, Sebastian Raschka State of
  LLMs 2025.
- WebSearch: "GRPO RLHF small LLM 1B 3B trading" → Fin-R1, llm-stats
  post-training-2026, hybrid LLM-strategy + RL-execution.
- WebSearch: "LLM trading agent sub-100ms latency" → consensus that pure
  LLM HFT is hard; hybrid is the pattern.
- WebFetch: Fin-R1 paper (arXiv 2503.16252) → confirmed two-stage
  SFT-then-GRPO with binary format + accuracy reward, 60K SFT samples,
  Qwen2.5-7B baseline reaches 75.2 avg on FinBench.

### 2026-04-28T20:55Z — SPEC v0 written

12 sections, 280 lines. Architecture: LoRA + numerical adapter on Qwen3
base, two-stage SFT-then-GRPO. **Initial teacher choice: Gemini-flash
distillation as primary SFT source.** Pushed initial commit to private
remote.

### 2026-04-28T21:00Z — STATE v0 written

Decisions D-001 through D-007 logged. Phase ledger v0. Awaiting Patrick.

### 2026-04-28T21:15Z — Patrick partial sign-off

Tier-1 confirmed (analyst 3B, trader LLM-with-grammar). Other decisions:
"其他都无所谓". Constraints added:
- Private, fully decoupled (no API exposure, no integration with portfolio v2)
- Hard cap $100 Modal, never go over.

### 2026-04-28T21:30Z — Patrick rejects Gemini-flash teacher

> 不对 我觉得你teacher 找错了 我们那个portfolio 就是判断的不咋滴啊

Critical pivot. The portfolio v2 has known quality issues (autopm
position-trapped, evaluations format-only, cumulative P&L −$310, recent
prompt audit findings). Distilling from it = inheriting its bugs.

I proposed three alternatives:
- A: outcome-only training (no teacher distillation; supervision = realized return)
- B: synthetic hindsight teacher (frontier model retrospectively labels)
- C: manual seed (Patrick hand-writes 50-100 examples)

Patrick chose **A** ("ok 那就A"). Recorded as **D-008**.

### 2026-04-28T21:45Z — Patrick demands institutional-grade SPEC

> 你写好事无巨细的SPEC 和执行状态MD 然后我compact 准备开干 要 research and
> institutional grade 的严谨。事无巨细

Pivoted to a full v1 rewrite of SPEC. Length expanded from 280 → 800+ lines.
Added: full I/O schemas, literal prompt templates, mathematical reward
function, exact GRPO hyperparameters, KL clamp/clip values, acceptance
gates per phase, abort triggers, risk register with quantified
probabilities, reproducibility checklist, glossary, references.

### 2026-04-28T22:30Z — SPEC v1 + STATE v1 frozen

This commit. Awaiting Patrick `/compact`.

---

## Decisions log (snapshot)

| ID | Decision | Status | Source |
|---|---|---|---|
| D-001 | Repo name `sf-decision-stack` | Patrick OK | "其他都无所谓" |
| D-002 | Two-stage SFT-then-GRPO | Patrick OK | Fin-R1 recipe |
| D-003 | LoRA + numerical adapter, not pure RL | Patrick OK | preserves common sense |
| D-004 | ~~Distill from Gemini-flash~~ | **REJECTED** | "我们那个portfolio 就是判断的不咋滴啊" |
| D-005 | Tick-data ingestion: defer | Patrick OK | top-of-book is enough for v0 |
| D-006 | Trader = Qwen3-1.7B with grammar | Patrick OK | latency budget achievable |
| D-007 | Analyst = Qwen3-3B | Patrick OK | iteration speed > peak quality at v0 |
| D-008 | **OPTION A: outcome-only training** | Patrick OK | "ok 那就A" |
| D-009 | Hard cost cap $100, abort at $80 | Patrick OK | "不要 go over 100 一定不行" |
| D-010 | Decoupled — no live integration | Patrick OK | "完全去耦合 ... 我们就是测试一下" |
| D-011 | R2 archive as data source | autonomous | cleanest decoupling |
| D-012 | Stage-1 SFT outputs are programmatic stubs | autonomous | preserves D-008's intent |
| D-013 | Reward clip at ±3 spread-units | autonomous | numerical stability |
| D-014 | Strict time-and-canonical-market train/test split | autonomous | no leakage |

---

## Cost tracker

| When | What | $ | Cumulative |
|---|---|---|---|
| 2026-04-28 | Phase 0 (planning) | 0.00 | 0.00 |

Modal free credit: ~$13 remaining (after ttt-conv-memory). Project plan
total $71.20 → ~$58 paid spillover. Hard cap $100; abort trigger $80.

---

## What I'd do next (Phase 1.0, after Patrick `/compact`)

Read order on resume:
1. `SPEC.md` (especially §2, §4, §6, §7, §9, §13, §16).
2. This file (top: phase ledger).
3. `docs/data-inventory.md` (data-table reference).

Then run Phase 1.0:

1. Verify R2 access:
   ```bash
   grep -E "R2_(ACCESS_KEY_ID|SECRET_ACCESS_KEY|BUCKET)" \
     /Users/liuyizhou/spfunctions/simplefunctions-landing/.env.local \
     /Users/liuyizhou/spfunctions/simplefunctions-landing/wrangler.toml \
     2>/dev/null
   ```
2. Write `src/data/r2_client.py` reading one sample (10 rows) from each
   target table. Smoke test locally; no Modal needed.
3. Write `modal_app.py` skeleton. Don't run any GPU function yet.
4. Update STATE.md with row counts, table inventory, observations.

Acceptance gate (SPEC §13 Phase 1.0):
- All target tables readable.
- Row counts >= the lower bounds in SPEC §13 Phase 1.5.
- Modal smoke (CPU-only) builds the image successfully.

If any of those fails, STOP and report to Patrick. Do not proceed to
Phase 1.5 without confirmation.

---

## Pointers

- Spec: `SPEC.md` (v1, frozen).
- Data inventory: `docs/data-inventory.md`.
- Predecessor (closed): https://github.com/spfunctions/ttt-conv-memory
- Memory file: `~/.claude/projects/.../memory/project_sf_decision_stack.md`
  (next-session pickup hint).
- Modal workspace: `patrick-43806` (same as ttt-conv-memory).
- HF cache volume: `ttt-conv-memory-vol` exists on Modal — we should reuse
  it to avoid re-downloading Qwen3 weights ($0 saved + 30 min wall time).
  TODO confirm volume sharing across apps.
