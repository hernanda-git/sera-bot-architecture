# Dynamic Top-5 Pairs Selection System

**Status:** Design / planning. No execution.
**Part of:** sera-bot-architecture (added 2026-07-30)
**Related:** 01-database-architecture.md, 00-master-plan.md

---

## 0. The idea (val's directive)

> I want there are tables to stores all the pairs, and there will be order or priority, then all of bots will scalps top 5 most active pairs, all of this dynamic, all of this realtime.

Instead of every bot trading a static universe of 15 pairs, there is a **dynamic priority table** that:
- stores ALL candidate pairs with a priority score
- ranks them by real-time activity (volume, trade count, spread, volatility)
- each bot (main, wave, alpha) **scraps only the top 5** at any given moment
- the ranking updates continuously — the top 5 is a living set, not a config
- a pair can enter and exit the top 5 dynamically based on market conditions

This directly addresses the crowded-bot problem: if millions of bots all trade the same static 15 pairs, your signal becomes stale. A dynamic top 5 means bots rotate — different pairs get attention at different times, reducing collective adverse selection.

---

## 1. Architecture: how it fits in the fleet

```
  BINANCE (WS + REST)
        │
        ▼
  ┌──────────────────────────────────┐
  │  PAIR PRIORITY ENGINE            │  ← new service (container or daemon)
  │  - polls all USDT pairs every 30s │
  │  - computes activity score         │
  │  - writes pair_priority table      │
  │  - top 5 = dynamic slice           │
  └──────┬──────────┬──────────┬───────┘
         │          │          │
         ▼          ▼          ▼
  ┌─────────┐ ┌─────────┐ ┌────────────┐
  │vaisravana│ │vaisravana│ │vaisravana- │
  │  (main)  │ │  (wave)  │ │   alpha    │
  │reads top5│ │reads top5│ │reads top5  │
  │only trades│ │only trades│ │only trades│
  │from top5 │ │from top5 │ │from top5   │
  └─────────┘ └─────────┘ └────────────┘
```

The Pair Priority Engine is a **shared service**, not a per-bot config. Each bot queries the same table for the current top 5 at the start of each decision cycle. The engine runs independently (its own Docker container or a lightweight Python daemon on the host).

---

## 2. Database schema: `pair_priority` table

To be added to the unified `tradingdb` schema (from `01-database-architecture.md` §4).

```sql
CREATE TABLE pair_priority (
  pair            TEXT PRIMARY KEY,       -- e.g. 'BTCUSDT', '1000BONKUSDT'
  bot_id          TEXT NOT NULL,          -- which bot claimed this pair last
  priority        REAL NOT NULL DEFAULT 0.0,  -- composite score (higher = better)
  rank            INTEGER,                -- computed: 1..N, refreshed each cycle
  in_top5         BOOLEAN NOT NULL DEFAULT FALSE,
  -- activity signals (recomputed each poll cycle)
  volume_1h       REAL NOT NULL DEFAULT 0.0,   -- USDT volume last 1h
  volume_24h      REAL NOT NULL DEFAULT 0.0,   -- USDT volume last 24h
  trade_count_1h  INTEGER NOT NULL DEFAULT 0,
  trade_count_24h INTEGER NOT NULL DEFAULT 0,
  spread_bps      REAL NOT NULL DEFAULT 0.0,   -- current best bid/ask spread in bps
  atr_pct_1h      REAL NOT NULL DEFAULT 0.0,   -- ATR(20) / price * 100, 1h
  volatility_1h   REAL NOT NULL DEFAULT 0.0,   -- std dev of returns, 1h window
  momentum_score  REAL NOT NULL DEFAULT 0.0,   -- EMA direction strength 0..1
  -- metadata
  last_poll_ts_ms INTEGER NOT NULL,
  is_active       BOOLEAN NOT NULL DEFAULT TRUE,  -- exclude delisted/paused pairs
  created_at      INTEGER NOT NULL,
  updated_at      INTEGER NOT NULL
);

CREATE INDEX idx_pair_priority_rank    ON pair_priority(rank);
CREATE INDEX idx_pair_priority_top5    ON pair_priority(in_top5, priority DESC);
CREATE INDEX idx_pair_priority_bot     ON pair_priority(bot_id, priority DESC);
```

### The `priority` score (composite, tunable)

```
priority = w1 * normalized_volume_24h
         + w2 * normalized_trade_count_1h
         + w3 * (1.0 / (spread_bps + 1))     -- tighter spread = higher score
         + w4 * atr_pct_1h                    -- more volatility = more opportunity
         + w5 * momentum_score               -- trending pairs get priority
         + w6 * (1.0 / (1 + current_open_interest_pct)) -- fewer open positions = more room
```

Default weights (all tunable via ParameterSurface — additive gate):
| Weight | Default | Meaning |
|---|---|---|
| w1 | 0.30 | 24h volume (liquidity proxy) |
| w2 | 0.25 | 1h trade count (activity) |
| w3 | 0.15 | Spread tightness (cost of entry) |
| w4 | 0.15 | ATR volatility (move budget) |
| w5 | 0.10 | Momentum strength (trend quality) |
| w6 | 0.05 | Position capacity (fewer concurrent positions = more room) |

