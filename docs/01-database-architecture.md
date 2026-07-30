# Trading Bot Fleet — Advanced Database Architecture (sera)

**Author:** Hermes (research + design subagent)
**Date:** 2026-07-30
**Scope:** Design document only — no execution. Research-backed, production-grade.
**Target host:** `sera` (43.157.208.115) — single VPS, docker-compose, no Fly.io.
**Fleet:** `vaisravana` (paper, SQLite `/opt/bots/vaisravana/data/vaisravana.db`), `vaisravana-wave` (wave engine, SQLite `/opt/bots/vaisravana-wave/data/vaisravana-wave.db`), `vaisravana-alpha` (layered package + agentic DB `/root/vaisravana-alpha`), and a `listener`/`fatty` bot.

---

## 0. Executive Summary

You have four bots, each already owning its own SQLite file. The current pain is **not** primarily concurrency — at paper-trading volumes (a few dozen writes/sec peak per bot) SQLite WAL handles that trivially. The real problems are **(a)** an inconsistent, under-specified schema across the four bots and **(b)** correctness bugs in how fees and entry/SL/TP markers are written. The "homogeneous but divergent" state means you cannot query the fleet as one.

**Recommendation — a two-tier hybrid with a single shared schema:**

1. **Tier 1 (hot / system-of-record per bot):** Each bot keeps its own SQLite file in WAL mode, but **all four adopt one standardized schema** defined in a shared Python package (`tradingdb`). Per-bot file ownership means *zero cross-process write contention by construction* — the single-writer lock never arbitrates between bots.
2. **Tier 2 (cold / fleet-aggregate):** A single PostgreSQL container on `sera` (or optionally a fifth aggregated SQLite) ingests every bot's events via an **outbox → collector** pattern, giving you one place for cross-bot PnL, fleet alerts, evaluator input (DSR/CSCV/PBO per the loop skill), and archival/retention.

This delivers "unified **and** standardized" without forcing the hot path through a network round-trip or a single-writer bottleneck. The schema below is written to run **unchanged on both SQLite and PostgreSQL** (types are compatible; only the DDL dialect and partition clauses differ).

### Fixes delivered by this design
| Symptom | Root cause (typical) | Fix in this design |
|---|---|---|
| Fees not calculated on open/close | Reads `fill['fee']['cost']` without maker/taker split; fee only captured on close | `fee_ledger` table: one row per fee event with `side ∈ {open,close}`, `fee_type ∈ {maker,taker}`, `rate`, `cost`, `currency`. Trade carries `fee_open_total`, `fee_close_total`. |
| Entry/SL/TP shows `0.0` | Marker row inserted at signal time with default `0.0`; price only known after fill | Markers are `NULLABLE` until the fill arrives; `CHECK (price > 0 OR price IS NULL)`; entry price is sourced from the **fill**, SL/TP are derived from entry + configured pct at open. |
| Balance/Used/Unrealized/Realized missing from notifications | No per-cycle equity snapshot persisted; notification reads ephemeral wallet state | `equity_snapshot` table written every cycle (or on every position change) per bot; notifications read `latest_snapshot(bot_id)`. |

---

## 1. Architecture Options & Trade-offs

### Option A — Separate SQLite per bot (current)
Each bot owns `/opt/bots/<bot>/data/<bot>.db`. No shared file → no cross-bot lock contention.
- ✅ Zero operational overhead, microsecond-latency writes, trivial backup (file copy), easy offline inspection.
- ✅ Matches the "each writer owns its file" best practice exactly.
- ❌ No unified fleet view; four divergent schemas; duplication; can't join PnL across bots; evaluator/alerting must read 4 files.

### Option B — Single unified PostgreSQL (one SOR)
One PG container; every bot connects with a `bot_id` tenant column; MVCC gives true concurrent writers.
- ✅ Real concurrency (no single-writer lock), rich types, FKs, JSONB, window functions, replication, point-in-time recovery, one fleet query surface.
- ✅ Time-series partitioning, retention, rollups are first-class.
- ❌ Operational overhead: run + monitor PG in Docker, connection pooling, schema migrations, backup/restore discipline. A network/config outage can stall all bots if PG is the SOR for hot writes.
- ❌ Per-write latency is higher (TCP/socket + MVCC) vs in-process SQLite (still <1ms locally).

