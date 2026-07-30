# Implementation Roadmap — Phased Execution

**Status:** DESIGN. Execution is gated on val's explicit go-ahead ("the goal is to plan, no execution").
**Part of:** sera-bot-architecture
**Sentinel note:** each phase lists which changes are ParameterSurface/additive (auto) vs engine (needs approval).

---

## Phase 0 — Unified schema package `tradingdb` (foundation)
**Type:** additive infrastructure. No strategy change.
1. Create `tradingdb` Python package with the DDL from 01 §4 (SQLite dialect) + a `Base` model + connection helper applying the §7.1 PRAGMAs (WAL, busy_timeout=5000, synchronous=NORMAL, foreign_keys=ON).
2. All four bots import it; each still writes its own file. **This alone fixes the fee / 0.0 / footer bugs** because the write logic lives in one place.
3. Add the `fee_ledger`, `marker`, `equity_snapshot`, `outbox` tables.
**Verify:** a throwaway script writes a position with NULL entry, fills it, asserts `entry_price > 0`, asserts two `fee_ledger` rows (open+close), asserts `equity_snapshot` footer renders.

## Phase 1 — Backfill + validation
1. `PRAGMA user_version` migration scripts add new tables to existing DBs.
2. Backfill `equity_snapshot` from historical `position` aggregates.
3. Alert if any `entry_price = 0` (the bug val saw) — these rows get NULL-render fixes.
**Verify:** every bot's DB passes `PRAGMA integrity_check`; zero `entry_price = 0` rows after migration.

## Phase 2 — Notification format (D3)
**Type:** display-only. Sentinel-compliant.
1. Port `render_open` / `render_close` (02 §5) into both bots' notifiers.
2. Main bot: replace `notify_fill` / `notify_close` bodies.
3. Wave bot: replace `notify_wave_open` / `notify_wave_close` (currently emoji); supply the aggregate `stats` dict from `wave_log` summary.
4. Extend `test_notify_fill_and_close_shape` to assert `<pre>` wrapper + `▓` bar + no emoji.
**Verify:** render both cards with a stub client; assert `not _EMOJI.search(text)` AND `"LONG" in text` AND `"WIN" in text`.

## Phase 3 — Fee constant fix (D5)
**Type:** ParameterSurface-tunable. Non-destructive.
1. `FEE_BPS_ROUNDTRIP` 6.0 → 7.0 (6.3 with BNB).
2. Verify `CLOSE_FEE_RATE` against live Binance VIP0 taker (0.0005, not 0.0004).
3. The `fee_ledger` from Phase 0 now records the *actual* per-fill fee, so the EV gate and notifications are truthful.
**Verify:** a trade's `net_pnl` = realized − fee_open − fee_close matches the exchange statement within rounding.

## Phase 4 — Agentic coordinator (D4)
**Type:** infrastructure + loop logic. No engine change.
1. Promote the wave bot's 30-min loop to the coordinator + SKEPTIC / PRACTITIONER / ARCHITECT triple (`delegate_task`, background).
2. Wire the 6-layer eval (AUDITOR) as the promotion gate.
3. Add `knowledge-base/transfer.md` for cross-bot learning.
**Verify:** one full iteration: hypothesis → 3-agent vote → patch → eval → KEEP/REVERT committed with evidence.

## Phase 5 — Dynamic top-5 pairs system
**Type:** infrastructure + new service container. No engine change (just a new service the bots query).
1. Deploy `bots-pair-priority` container (new service, see 07-top5-pairs-system.md).
2. The engine polls Binance REST + WS for all USDT perpetuals; computes composite priority score; writes `pair_priority` table.
3. Each bot reads `SELECT pair FROM pair_priority WHERE in_top5=TRUE ORDER BY rank LIMIT 5` at every decision cycle.
4. Non-top-5 pairs are skipped for entries; exits continue for all pairs (risk management unchanged).
5. Transition rules: 300s exit grace, 120s entry warmup (both tunable).
**Verify:** top 5 table updates every 30s; bots gracefully rotate pairs without forced closes; Telegram alerts on top 5 change.

## Phase 6 — Cross-bot learning (D9)
**Type:** infrastructure + knowledge base. No strategy change.
1. Enable `knowledge-base/transfer.md` propagation in the swarm loop (03 §2.2).
2. When alpha discovers a high-ROI additive gate (e.g. fee-aware EV), the coordinator writes it to `transfer.md`.
3. main and wave bots' next ticks read that file and can adopt the gate as an additive change without re-discovering it.
**Verify:** cross-bot transfer documented in LOOP_STATUS.md; no deployment errors from the KB propagation.

## Phase 7 — Fleet aggregate (PostgreSQL, D7 Option C)
**Type:** infrastructure. Optional until Phase 0 is stable.
1. Deploy `trading_fleet` PG container (partitioned by day, rollups, retention).
2. Each bot appends events to `outbox`; deploy the `collector` container that drains to PG.
3. Cross-bot Telegram digests, fleet PnL, evaluator reads from PG.
**Rollback:** stop the collector → bots are fully independent again (Tier 1 unchanged).

---

## Sequencing rationale
Phases 0–3 are the "make it correct + truthful" track: unified DB, fixed notifications, correct fees. Phase 4 makes the fleet learn. Phase 5 adds dynamic pair rotation. Phase 6 enables cross-bot knowledge sharing. Phase 7 is the nice-to-have fleet unification.

**Nothing in Phases 0–6 touches the engine or StrategyProfile** — they are all Sentinel-compliant (ParameterSurface / additive / display / infrastructure). Phase 5 is the only new service container, not an engine change. Phase 7 is optional infrastructure.

---

## Exit criteria for "100% good" (val's bar)
- [ ] Every trade alert is self-contained: Balance / Used / Unrealized / Realized / win rate / W:L / cumulative fees, with `—` for pending.
- [ ] Fees recorded on BOTH open and close, maker/taker split visible.
- [ ] Entry/SL/TP never render `0.0` — only real prices or `—`.
- [ ] `/stop` actually halts and survives restart (DONE for main; wave/alpha already OK).
- [ ] One unified schema across all four bots; fleet query possible.
- [ ] Swarm loop runs with adversarial check (SKEPTIC) + 6-layer eval gate.
- [ ] EV gate uses the correct 7.0 bps fee constant.
- [ ] At least CVD-entry + OI + funding gates live (additive) — the documented edge ships.
- [ ] Dynamic top 5 pairs system live: bots scrape only the top 5 most active pairs, real-time rotation, no crowding.
- [ ] Cross-bot learning enabled: alpha's discoveries propagate to main/wave automatically.