All weights must sum to 1.0 (enforced by ParameterSurface constraint).

---

## 3. How each bot uses the top 5

### Main bot (vaisravana)
- At the start of each decision cycle (`run()` loop), queries the `pair_priority` table for `rank <= 5 AND in_top5 = TRUE`
- If the current pair is NOT in top 5, the bot skips that pair for this cycle (graceful exit, no wasted computation)
- If top 5 changes (a pair drops out, a new one enters), the bot fades out its position in the dropped pair over the next `MAX_HOLD` window and does not open new positions in it until it re-enters top 5

### Wave bot (vaisravana-wave)
- Same mechanism: reads top 5 at each wave cycle (30-min cron tick)
- The wave engine already has a `pairs` config list; the top 5 dynamically overrides it
- If the wave engine is mid-position in a pair that drops from top 5, it completes the wave (no forced close), but no new waves open for that pair

### Alpha bot (vaisravana-alpha)
- The alpha's `ExitEngine` already sees every tick for all 15 pairs (exit engine runs across all pairs). The top 5 filter applies only to the **entry/decision** path (`AlphaEngine.on_tick`), not to the exit path (which must always be live for risk management)
- Exit engine continues to monitor all pairs for stop-loss / take-profit regardless of top 5 status

---

## 4. The Pair Priority Engine — service spec

**Container:** `bots-pair-priority` (new container in the `bots` compose stack)

### Input
| Source | Endpoint | Cadence |
|---|---|---|
| Binance REST `/fapi/v1/ticker/24hr` | All USDT perpetual symbols | Every 30s |
| Binance REST `/fapi/v1/ticker/bookTicker` | Best bid/ask for top 50 pairs | Every 30s |
| Binance WS `aggTrade` | Trade counts per pair (for volume) | Continuous |
| Binance WS `bookTicker` | Spread real-time | Continuous |

### Processing
1. Every 30s: fetch 24hr stats for all USDT perpetuals
2. Compute `volume_24h`, `trade_count_24h` from REST
3. Compute `volume_1h`, `trade_count_1h` from the last 60 data points of WS aggTrade
4. Compute `spread_bps` from `bookTicker` (ask - bestBid) / bestAsk * 10000
5. Compute `atr_pct_1h` from the last 60 1m klines stored in a rolling window
6. Compute `momentum_score` from the EMA slope of the pair's 1h close (sign * magnitude)
7. Apply the priority formula, rank 1..N, set `in_top5` for ranks 1-5
8. UPSERT into `pair_priority` table (PostgreSQL, or the pair_priority DB)
9. Emit a `/pair_rank_update` Telegram message to val's chat showing the new top 5 (compact)

### Output table (where each bot reads)
Each bot queries: `SELECT pair, priority FROM pair_priority WHERE in_top5 = TRUE ORDER BY rank LIMIT 5`

### Failure modes
- **Engine down:** bots fall back to their static pair list (no change from current behavior). The engine's restart is self-healing.
- **Partial data:** if <20 pairs have valid data, the top 5 uses whatever has scores; if 0 pairs, all bots halt trading (similar to the stop-flag behavior).
- **Binance rate limit:** the 24hr endpoint allows 1200 weight/min per IP; polling 200 pairs every 30s = ~500 weight/cycle — well within limits.

---

## 5. The top 5 rotation — how it addresses the crowded-bot problem

### Why rotation helps
If every retail bot runs EMA crossovers on a static 15-pair list, the collective entries cluster on the same pairs at the same time. The EMA cross triggers for BTC → 50k bots pile in → adverse selection. The signal has already been consumed by the time your order hits the book.

With dynamic top 5 rotation:
- Pair A might be top 5 at 10:00 (high volume, strong momentum) → bots concentrate there
- At 10:30, pair A's ATR compresses (move exhausted), pair D's volume spikes (news event) → pair D enters top 5
- Now bots diverge: some are in A (riding), some are entering D (fresh signal)
- **Less collective clustering = less adverse selection per bot**

### The mathematical argument
Adverse selection is proportional to the *density* of bots on a given signal at a given moment. If N bots all trade the same signal simultaneously, each bot's expected edge = `edge₀ / N` (rough linear model). If the top 5 rotates such that only `M` bots chase a given signal at once (where M < N because some bots are in other pairs), then each bot's effective edge is `edge₀ / M > edge₀ / N`.

This doesn't create alpha — it **preserves** the alpha that otherwise gets destroyed by crowding. The highest-ROI move is pairing signal quality with reduced crowding.

---

## 6. Priority table lifecycle

```
  ┌────────────┐     30s poll     ┌──────────────┐
  │  Binance    │ ──────────────► │  Engine       │
  │  REST + WS  │                 │  computes     │
  └────────────┘                 │  priority     │
                                 │  scores       │
                                 │               │
                                 │  UPSERT to    │
                                 │  pair_priority│
                                 └───────┬───────┘
                                         │
                              SELECT top 5
                                         │
                    ┌────────────────────┼────────────────────┐
                    │                    │                    │
              ┌─────▼─────┐    ┌────────▼─────┐    ┌───────▼───────┐
              │  main bot  │    │  wave bot    │    │  alpha bot    │
              │  reads top5│    │  reads top5  │    │  reads top5   │
              │  skips     │    │  skips       │    │  skips        │
              │  non-top5  │    │  non-top5    │    │  (entries;    │
              │  trades    │    │  trades      │    │  exits all)   │
              └─────────────┘    └──────────────┘    └───────────────┘
```

