# Sera Bot Architecture — Master Plan (PLANNING ONLY)

**Status:** Planning / Design phase. No code execution, no deployment.
**Date:** 2026-07-30 (updated)
**Author:** Kira (valarion's assistant) + multi-agent research sprint
**Scope:** Database design, notification format, agentic architecture, honest evaluation, dynamic top-5 pairs system for all bots on sera (43.157.208.115).

---

## What this repository contains

```
sera-bot-architecture/
  README.md                         # this file
  docs/
    00-master-plan.md               # executive summary + decisions + open questions
    01-database-architecture.md     # unified DB design (SQLite + PostgreSQL hybrid)
    02-notification-format.md       # trade card templates (open/close, no emoji)
    03-agentic-architecture.md      # multi-agent swarm design
    04-honest-evaluation.md         # expert review of current bot stack
    05-fee-model.md                 # Binance maker/taker analysis, small-account math
    06-implementation-roadmap.md    # phased execution plan (waits for val's go)
    07-top5-pairs-system.md         # dynamic top-5 pairs selection by activity
  diagrams/
    architecture-overview.txt       # ASCII fleet diagram
    data-flow.txt                   # tick-to-notification data flow
  src/
    trade_notif_render.py           # verified no-emoji renderer (run and test)
```

## Bots covered

| Bot | Repo | Role | DB today |
|---|---|---|---|
| vaisravana (main) | hernanda-git/vaisravana | directional taker, paper | SQLite `vaisravana.db` |
| vaisravana-wave | hernanda-git/vaisravana-wave | wave/EMA momentum, paper | SQLite `vaisravana-wave.db` |
| vaisravana-alpha | hernanda-git/vaisravana-alpha | clean redesign + real-time exit | SQLite + agentic DB |
| listener (fatty) | hernanda-git/learnernoearner-listener | Telegram/LLM feed listener | SQLite `trades.db` |

## Decisions (D1–D10)

| ID | Decision | Status |
|---|---|---|
| D1 | Database: hybrid (per-bot SQLite WAL hot + PG fleet-aggregate via outbox) | ✅ Confirmed |
| D2 | Unified schema package `tradingdb` for all bots | ✅ Designed |
| D3 | Notification: monospace `<pre>` card, no emoji, LONG/SHORT, WIN/LOSS, `—` for pending | ✅ Verified (renderer ran clean) |
| D4 | Agentic swarm: 5 roles (SKEPTIC / PRACTITIONER / ARCHITECT / RESEARCHER / AUDITOR) + coordinator | ✅ Designed |
| D5 | Fee constant fix: 6.0→7.0 bps RT (Binance VIP0 taker is 0.05%, not 0.04%) | ✅ Designed |
| D6 | Sentinel boundary preserved | ✅ Enforced |
| D7 | PostgreSQL fleet aggregate: YES (val confirmed) | ✅ Confirmed |
| D8 | Funding-carry satellite: NO (val declined) | ✅ Declined |
| D9 | Cross-bot learning: YES (enable `transfer.md` in swarm KB) | ✅ Confirmed |
| D10 | Dynamic top-5 pairs system: shared pair priority table, real-time ranking, all bots scrape only top 5 | ✅ Designed |

## Problems being solved

1. **Stop command doesn't work.** FIXED (v0.0.37, pushed). `/stop` writes persistent `vaisravana_stop.flag`; bot refuses to trade at boot while flag exists; `/clean` removes it.
2. **Fee not calculated on open AND close.** Fixed by `fee_ledger` table capturing both legs with maker/taker split.
3. **Entry/SL/TP shows 0.0 when pending.** Fixed by nullable prices + `—` placeholder in notifications.
4. **Balance/Used/Unrealized/Realized missing from cards.** Fixed by `equity_snapshot` table + always-present footer.
5. **Heterogeneous DBs.** Unified schema + `tradingdb` package.
6. **Crowded-bot problem.** Addressed by dynamic top-5 rotation (07) reducing collective clustering.
7. **Real-time scraping gaps.** Identified in 04: OI, funding, depth, CVD-entry, BTC lead-lag, volatility gates all missing.

## Standing constraints

- All deployments LOCAL to sera only. No Fly.io. docker-compose `bots` stack.
- Sentinel: only ParameterSurface + additive gates change without val's explicit approval. Engine / StrategyProfile changes need val's go.
- Notification style: clean, professional, NO emoji. LONG/SHORT, WIN/LOSS. `—` for pending values (NOT zero). No em dashes as separators.
- Git identity: valarion / 42990222+hernanda-git@users.noreply.github.com globally and per-repo.
- Wave bot repo separate from main bot repo (already the case).
- All bots stopped as of this session. No execution without explicit go-ahead.