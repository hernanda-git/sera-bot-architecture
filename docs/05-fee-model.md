# Fee Model — Binance Maker/Taker Analysis for Small Accounts

**Status:** Design / planning. No execution.
**Part of:** sera-bot-architecture
**Source:** extracted + expanded from docs/04-honest-evaluation.md section 3.

---

## 0. The spec vs reality

Val's stated fee spec: **maker 0.02% open + taker 0.04% close = 6.0 bps round-trip**.

The evaluation found this is **mildly optimistic**. Verified Binance USDⓈ-M Futures
schedule (2026-07-30):

| Tier | 30d volume | BNB | Maker | Taker |
|---|---|---|---|---|
| VIP0 (regular) | 0 | — | **0.0200%** | **0.0500%** |
| VIP0 + BNB 10% | 0 | any | 0.0180% | 0.0450% |
| VIP1 | >= 5M | >= 5 | 0.0180% | 0.0500% |
| VIP2 | >= 10M | >= 25 | 0.0160% | 0.0400% |
| VIP3+ | >= 20M | >= 100 | 0.0140% → | 0.0350% → (negative maker rebates at top tiers) |

### The two findings that matter

1. **The spec understates taker cost.** Real VIP0 taker is **0.05%**, not 0.04%.
   True round-trip = **7.0 bps** (6.3 bps with BNB). The survival gate's
   `FEE_BPS_ROUNDTRIP=6.0` therefore **understates cost by ~14%** — meaning the EV
   gate occasionally passes trades that are actually fee-negative. *Fix: set RT to
   7.0 (6.3 with BNB).*
2. **Rebates are gated to 8-figure monthly volume.** The maker rebates that
   "fund" market-making are unreachable at this account size. Rebate-harvesting MM
   is dead for sub-institutional capital. The bot is correctly *not* an MM bot.

---

## 1. Small-account math ($10–$100)

Fees are charged on **notional**, not equity. With `max_position_size_usdt: 5`, a
typical position is a few USD notional, so absolute fees per trade are cents. The
problem is **not** absolute fee size — it is **edge-per-trade < fee-per-trade**:

- The scalping research doc measured gross ≈ breakeven at 45% WR / 106 trades per
  10h, fees −$8.40 → the strategy had **< 8 bps average captured move vs ~7 bps
  round-trip cost**.
- Perp directional scalping after taker fees needs implausible win rates
  (~92% on spot per tv-hub 2026 math; ~57% even for maker futures). **No EMA/SMC
  strategy sustains that.**
- **Therefore: at $10–$100, a directional taker strategy cannot be net-positive.
  The only positive-expectancy paths at this size are (a) maker/post-only entries
  to halve cost, and (b) structural carry (funding) that is not a latency game.**

The bot's survival gates are precisely the right response to this — but they *cap
the bleed*; they do not *create* the edge that makes the remaining trades net-positive.

---

## 2. Recommended fee model for the unified DB

From docs/01-database-architecture.md, the `fee_ledger` table captures every fill
with `leg ∈ {open, close}`, `fee_type ∈ {maker, taker}`, `rate`, `cost`, `currency`.
This makes the actual fee model observable and auditable:

- **Open leg:** maker 0.02% (post-only / limit-inside-book) OR taker 0.05% (market /
  crossing). Capture the *actual* liquidity from the exchange's `makerOrTaker` flag.
- **Close leg:** taker 0.04% (or 0.05% — verify against the live `takerOrMaker`
  flag, do not assume 0.04%).
- **Net PnL** = realized_pnl − fee_open_total − fee_close_total (generated column).
- **Notification** shows the split explicitly: `(open 0.3000 + close 0.2000)` so the
  owner can audit the round-trip without mental math.

### Config constants to fix now (non-destructive, ParameterSurface-tunable)
| Constant | Current | Recommended |
|---|---|---|
| `FEE_BPS_ROUNDTRIP` (EV gate) | 6.0 | 7.0 (6.3 with BNB) |
| `OPEN_FEE_RATE` | 0.0002 | keep 0.0002 (maker) but verify it's actually maker |
| `CLOSE_FEE_RATE` | 0.0004 | **verify** — Binance VIP0 taker is 0.0005 |

---

## 3. The structural edge sized for this account: funding carry

The BIS "Crypto Carry" (2025) paper shows funding carry averages ~7% annualized
(BTC/ETH), spiking to 40–60% at peaks, driven by retail trend-chasing leverage.
A **delta-neutral** structure (long spot / short perp, or short-perp-collects-funding)
is *not a latency game* and is the least-bad path for small capital. The current
bot is directional only — it ignores the one edge sized for it.

This is a Phase-D research item (engine change, needs val's approval per Sentinel),
not a ParameterSurface tweak. Documented here so it is on the roadmap.

---

## 4. Bottom line

- The fee spec is close but optimistic by ~14%. Fix the constant.
- At $10–$100 the directional taker strategy is structurally fee-negative; the fix
  is **signal quality** (see 04-honest-evaluation.md section 4) + **cheap
  maker entries** + optionally **funding carry**, not more trades.
- The DB must record the *actual* fee per fill, not an assumed constant, so the
  EV gate and notifications are truthful.