### Transition rules (smooth handoffs)
When a pair drops from top 5:
1. The bot marks the pair as `top5_status = 'exiting'`
2. In-flight positions are held until natural close (TP/SL/MAXHOLD)
3. No new entries are opened for that pair until it re-enters top 5
4. After `TOP5_EXIT_GRACE_S` (default 300s, 5 min) of being outside top 5, the pair is fully excluded from decision-making (reduces churn from rapid rotations)

When a pair enters top 5:
1. Pair moves to `top5_status = 'entering'`
2. Bot warms up indicators for that pair (seed EMA from last 60 1m candles from the DB)
3. After `TOP5_ENTRY_WARMUP_S` (default 120s), the pair is fully eligible for entries
4. Warmup prevents the bot from chasing a pair that just spiked into top 5 on a one-off volume spike

---

## 7. DB placement: pair_priority table

The `pair_priority` table belongs to the **PostgreSQL fleet aggregate** (Tier 2 in 01-database-architecture.md), not to per-bot SQLite, because:
- It is **shared state** — all bots read the same top 5
- It is **write-heavy on one process** (the engine, not the bots) — no contention
- It is **read-heavy on many processes** (each bot queries it every cycle) — PG's MVCC handles this gracefully
- It benefits from PG's `BRIN` index on `rank` for fast top-5 queries

The engine writes; each bot reads. The engine's write is the single source of truth.

---

## 8. Parameters (all tunable via ParameterSurface)

| Parameter | Default | Range | Env var |
|---|---|---|---|
| POLL_INTERVAL_S | 30 | 10..300 | `PAIR_PRIORITY_POLL_S` |
| TOP_N | 5 | 3..15 | `PAIR_PRIORITY_TOP_N` |
| TOP5_EXIT_GRACE_S | 300 | 60..900 | `TOP5_EXIT_GRACE_S` |
| TOP5_ENTRY_WARMUP_S | 120 | 0..300 | `TOP5_ENTRY_WARMUP_S` |
| w1 (volume) | 0.30 | 0.0..1.0 | `PAIR_PRIORITY_W1` |
| w2 (trade count) | 0.25 | 0.0..1.0 | `PAIR_PRIORITY_W2` |
| w3 (spread) | 0.15 | 0.0..1.0 | `PAIR_PRIORITY_W3` |
| w4 (volatility) | 0.15 | 0.0..1.0 | `PAIR_PRIORITY_W4` |
| w5 (momentum) | 0.10 | 0.0..1.0 | `PAIR_PRIORITY_W5` |
| w6 (capacity) | 0.05 | 0.0..1.0 | `PAIR_PRIORITY_W6` |
| MIN_PAIRS_FOR_TOP5 | 5 | 2..50 | `PAIR_PRIORITY_MIN_PAIRS` |

All weight constraints (sum = 1.0) are enforced by ParameterSurface validation. Individual weight edits are additive.

---

## 9. Relation to existing systems

### To the swarm (03-agentic-architecture.md)
The RESEARCHER agent (03 §1.4) can propose new signal sources for the priority formula. For example: "order-flow imbalance is a better activity signal than volume" → RESEARCHER proposes → SKEPTIC validates against historical data → if ACCEPTED, the weight shifts or a new signal is added to the formula as an additive gate. No engine change.

### To the dynamic top 5 and the wave bot's existing pair list
The wave bot currently trades 15 pairs hardcoded. The top 5 system **replaces** that static list with the dynamic slice. The wave engine's `pairs` config stays as a fallback (min 10 pairs in the config ensures the system has candidates to rank).

### To the unified DB schema (01-database-architecture.md)
The `pair_priority` table is added to the schema in 01 §4.2 (position) area. It is a dimension table, not a fact table — it does not join directly to positions or fills (it references pairs by symbol, not by position_id). It is consumed by the engine, not by the evaluator.

---

## 10. Open questions for val

1. **Minimum active pairs:** How many pairs must have valid data before the system starts ranking? Default is 5 (`MIN_PAIRS_FOR_TOP5`), but if you only want ranking when 20+ pairs are active, that's tunable.
2. **Grace periods:** The 300s exit grace and 120s entry warmup defaults are reasonable starting points. Adjust based on how fast you want bots to rotate.
3. **Telegram alert on top 5 change:** the engine emits a `/pair_rank_update` when top 5 changes. Want this in the no-emoji card format (02)? This is a display-only change, no engine impact.
4. **Should each bot have its OWN top 5 or a SHARED top 5?** A shared top 5 means less total crowding across the fleet. Individual top 5s (weighted per-bot by that bot's strategy) mean more targeting. I recommend shared for now; can evolve to per-bot weighting in the priority formula's w6 (capacity) term.
