# sf-decision-stack

Two small low-latency LLMs over SF (simplefunctions) data, decoupled from
the live system (read-only data access, no integration, paper-only):

- **Analyst** — produces alpha (Qwen3-3B + LoRA + numerical adapter, target p95 ≤ 3 s).
- **Trader** — executes alpha (Qwen3-1.7B + grammar-constrained JSON, target p95 ≤ 200 ms).

Trained via **outcome-only supervision** — no opinion distillation. The only
training signal is the realized future return computed from price history.
Two stages: format-only SFT (~$2) then GRPO (~$30-60) per Fin-R1 recipe.

**Hard cost cap: $100 Modal**, abort trigger at $80.

**Status:** SPEC v1 frozen 2026-04-28. Awaiting Patrick `/compact` and Phase 1.0 start.

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
