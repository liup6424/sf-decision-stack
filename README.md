# sf-decision-stack

Two small low-latency LLMs over SF (simplefunctions) data:

- **Analyst** — produces alpha (trade ideas).
- **Trader** — executes alpha (orders).

Both fine-tuned (LoRA + numerical adapter) from a Qwen3 pretrained base via
a two-stage SFT-then-GRPO recipe, distilled from SF's existing Gemini 2.5
Flash production decision pipeline plus the `trade_ideas` and `thesis_state`
ground truth.

**Status:** SPEC v0 written 2026-04-28 (autonomous research session).
No code. Awaiting Patrick review before Phase 1.

## Read order for newcomers

1. [`SPEC.md`](SPEC.md) — what we're building, why, and how.
2. [`STATE.md`](STATE.md) — live progress, decisions, cost tracker.
3. [`docs/data-inventory.md`](docs/data-inventory.md) — what SF data exists today and where.

## Layout (planned)

```
sf-decision-stack/
├── README.md              # this file
├── SPEC.md                # detailed design + phase ledger
├── STATE.md               # chronological progress log
├── DECISIONS.md           # research-level forks (will grow with the project)
├── docs/
│   ├── data-inventory.md  # full SF data inventory
│   └── adapter-design.md  # (Phase 1) numerical adapter shape & rationale
├── data/                  # (Phase 2) cached parquet of SFT pairs
├── src/                   # (Phase 3+) training and inference code
└── modal_app.py           # (Phase 3+) Modal entry points
```

Nothing under `src/` or `modal_app.py` yet — explicitly Phase 0 only.
