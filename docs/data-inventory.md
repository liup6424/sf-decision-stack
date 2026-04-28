# SF Data Inventory — for sf-decision-stack training

**Verified:** 2026-04-28 against the live `simplefunctions-landing` codebase
and memory notes (2026-04-14 → 2026-04-27 timeframe).

Headlines:
- Analyst data readiness: **8/10** (rich indicators + trade ideas + thesis evals).
- Trader data readiness: **6/10** (top-of-book orderbook only; no depth history,
  no tick-level trade tape).

---

## 1. Market data tables (Drizzle)

| Table | Granularity | Retention | Status | Useful for |
|---|---|---|---|---|
| `market_price_snapshots` | 15-min snapshots | ~7d (pruned nightly) | LIVE | Analyst (history) |
| `market_changes` | event-triggered (>5c deltas) | ~3d | LIVE | Analyst (catalyst detection) |
| `orderbookSnapshots` | sparse on-demand | ~48h | LIMITED | Trader v0 (top-of-book + spread + depth-ratio only) |
| `market_regime_snapshots` | 6h windows | ~30d | LIVE | Trader (regime + liquidity state) |
| `market_indicators` | 4×/hour (15-min) | current + 48h history | LIVE | Analyst & trader (9 indicators: IY, IY-yes/no, CRI, EE, LAS, CVR, RV, VR, IAR) |
| `market_indicator_history` | 15-min snapshots | 48h rolling | LIVE | Analyst (short-term signals) |
| `canonical_markets` + `canonical_market_venues` | static | indefinite | LIVE | Cross-venue ID mapping |
| `external_markets` | ~5 min (Kalshi & Poly) | ~7d | LIVE | Raw venue snapshots |

**Critical gap:** No tick-level trades. Top-of-book only. Blocks microstructure
learning for trader.

## 2. Orderbook & quote data

| Source | Depth | Format | Freshness | Stored? |
|---|---|---|---|---|
| Kalshi | top-of-book | REST `/orderbook/:ticker` via `kalshi-proxy` Cloudflare worker | on-demand | snapshot-only into `orderbookSnapshots` |
| Polymarket | top-200 by volume | WebSocket in `sf-terminal` Go binary | real-time | **NOT persisted** — lost after the live session |
| Cross-venue arb | top-of-book gap | `crossVenuePairs` daily rebuild | daily 04:00 UTC | LIVE (gap snapshots) |

## 3. Indices, betas, correlations

| Table | Cadence | Use |
|---|---|---|
| `sfIndexSnapshots` | ~6h + on-demand | Analyst (macro regime) |
| `market_indicators` (IY/CRI/CVR/EE/LAS/RV/VR/IAR) | 4×/hour | Both |
| `marketGroupBetas` | weekly (Sun) | Analyst (cross-market correlation) |
| `seriesTermFit` | partial (rebuild cadence unclear) | Analyst (term structure) |

`INDICATOR_TABLE_ENABLED=true` in prod.

## 4. News / text data

| Source | Storage | Frequency | Volume | Status |
|---|---|---|---|---|
| Congress.gov + OpenStates bills | `events`, `government_news`, `legislation_market_pairs` | weekly cron | ~1k bills/week | LIVE |
| News scraping (Firecrawl `monitor-news` cron) | `events` | background (cadence unclear) | TBD | PARTIAL |
| Government research summaries | `governmentResearchSnapshots`, `government_entities`, `electedOfficials` | on-demand | ~300 entities | LIVE |
| Market descriptions | `external_markets.title/outcomes` | 5-min | ~9.8k markets | LIVE |
| Blog / opinions / technicals / concept pages | `blog_posts`, `opinions`, `technicals`, `concept_pages` | manual + agent | ~500 published | LIVE |
| Trade ideas | `trade_ideas` (daily cron output) | daily 18:00 UTC | ~30/day | LIVE |
| Twitter/X | (queried via xdk, not stored) | on-demand | n/a | NOT STORED |
| Research notes | `research_notes` | daily | ~100 | LIVE |
| Weekly summaries | `weekly_summaries` | Mon | ~52/yr | LIVE |

**Gaps:** Twitter sentiment lost; no macro-economic calendar; no RSS; no arXiv.

## 5. User intent + portfolio state (from `project_portfolio_automation.md`)

Portfolio v2 deployed 2026-04-14, 1M-context Gemini 2.5 Flash + Claude Sonnet
escalation. Multi-tenant via Trigger.dev.