### Option C — Hybrid (recommended): per-bot SQLite (hot) + PostgreSQL (fleet-aggregate)
Tier 1 = per-bot SQLite WAL (system of record for that bot, synchronous, low-latency). Tier 2 = PostgreSQL fed asynchronously via an **outbox** in each bot's SQLite, drained by a single `collector` service.
- ✅ Hot path never blocks on the network or a shared lock. Each bot is independently durable.
- ✅ Full fleet unification in PG for reporting, alerting, evaluator, retention.
- ✅ Bot crash can't lose data the collector hasn't shipped (outbox is durable in the bot's own DB).
- ✅ Migration is incremental: ship the shared schema to bots first (fixes bugs), add the collector later; rollback is trivial (stop collector).
- ❌ Two storage systems to operate (but PG here is "analytics/archive", not on the hot path, so its downtime is non-critical).
- ❌ Eventual consistency between Tier 1 and Tier 2 (seconds, acceptable for reporting).

### Option D — Single shared SQLite (all bots write one file)
One `fleet.db` with `bot_id`.
- ✅ Unified and simple, no second system.
- ❌ Single-writer lock now arbitrates between bots; a slow/long transaction in one bot stalls others' commits (mitigated by `busy_timeout` + batching, but still serialized). At your volume it *would* work, but it re-introduces the exact contention risk you're worried about and gives up per-bot independence.
- ❌ One file = one point of failure for all bots.

### Decision matrix
| Criterion | A (per-bot SQLite) | B (unified PG) | **C (hybrid)** | D (shared SQLite) |
|---|---|---|---|---|
| Cross-bot unification | ❌ | ✅ | ✅ | ✅ |
| Hot-path write latency | ✅ µs | ⚠️ <1ms | ✅ µs | ⚠️ serialized |
| Concurrent multi-writer | ✅ (per file) | ✅ (MVCC) | ✅ (per file) | ❌ single writer |
| Operational overhead | ✅ none | ❌ high | ⚠️ medium | ✅ none |
| Blast radius of one failure | ✅ isolated | ❌ all bots | ✅ isolated | ❌ all bots |
| Time-series retention/rollups | ⚠️ manual | ✅ native | ✅ in PG | ⚠️ manual |
| Evaluator / DSR+CSCV feeding | ❌ 4 files | ✅ | ✅ | ⚠️ one file |

**Verdict: Option C** (D7 confirmed by val 2026-07-30). Ship a single shared schema to all four bots (Option A's safety + standardization), then add the PostgreSQL aggregate (Option B's unification) behind an outbox so the hot path stays local and independent.

---

## 2. Recommended Architecture (Option C)

```
┌────────────────┐  ┌────────────────┐  ┌────────────────┐  ┌─────────────────┐
│ vaisravana     │  │ vaisravana-wave│  │ vaisravana-    │  │ listener/       │
│ (SQLite WAL)   │  │ (SQLite WAL)   │  │ alpha          │  │ fatty (SQLite)  │
│  per-bot.db    │  │ per-bot.db     │  │ per-bot.db     │  │ per-bot.db      │
└───────┬────────┘  └───────┬────────┘  └───────┬────────┘  └────────┬────────┘
        │ writes           │ writes            │ writes            │ writes
        │  (1) system      │                   │                   │
        │      of record   │                   │                   │
        └────────────┬─────┴───────────────────┴───────────────────┘
                     │  outbox table (durable, local)
                     │  drain every N events / T ms
                     ▼
             ┌───────────────────────┐
             │  collector service     │  (single process, owns PG pool)
             │  - reads outbox        │
             │  - upserts to PG       │
             │  - marks shipped ✓     │
             └───────────┬───────────┘
                         │
                         ▼
             ┌───────────────────────┐
             │  PostgreSQL            │
             │  trading_fleet         │  partitioned by day, rollups,
             │                        │  retention, JSONB for signals
             └───────────┬───────────┘
                         │  fleet queries (batch only)
                         ▼
               Alerts / Telegram / Evaluator (DSR,CSCV,PBO)
```

**Why an outbox instead of dual-write:** The bot writes to its own SQLite synchronously (never loses a trade), and appends the same event to a local `outbox` table. The collector (a separate lightweight container or cron) drains the outbox into PG. If PG is down, the outbox just grows; when it recovers, the collector catches up. The bot's hot path never waits on the network — this is the critical design guarantee for a VPS with intermittent connectivity.

---

## 3. Real-Time Ingestion Without Contention

### 3.1 Per-bot (Tier 1)
- **File ownership = no contention.** Each bot writes only its own file. SQLite's single-writer lock is internal to one process; four separate files means four independent locks. This is the single most important design point: *keep writers on separate files* and the concurrency problem disappears at the hot tier.
- **In-process write queue.** Within a bot, serialize all DB writes through one `queue.Queue` + a dedicated writer thread (or `asyncio` task). This avoids SQLite's "multiple connections from one process" footguns and keeps transactions short.
- **Batching.** Buffer events; flush every `N=100` events or `T=250ms`, whichever comes first, inside a single `BEGIN/COMMIT`. This collapses many lock acquisitions into one and is the standard fix for "WAL overruns / missing data" reported by heavy writers.
- **WAL + busy_timeout.** `PRAGMA journal_mode=WAL; PRAGMA busy_timeout=5000;` with a single writer thread, `busy_timeout` is a safety net, not a hot path.

### 3.2 Fleet (Tier 2 → PostgreSQL)
- **Connection pool per bot** (SQLAlchemy `QueuePool`, `pool_size=5`, `max_overflow=10`) or a single `pgbouncer` in transaction mode. Never share a pool across forked children (dispose engine in child, or use `NullPool`).
- **Bulk load.** Use `COPY` / `executemany` for the collector's drain, not row-by-row `INSERT`.
- **Idempotent upsert.** Outbox events carry a deterministic `event_id` (e.g., `bot_id || ':' || entity || ':' || seq`); collector uses `INSERT ... ON CONFLICT (event_id) DO NOTHING` so replays are safe.
- **Ordering.** Outbox is append-ordered by an autoincrement `outbox_seq`; collector drains in `outbox_seq` order so PG reflects causal order.

---

## 4. Unified Schema (runs on SQLite and PostgreSQL)

Conventions:
- All timestamps stored as **INTEGER epoch milliseconds UTC** (`ts_ms`) for natural time-clustering and dialect portability; also a `created_at` for PG partitioning.
- Money/price stored as `REAL` (SQLite) / `NUMERIC` (PG) with documented precision; qty as `REAL`.
- `bot_id` is a short enum: `vaisravana`, `vaisravana-wave`, `vaisravana-alpha`, `listener`.
- `NULL` means "not yet known" (critical for the 0.0 bug). Use `CHECK` constraints to forbid silent `0.0`.

### 4.1 Dimension / registry tables
```sql
CREATE TABLE bot (
  bot_id        TEXT PRIMARY KEY,
  display_name  TEXT,
  strategy      TEXT,
  exchange      TEXT,
  trading_mode  TEXT,          -- spot | margin | futures
  margin_mode   TEXT,           -- isolated | cross
  stake_currency TEXT,
  created_at    INTEGER
);

CREATE TABLE account (
  account_id  TEXT PRIMARY KEY,   -- e.g. 'vaisravana@binance'
  bot_id      TEXT NOT NULL REFERENCES bot(bot_id),
  exchange    TEXT,
  api_label   TEXT
);

CREATE TABLE instrument (
  pair            TEXT PRIMARY KEY,   -- 'BTC/USDT'
  base            TEXT,
  quote           TEXT,
  tick_size       REAL,
  lot_size        REAL,
  maker_fee       REAL,              -- configured/observed
  taker_fee       REAL,
  price_precision INTEGER,
  qty_precision   INTEGER
);
```

### 4.2 Positions / Trades (the core lifecycle entity)
A **position** is one logical trade (open→closed). Multiple fills and orders attach to it. This mirrors Freqtrade's `trades` model but with explicit marker + fee separation and `NULL`-safe prices.

```sql
CREATE TABLE position (
  position_id     INTEGER PRIMARY KEY,           -- autoincrement
  bot_id          TEXT NOT NULL REFERENCES bot(bot_id),
  account_id      TEXT NOT NULL REFERENCES account(account_id),
  pair            TEXT NOT NULL REFERENCES instrument(pair),
  side            TEXT NOT NULL,                 -- long | short
  leverage        REAL NOT NULL DEFAULT 1.0,
  status          TEXT NOT NULL,                 -- open | closed | liquidated | canceled
  -- entry (NULL until first fill confirms)
  entry_price     REAL,                          -- NULL until filled; CHECK prevents 0.0
  entry_qty       REAL,
  entry_ts_ms     INTEGER,
  entry_source    TEXT,                          -- signal | fill  (audit: was price real?)
  -- exit
  exit_price      REAL,
  exit_qty        REAL,
  exit_ts_ms      INTEGER,
  exit_reason     TEXT,                          -- signal | sl | tp | manual | liquidation
  -- PnL (computed, persisted for fast reads)
  unrealized_pnl  REAL,
  realized_pnl    REAL NOT NULL DEFAULT 0.0,
  fee_open_total  REAL NOT NULL DEFAULT 0.0,     -- sum of fee_ledger where leg='open'
  fee_close_total REAL NOT NULL DEFAULT 0.0,     -- sum of fee_ledger where leg='close'
  net_pnl         REAL GENERATED ALWAYS AS (realized_pnl - fee_open_total - fee_close_total) STORED,
  max_price       REAL,                          -- observed extreme while open
  min_price       REAL,
  created_at      INTEGER,
  updated_at      INTEGER,
  CONSTRAINT chk_entry_price CHECK (entry_price IS NULL OR entry_price > 0),
  CONSTRAINT chk_exit_price  CHECK (exit_price  IS NULL OR exit_price  > 0),
  CONSTRAINT chk_side CHECK (side IN ('long','short')),
  CONSTRAINT chk_status CHECK (status IN ('open','closed','liquidated','canceled'))
);
CREATE INDEX idx_position_bot_open ON position(bot_id, status, updated_at DESC);
CREATE INDEX idx_position_pair_ts   ON position(bot_id, pair, entry_ts_ms DESC);
```

### 4.3 Orders (exchange order lifecycle)
```sql
CREATE TABLE orders (
  order_id      TEXT PRIMARY KEY,                 -- exchange order id
  exchange_oid  TEXT,
  bot_id        TEXT NOT NULL,
  position_id   INTEGER REFERENCES position(position_id),
  pair          TEXT NOT NULL,
  side          TEXT NOT NULL,                    -- buy | sell | stoploss
  order_type    TEXT,                             -- limit | market | stop | stop_limit
  status        TEXT NOT NULL,                    -- open | closed | canceled | expired | rejected
  requested_price REAL,
  requested_qty   REAL,
  filled_price    REAL,
  filled_qty      REAL,
  avg_price       REAL,
  cost            REAL,
  ts_ms           INTEGER,
  updated_at      INTEGER
);
CREATE INDEX idx_orders_position ON orders(position_id);
CREATE INDEX idx_orders_bot_ts   ON orders(bot_id, ts_ms DESC);
```

### 4.4 Fills (execution reports — the source of truth for prices & fees)
```sql
CREATE TABLE fill (
  fill_id       INTEGER PRIMARY KEY,
  bot_id        TEXT NOT NULL,
  position_id   INTEGER REFERENCES position(position_id),
  order_id      TEXT,
  pair          TEXT NOT NULL,
  side          TEXT NOT NULL,                    -- buy | sell
  is_open_leg   INTEGER NOT NULL,                 -- 1 = opening leg, 0 = closing leg
  price         REAL NOT NULL CHECK (price > 0),  -- REAL fill price, never 0.0
  qty           REAL NOT NULL CHECK (qty > 0),
  notional      REAL NOT NULL,                    -- price*qty in quote
  fee_rate      REAL,                             -- maker or taker rate applied
  fee_cost      REAL,                             -- absolute fee in fee_currency
  fee_currency  TEXT,
  fee_type      TEXT,                             -- maker | taker  (from exchange 'makerOrTaker'/'takerOrMaker')
  liquidity     TEXT,                             -- maker | taker (alias of fee_type for clarity)
  ts_ms         INTEGER NOT NULL,
  raw_json      TEXT                              -- full ccxt fill, JSON, for audit/replay
);
CREATE INDEX idx_fill_position ON fill(position_id);
CREATE INDEX idx_fill_bot_ts   ON fill(bot_id, ts_ms DESC);
```

### 4.5 Markers (entry / SL / TP / exit targets) — fixes the "0.0" bug
A marker is a *planned or observed* price level tied to a position. It is `NULL` until known; we forbid `0.0`.

```sql
CREATE TABLE marker (
  marker_id     INTEGER PRIMARY KEY,
  bot_id        TEXT NOT NULL,
  position_id   INTEGER NOT NULL REFERENCES position(position_id),
  kind          TEXT NOT NULL,                    -- entry | sl | tp | be | exit
  price         REAL,                             -- NULL until set; CHECK(price>0 OR price IS NULL)
  pct_from_entry REAL,                            -- e.g. SL = -0.05, TP = +0.10
  qty           REAL,
  status        TEXT NOT NULL DEFAULT 'planned', -- planned | active | triggered | canceled
  set_at_ts_ms  INTEGER,                          -- when this level was *established*
  trigger_ts_ms INTEGER,                          -- when it was hit (NULL if not)
  trigger_price REAL,                             -- actual fill price when triggered
  source        TEXT,                             -- computed | exchange | manual
  created_at    INTEGER,
  CONSTRAINT chk_marker_price CHECK (price IS NULL OR price > 0),
  CONSTRAINT chk_marker_kind CHECK (kind IN ('entry','sl','tp','be','exit')),
  CONSTRAINT chk_marker_status CHECK (status IN ('planned','active','triggered','canceled'))
);
CREATE INDEX idx_marker_position ON marker(position_id);
CREATE INDEX idx_marker_bot_kind ON marker(bot_id, kind, status);
```

**Why this kills the 0.0 bug:** Markers are inserted with `price = NULL` at *plan* time. The entry marker is only set to a real number when the **first fill** confirms; SL/TP markers are derived from `entry_price * (1 + pct)` *after* entry exists (so they can never be `entry 0.0 * pct`). The `CHECK` rejects any accidental `0.0` insert. Notifications render `price` only when non-NULL.

### 4.6 Fee ledger (explicit open + close, maker/taker split) — fixes the fee bug
```sql
CREATE TABLE fee_ledger (
  fee_id       INTEGER PRIMARY KEY,
  bot_id       TEXT NOT NULL,
  position_id  INTEGER REFERENCES position(position_id),
  fill_id      INTEGER REFERENCES fill(fill_id),
  leg          TEXT NOT NULL,                     -- open | close
  fee_type     TEXT NOT NULL,                     -- maker | taker
  rate         REAL NOT NULL,                     -- e.g. 0.0004
  cost         REAL NOT NULL,                     -- absolute fee in fee_currency
  currency     TEXT NOT NULL,
  ts_ms        INTEGER NOT NULL
);
CREATE INDEX idx_fee_position ON fee_ledger(position_id);
CREATE INDEX idx_fee_bot_leg  ON fee_ledger(bot_id, leg, ts_ms DESC);
```
The bot **always** writes one `fee_ledger` row per fill, capturing `leg` from whether the fill is an opening or closing leg, and `fee_type` from the exchange's reported liquidity (`fill['info']['makerOrTaker']` on Binance/Futures). `position.fee_open_total` / `fee_close_total` are maintained by triggers (PG) or by the writer (SQLite) as the `SUM(cost)` per leg. This guarantees fees are accounted on **both** open and close.

### 4.7 Signals (strategy entries/exits from the unified DB view)
```sql
CREATE TABLE signal (
  signal_id        INTEGER PRIMARY KEY,
  bot_id           TEXT NOT NULL,
  pair             TEXT NOT NULL REFERENCES instrument(pair),
  direction        TEXT NOT NULL,                 -- long | short
  kind             TEXT NOT NULL,                 -- enter | exit
  tag              TEXT,                          -- enter_tag / exit_tag / source_name
  timeframe        TEXT,
  price            REAL,
  strength         REAL,                          -- 0..1 confidence
  indicators       TEXT,                          -- JSON blob of indicator snapshot at signal time
  ts_ms            INTEGER NOT NULL,
  resulted_in_position_id INTEGER REFERENCES position(position_id)
);
CREATE INDEX idx_signal_bot_ts ON signal(bot_id, ts_ms DESC);
CREATE INDEX idx_signal_pair   ON signal(pair, ts_ms DESC);
```

### 4.8 Evaluations (feeds the autonomous loop's DSR/CSCV/PBO)
Aligned with the autonomous-trading-bot-loop skill: the evaluator loads trades and computes Deflated Sharpe, CSCV PBO, verdict. Persist results so promotion is reproducible.
```sql
CREATE TABLE evaluation (
  eval_id        INTEGER PRIMARY KEY,
  bot_id         TEXT NOT NULL,
  candidate_id   TEXT,                            -- DGM/agentic candidate label
  window_start_ms INTEGER,
  window_end_ms   INTEGER,
  n_trades        INTEGER,
  sharpe          REAL,
  deflated_sharpe REAL,
  dsr_p           REAL,
  cscv_pbo        REAL,
  worst_r         REAL,
  verdict         TEXT,                           -- KEEP | REJECT | INCONCLUSIVE
  lineage_parent  TEXT,
  created_at      INTEGER
);
CREATE INDEX idx_eval_bot ON evaluation(bot_id, created_at DESC);
```

### 4.9 Equity / balance snapshot — fixes the missing footer
Written **every cycle** (or on every position/balance change) per bot. Notifications read `latest_equity(bot_id)`.
```sql
CREATE TABLE equity_snapshot (
  snap_id        INTEGER PRIMARY KEY,
  bot_id         TEXT NOT NULL,
  ts_ms          INTEGER NOT NULL,
  total_balance  REAL NOT NULL,   -- wallet total in stake currency
  free_balance   REAL NOT NULL,   -- unused
  used_margin    REAL NOT NULL,   -- margin tied in open positions
  unrealized_pnl REAL NOT NULL DEFAULT 0.0,
  realized_pnl   REAL NOT NULL DEFAULT 0.0,
  equity         REAL NOT NULL,   -- total_balance + unrealized_pnl (mark-to-market)
  open_positions INTEGER,
  drawdown_pct   REAL,
  note           TEXT
);
CREATE INDEX idx_equity_bot_ts ON equity_snapshot(bot_id, ts_ms DESC);
```
Notification footer = `SELECT * FROM equity_snapshot WHERE bot_id=? ORDER BY ts_ms DESC LIMIT 1;`

### 4.10 Outbox (Tier 1 → Tier 2 shipping)
```sql
CREATE TABLE outbox (
  outbox_seq   INTEGER PRIMARY KEY AUTOINCREMENT,
  event_id     TEXT NOT NULL UNIQUE,              -- bot_id:entity:seq (idempotency key)
  bot_id       TEXT NOT NULL,
  entity       TEXT NOT NULL,                     -- position|fill|marker|fee|signal|equity|evaluation
  payload      TEXT NOT NULL,                     -- JSON of the row
  created_at   INTEGER NOT NULL,
  shipped_at   INTEGER                             -- NULL until collector ships
);
CREATE INDEX idx_outbox_unsent ON outbox(shipped_at, outbox_seq);
```

### 4.11 Optional: market data (for backfill / context)
```sql
CREATE TABLE candle (
  pair        TEXT NOT NULL,
  timeframe   TEXT NOT NULL,
  ts_ms       INTEGER NOT NULL,
  open REAL, high REAL, low REAL, close REAL, volume REAL,
  PRIMARY KEY (pair, timeframe, ts_ms)
);
```

---

## 5. Fixing the Four Concrete Bugs (schema + write logic)

### 5.1 Fee on BOTH open and close (maker/taker split)
**Write rule (pseudo):**
```python
for fill in exchange.fetch_my_trades()/fills:
    leg = 'open' if is_opening_leg(position, fill) else 'close'
    fee_type = fill['info'].get('makerOrTaker') or fill.get('takerOrMaker') or infer(price vs book)
    rate = fill['fee']['rate']
    cost = fill['fee']['cost']
    # ALWAYS persist, for both legs:
    db.insert('fee_ledger', bot_id, position_id, fill_id, leg, fee_type, rate, cost, currency, ts)
    db.insert('fill', ..., fee_type=fee_type, fee_rate=rate, fee_cost=cost, ...)
# after all fills for a position leg:
pos.fee_open_total  = SUM(fee_ledger.cost WHERE leg='open')
pos.fee_close_total = SUM(fee_ledger.cost WHERE leg='close')
```
Key: do **not** gate fee capture on `side == 'sell'`; capture on every fill and tag the leg. The `net_pnl` generated column subtracts both.

### 5.2 entry/SL/TP never 0.0
- Insert `position` with `entry_price = NULL`.
- On first fill: `UPDATE position SET entry_price = fill.price, entry_qty = fill.qty, entry_ts_ms = fill.ts_ms, entry_source='fill' WHERE position_id=? AND entry_price IS NULL;`
- Only *after* entry exists, compute markers: `INSERT marker(kind='sl', price = entry_price*(1 - sl_pct), pct_from_entry = -sl_pct, status='active', source='computed')`. Because `entry_price > 0` is enforced and SL derives from it, SL can never be `0.0`.
- Notification guard: render `price` only when `IS NOT NULL`.

### 5.3 Balance / Used / Unrealized / Realized footer always present
- Every main loop cycle (and on every fill / position open-close), compute and `UPSERT` one `equity_snapshot` row.
- Notification builder **always** joins the latest `equity_snapshot` for the bot; if somehow missing, compute from `position` aggregates as a fallback so the footer is never blank.

### 5.4 General correctness guards
- `CHECK (price > 0 OR price IS NULL)` on every price column.
- `NOT NULL DEFAULT 0.0` only on *sums* (fees, realized_pnl), never on observed prices.
- `entry_source` audit column distinguishes a real fill price from a (possibly wrong) signal price — directly addresses "entry shows 0.0 / wrong".

---

## 6. Time-Series Optimization

### 6.1 SQLite (Tier 1)
- **Natural clustering:** `INTEGER PRIMARY KEY` (rowid) and `ts_ms` indexes keep time-ordered data physically sequential → range scans are cheap.
- **WAL + checkpoint:** `PRAGMA journal_mode=WAL; PRAGMA wal_autocheckpoint=1000;` periodically checkpoint to avoid WAL growth. `PRAGMA synchronous=NORMAL` (safe with WAL on local SSD; `FULL` if you want max durability).
- **Indexes:** only what queries need — `(bot_id, ts DESC)` for recent reads, `(position_id)` for joins. Avoid over-indexing write-heavy tables.
- **Maintenance:** schedule `PRAGMA wal_checkpoint(TRUNCATE);` and occasional `VACUUM;` (off-peak) to reclaim space and refresh stats. `PRAGMA optimize;` after big changes.
- **Retention:** hot SQLite keeps ~90 days raw; older rows can be `DELETE` (or copied to a cold archive SQLite) by a nightly job using the `ts_ms` index.

### 6.2 PostgreSQL (Tier 2) — native time-series
- **Range partitioning by day** on `created_at` for `position`, `fill`, `fee_ledger`, `signal`, `equity_snapshot`, `evaluation`. Partition pruning makes "last 24h per bot" scans touch one partition.
- **Per-partition indexes:** rich b-tree `(bot_id, ts_ms)` on recent partitions; **BRIN** index on `ts_ms` for cold partitions (tiny, ideal for append-only time-series).
- **Retention by DETACH + DROP** (metadata-only, no table scan, no vacuum pain):
  ```sql
  ALTER TABLE fill DETACH PARTITION fill_2026_04;
  DROP TABLE fill_2026_04;  -- or move to cold storage first
  ```
  Automate with `pg_partman` (`retention = '90 days'`, `retention_keep_table = false`).
- **Rollups:** pre-aggregate hourly/daily PnL into `pnl_rollup(bot_id, day, realized, fees, trades, max_dd)` so dashboards answer in milliseconds instead of scanning millions of fills.
- **TimescaleDB (optional):** if you want hypertables + continuous aggregates, it's a drop-in extension; same SQL otherwise. Not required for this scale.
- **Storage tiering:** hot partitions on fast volume, old partitions on cheaper volume via tablespaces.

---

## 7. WAL, Connection Pooling & Concurrency (Docker)

### 7.1 SQLite across Docker containers — it works on one host
Research (Simon Willison, Apr 2026; Rick Branson / Segment; numerous HN threads) confirms: **WAL-mode SQLite shared across containers on the same host works** because the containers share the host kernel and filesystem semantics (including the `-shm` shared-memory file). Caveats:
- ✅ Must be the **same host / local volume** — never NFS/network filesystem (WAL needs shared memory; NFS is unreliable for the locking primitives).
- ✅ Each bot owns its own file → no cross-container lock arbitration (our design).
- ✅ Set `PRAGMA busy_timeout=5000` so a brief lock wait blocks instead of erroring (`SQLITE_BUSY`).

**Required PRAGMAs (apply at connection open, every connection):**
```sql
PRAGMA journal_mode = WAL;
PRAGMA busy_timeout = 5000;
PRAGMA synchronous  = NORMAL;      -- WAL-safe on local SSD; use FULL for max durability
PRAGMA foreign_keys = ON;
PRAGMA wal_autocheckpoint = 1000;
-- PRAGMA mmap_size = <1TiB on 64-bit>; only if DB fits in address space
```

### 7.2 SQLite connection discipline (per bot)
- One writer connection per bot (the dedicated writer thread). Readers (alerts, notifications, evaluator) open their own short-lived connections with `busy_timeout` — WAL lets them read without blocking the writer.
- `check_same_thread=False` only if you truly share a connection across threads; safer to give each thread its own connection.
- **Never fork a pooled connection** into a child process (multiprocessing). If you spawn workers, either `engine.dispose()` in the child or use `NullPool` / open connections lazily per process.

### 7.3 PostgreSQL pooling (Tier 2 collector)
- **SQLAlchemy `QueuePool`**: `pool_size=5`, `max_overflow=10`, `pool_pre_ping=True`, `pool_recycle=1800`. The collector is one process → one pool is fine.
- **pgbouncer** (transaction mode) if you later add more writers; keeps PG's per-connection process cost low (PG spawns a process per connection; 5000 connections will deadlock a default server — pooled connections are mandatory at scale).
- For analytics/reports, point reads at the PG instance (or a read replica) so heavy scans never touch bot hot paths.

### 7.4 Docker specifics on `sera`
- **Volumes:** mount the DB directory as a **named volume** or bind-mount to local SSD (`/var/lib/docker/volumes/...` or `/opt/bots/**/data`). Back it by the instance's local NVMe/SSD, not network storage.
- **Backups:** SQLite → `sqlite3 <db> ".backup 'file'" ` or copy the `db` + `db-wal` + `db-shm` atomically (stop writer or use `.backup`). PG → `pg_basebackup` / `pg_dump` nightly; consider `litestream`-style WAL shipping for SQLite if you want continuous off-host backup.
- **Healthchecks:** add a `HEALTHCHECK` that runs a trivial `SELECT 1` / `PRAGMA integrity_check` so orchestration knows a DB is corrupt.
- **`shm_size`** for the PG container (default 64 MB is often too small for heavy sorts); set `shm_size: 256mb`.
- **Zero-downtime deploy:** spin up the new container on the same volume, wait for healthy, then stop the old one (WAL makes this safe).

---

## 8. Query Patterns: Fast vs Batch

### 8.1 Fast path (per tick / per notification / per alert) — must be <5ms
| Query | Index | Store |
|---|---|---|
| Latest equity footer for a bot | `equity_snapshot(bot_id, ts_ms DESC) LIMIT 1` | Tier 1 (SQLite) |
| Open positions for a bot | `position(bot_id, status, updated_at DESC)` where status='open' | Tier 1 |
| Recent fills (last N) | `fill(bot_id, ts_ms DESC) LIMIT N` | Tier 1 |
| Current unrealized PnL | from latest `equity_snapshot` | Tier 1 |
| SL/TP levels for open positions | `marker(position_id)` where status='active' | Tier 1 |

These are all covered by the `(bot_id, ts DESC)` / `(position_id)` indexes and return in microseconds from SQLite. Keep them on Tier 1 so notifications never depend on the network or PG.

### 8.2 Batch path (reports / evaluator / fleet rollups) — can take seconds, run off-peak
| Query | Store | Note |
|---|---|---|
| Full trade history for a bot | Tier 2 (PG) partitioned by day | Feeds evaluator DSR/CSCV/PBO |
| Fleet PnL across all 4 bots | Tier 2 | cross-bot JOIN |
| CSCV block splits / DSR | Tier 2 → evaluator service | heavy scan, time-ordered |
| Daily/weekly performance rollups | Tier 2 `pnl_rollup` | pre-aggregated |
| Backtest/replay from `raw_json` fills | Tier 2 `fill` | audit/replay |

The autonomous loop's evaluator loads `position`/`fill` from the DB and computes Deflated Sharpe, CSCV PBO, verdict — this is a **batch** workload; run it on Tier 2 (or a read replica) so it never competes with hot writes. Persist results to `evaluation` for reproducible promotion.

---

## 9. Migration Path (incremental, low-risk)

1. **Phase 0 — Shared schema package.** Create `tradingdb` (Python) with the DDL above (SQLite dialect) + a `Base` model + connection helper applying the §7.1 PRAGMAs. All four bots import it; each still writes its own file. **This alone fixes the fee/0.0/footer bugs** because the write logic lives in one place.
2. **Phase 1 — Backfill + validation.** Add the new tables to existing DBs via `PRAGMA user_version` migration scripts; backfill `equity_snapshot` from historical `position` aggregates; alert if any `entry_price = 0`.
3. **Phase 2 — Outbox + collector.** Each bot appends events to `outbox`; deploy the `collector` container that drains to PostgreSQL. PG starts empty and fills within minutes.
4. **Phase 3 — Fleet features.** Cross-bot Telegram digests, fleet PnL, evaluator reads from PG.
5. **Rollback:** stop the collector → bots are fully independent again (Tier 1 unchanged). No lock-in.

---

## 10. Key Risks & Mitigations

| Risk | Mitigation |
|---|---|
| SQLite corruption from unclean shutdown | WAL + `synchronous=NORMAL`; `.backup` nightly; `PRAGMA integrity_check` healthcheck. |
| Outbox grows unbounded if collector down | Monitor `outbox` unsent count; alert; collector is idempotent so safe to replay. |
| PG outage blocks bots | PG is Tier 2 only; bots never block on it. Collector retries with backoff. |
| Schema drift between bots | Single `tradingdb` package is the only source of truth; `user_version` migrations. |
| Forked-process connection sharing | `engine.dispose()` in children / `NullPool` for multiprocessing workers. |
| Network filesystem WAL breakage | DBs live on local named volume; never NFS. |

---

## 11. References (research sources)

- SQLite WAL across Docker containers — Simon Willison (2026-04-07): https://simonwillison.net/2026/Apr/7/sqlite-wal-docker-containers/
- SQLite concurrent writes / "database is locked" — https://tenthousandmeters.com/blog/sqlite-concurrent-writes-and-database-is-locked-errors/ and SQLite FAQ (WAL allows readers+1 writer).
- PostgreSQL partitioning & retention (DETACH+DROP, pg_partman, BRIN) — Tiger Data / Timescale learn docs; Medium "9 Postgres Partitioning Strategies for Time-Series at Scale" (2025).
- SQLite vs PostgreSQL trade-offs — Tinybird "Postgres vs SQLite"; Kunal Ganglani "SQLite vs PostgreSQL 2026"; HN "Have you used SQLite as a primary database?" (extensive practitioner war stories).
- Freqtrade DB model (reference for `trades`/`orders`/`fee_open`/`fee_close`/`realized_profit` fields) — https://www.freqtrade.io/en/stable/trade-object/ and `freqtrade/persistence/trade_model.py`.
- Connection pooling with multiprocessing / os.fork — SQLAlchemy docs "Using Connection Pools with Multiprocessing".
- Sharing SQLite across containers (Segment/ctlstore) — Rick Branson, "Sharing an SQLite database across containers is surprisingly brilliant."
- Autonomous trading-bot-loop skill (evaluator DSR/CSCV/PBO, 3-layer architecture) — local Hermes skill `autonomous-trading-bot-loop`.

---

*End of design document. No code was executed; this is a planning artifact. The schema DDL is ready to drop into the `tradingdb` package and the PostgreSQL `trading_fleet` database.*