| Data | Tables | Freshness |
|---|---|---|
| Portfolio config (per user: enabled, mode dry/live/halted) | `portfolio_config` | per edit |
| Portfolio state (cash, exposure, P&L, HWM, drawdown) | `portfolio_state` | ~5 min tick |
| Thesis state (confidence, causal tree, edge analysis, realized returns) | `thesis_state`, `thesis_evaluations` | 4×/hour |
| Trades (with PnL outcome) | `portfolio_trades` | real-time on fill |
| Open positions / orders | `intents`, `intentFills` | real-time |
| Trade ideas | `trade_ideas` | daily |
| Rolling LLM context (compacted at 300k → 80k via Gemini Flash) | `portfolio_llm_context` | per-tick |

## 6. R2 archives (per `project_r2_dump_architecture.md`, deployed 2026-04-18)

| Cron | Tables | Daily volume | Layout |
|---|---|---|---|
| `data-dump-compute` | 7 compute tables | 876k rows / 247 MB JSONL | `{table}/{date}.jsonl` |
| `data-dump-markets` | 10 market tables incl. `market_price_snapshots` | 272k rows + 4.68 GB JSONB per-row | `{table}/{date}.jsonl` + `{table}/{date}/{ts}_{shortId}.json` |
| `data-dump-thesis` | 11 thesis tables | 7k rows / 11 MB | JSONL |
| `data-dump-obs` | 11 observation tables | 63k rows / 30 MB | JSONL |

R2 retention: indefinite (free tier). Skipped: `cli_login_sessions`,
`vector_indexes`, `audit_log_entries`, `indicator_read_log`, Supabase auth.

## 7. API endpoints worth knowing about (existing plumbing)

Public:
- `/api/public/context` — global intelligence snapshot (15-min)
- `/api/public/markets`, `/api/public/market/{ticker}`, `/api/public/scan`,
  `/api/public/screen`, `/api/public/index`, `/api/public/regime/*`,
  `/api/public/contagion`, `/api/public/briefing`, `/api/public/ideas`,
  `/api/public/legislation`
- `/api/agent/world` and `/api/agent/world/delta` — compressed state for LLM
- `/api/agent/world/feed` — markdown change stream
- `/api/agent/inspect/{ticker}` — per-ticker analysis with suggested action

Internal: thesis CRUD, research generation, intent system, portfolio mgmt.

## 8. Per-model readiness summary

### Analyst (research / alpha generation) — 8/10 readiness

Strengths:
- Rich indicator coverage at 4×/hour (the 9-indicator suite).
- Catalysts: bills, news, events.
- Trade idea generation pipeline (signals to learn from): `trade_ideas`.
- Thesis evaluation ground truth (confidence drift vs realized return).
- Narrative feeds: briefing, world-state, weekly summaries.
- Production teacher present: Gemini 2.5 Flash logs in `portfolio_llm_context`.

Weaknesses:
- No tick-level event history.
- Twitter queried but not stored (no historical sentiment).
- News ingestion cadence unclear.
- No macro calendar.

### Trader (execution) — 6/10 readiness

Strengths:
- `crossVenuePairs` for arb signals.
- `orderbookSnapshots` for top-of-book + spread + depth-ratio.
- `market_regime_snapshots` for liquidity context.
- `intents` + `intentFills` for the action log of past trader behavior.
- `portfolio_state` provides risk gates.

Weaknesses (critical):
- **No orderbook depth history beyond top-of-book.** Cannot study
  liquidity provision, order flow imbalance, or depth dynamics.
- **No tick-level trade tape.** Cannot study execution slippage in detail.
- Polymarket execution not implemented (read-only).

## 9. Recommendation

- **Conditional GO.** Analyst can train successfully on existing data.
  Trader is viable for swing-style execution + arb detection but will
  underperform a system with tick + depth history.
- Before trader training, decide: ship v0 with top-of-book only, or budget
  3-4 weeks to add tick persistence (`tick_trades`, `tick_ob_changes`) and
  Polymarket execution.

## 10. Memory pointers used to compile this report

- `project_full_stack_audit.md` (2026-04-14)
- `project_portfolio_automation.md` (2026-04-14)
- `project_r2_dump_architecture.md` (2026-04-18)
- `project_thesis_git_phase5_canary.md` (2026-04-17)
- `project_audit_backlog.md`
- `project_external_metrics_2026-04-08.md`
- `project_sf_what_it_actually_is.md`
- `feedback_silent_failures.md` (some tables may be silently broken)
