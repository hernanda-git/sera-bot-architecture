# Trade Notification Card — Design Spec (no-emoji, single-card)

**Audience:** `vaisravana` (main bot) + `vaisravana-wave` (wave bot)
**Constraint (hard user preference, v0.0.36):** clean / professional / elegant — **no emoji, ever**.
**Deliverable:** two self-contained card templates (OPEN + CLOSE) usable on both bots.

This document contains (a) the exact rendered templates, (b) the rationale behind every
design decision, (c) the UX research that drove it, and (d) drop-in Python you can paste
into both bots' notifiers.

---

## 1. The OPEN card (rendered)

```
┌─ LONG  ETHUSDT  15m
│ Entry     : 3000.00
│ SL        : 2980.00
│ TP        : 3040.00
│ Size      : 1500.00$ notional  (0.5000)
│ Lev/Conf : 3x  ·  0.42
│ Fee       : 0.3000$  (maker 0.02% open)
├─ BALANCE
│ Equity    : 9.94$
│ Used      : 1.50$    Open : 1
│ Unreal    : +0.21$   Realized : -0.06$
├─ PORTFOLIO
│ Win rate : 42.9%  ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░  (3W / 4L · 7)
│ Fees paid : 0.0210$
└─
```

Pending TP (the case that used to render `0.0`): `TP : —` (em-dash, not zero).

## 2. The CLOSE card (rendered)

WIN variant:
```
┌─ WIN  ETHUSDT  15m  (TP)
│ Exit      : 3040.00
│ PnL       : +2.00R
│ Gross     : +3.00$
│ Fee       : 0.5000$  (open 0.3000 + close 0.2000)
│ Net       : +2.50$
├─ BALANCE
│ Equity    : 9.97$
│ Used      : 1.50$    Open : 0
│ Unreal    : +0.00$   Realized : +2.44$
├─ PORTFOLIO
│ Win rate : 57.1%  ▓▓▓▓▓▓▓▓▓▓▓░░░░░░░░░  (4W / 3L · 7)
│ Fees paid : 0.0710$
└─
```

LOSS variant (header `LOSS`, `Net` negative, shorter bar):
```
┌─ LOSS  BTCUSDT  1h  (SL)
│ Exit      : 61100.00
│ PnL       : -1.00R
│ Gross     : -1.20$
│ Fee       : 0.4600$  (open 0.2400 + close 0.2200)
│ Net       : -1.66$
├─ BALANCE
│ Equity    : 9.94$
│ Used      : 1.50$    Open : 1
│ Unreal    : +0.21$   Realized : -0.06$
├─ PORTFOLIO
│ Win rate : 42.9%  ▓▓▓▓▓▓▓▓▓░░░░░░░░░░░  (3W / 4L · 7)
│ Fees paid : 0.0210$
└─
```

Both cards are self-contained: every performance metric the owner asked for
(Balance / Used / Unrealized / Realized / win rate / total trades W:L / cumulative fees)
is present on **every** open and close alert. No need to open `/status` to understand the state.

---

## 3. Design rationale — why each choice

| Decision | Why |
|---|---|
| **No emoji, text labels** | `LONG/SHORT` (not BUY/SELL), `WIN/LOSS` (not ✅/❌), `KILL-SWITCH TRIPPED` etc. val's explicit style rule (v0.0.36). Emoji read as "casual/retail"; text reads as "terminal/quant". |
| **Whole card in `<pre>`** | Telegram renders `<pre>` as **monospace**. The current main-bot card mixes proportional-font box chars with `<code>` numbers, so the `┌─`/`└─` frame floats and columns don't line up. Monospace makes the box frame crisp and every column aligns. |
| **Left-rail frame + `├─` section dividers** | `┌─` top, `├─ BALANCE` / `├─ PORTFOLIO` dividers, `└─` bottom. We deliberately **omit the right border `│`** — keeping a closed box requires computing exact string width per line, which is fragile in generated text. The left rail + dividers still give a strong "card" silhouette without the fragility. |
| **Bold title + bold section labels** | Wrap the title line and `BALANCE`/`PORTFOLIO` in `<b>`. Bold inside `<pre>` is supported by Telegram. Gives 3-tier hierarchy: **title > section > datum** — the eye lands on what changed first. |
| **`—` for missing Entry/SL/TP** | Resolves the "shows 0.0 when not filled" bug. `—` (em-dash) is a typographic char, **not** emoji (outside the `U+1F300–U+1FAFF` range), and is already the v0.0.36 convention for pending. NOTE: this overrides the old `telegram_bot.py` docstring line "Em-dashes never used" — that rule was about separator artifacts (e.g. `v0.0.8 —`), not the pending-value placeholder. |
| **Right-aligned money, `+`/`-` sign** | `:+`/`-` prefix on every PnL/Unrealized/Realized/Net makes green/red redundant — sign *is* the color cue, no emoji needed. Consistent 2dp (4dp for fees). |
| **`▓░` win-rate bar** | A 20-cell text bar (each cell = 5%) is the "cool professional" indicator val asked for — a **unicode block bar, not emoji**. It visualizes win rate at a glance next to the numeric `42.9%`. Bars use `U+2593`/`U+2591` (block elements), outside emoji ranges. |
| **Fee model spelled out in-line** | OPEN: `(maker 0.02% open)`. CLOSE: `(open 0.3000 + close 0.2000)` shows the **maker-open / taker-close split** explicitly, so the owner can audit the 6bps round-trip without mental math. |
| **`(W / L · total)` next to win rate** | One line answers "is my win rate real?" — sample size is right there. |
| **Compact (~14 lines)** | Well under Telegram's 4096-char cap; reads fully on a phone without scrolling. Single card, never split across messages. |
| **Same template, both bots** | `vaisravana.notify_fill/notify_close` and `vaisravana-wave.notify_wave_open/notify_wave_close` call the *same* renderer with the *same* `stats` dict → formats stay byte-identical (the standing rule). |

---

## 4. UX research — what makes a Telegram alert "professional"

Synthesized from trading-bot / fintech notification best practice (Telegram HTML parse mode):

1. **Monospace + box frame = "terminal" credibility.** Proportional fonts make tabular data feel like a chat message; monospace + `┌─┐│└─┘` makes it feel like a market terminal. This is the single biggest "cool" lever and it's free.
2. **Strict 3-tier hierarchy.** Title (bold) → section (bold) → datum (plain). Flat lists with no grouping read as noise. Dividers (`├─`) do the grouping work without extra lines.
3. **Contrast without color.** Professionals can't rely on green/red (dark-mode + accessibility + the no-emoji rule). Use **sign (`+/-`), uppercase RESULT words, and block bars** as the contrast mechanism. The sign prefix is the most legible PnL cue on a phone.
4. **One source of truth per alert.** The card must be self-contained — the owner should never need to cross-reference `/status` or `/wave` to know equity / win rate / fees. That's why the BALANCE + PORTFOLIO footer is on *both* open and close.
5. **Graceful emptiness.** Missing data renders as `—` (or `pending`), never `0.0` / `null` / `None`. A `0.0` SL looks like a real (lethal) stop; `—` unambiguously means "not set yet".
6. **Consistent label width.** All labels padded to 10 chars (`Entry     :`) so the colons form a vertical rule — the second "alignment" cue after the frame.
7. **Bounded length + scannable first line.** The first line carries the verdict (`LONG ETHUSDT 15m` / `WIN ETHUSDT 15m (TP)`); everything after is detail. A glance at line 1 is enough to triage.

---

## 5. Drop-in implementation (Python)

Paste into a shared notifier module (or replicate in both `telegram_bot.py` and `wave/engine.py`). Verified to render the cards above and to contain **zero emoji** (`regex [U+1F300-U+1FAFF]` + dingbat ranges → no match).

```python
import re

BOX_T, BOX_L, BOX_S, BOX_B = "┌─", "│ ", "├─", "└─"
_EMOJI = re.compile(r"[\U0001F300-\U0001FAFF\U00002600-\U000027BF\U0001F000-\U0001F02F]")


def _price(v) -> str:                       # Entry/SL/TP → — when pending
    return "—" if v is None or v == 0 else f"{v:.2f}"


def _money(v, signed=False) -> str:
    return f"{v:+.2f}$" if signed else f"{v:.2f}$"


def _pct(v) -> str:
    return f"{v:.1f}%"


def _bar(wr: float, width: int = 20) -> str:
    filled = max(0, min(width, round(wr / 100 * width)))
    return "▓" * filled + "░" * (width - filled)


def _row(label: str, value: str, lw: int = 10) -> str:
    return f"{BOX_L}{label:<{lw}}: {value}"


def _footer_lines(s: dict) -> list:
    equity = s.get("equity", s.get("balance", 0.0))
    used = s.get("margin", s.get("used", 0.0))
    unreal = s.get("unrealized", 0.0)
    realized = s.get("realized", 0.0)
    open_n = s.get("open_n", 0)
    wr = s.get("win_rate", s.get("wr", 0.0))
    total = s.get("total_trades", s.get("n", 0))
    wins = s.get("total_wins", s.get("wins", 0))
    losses = s.get("total_loses", s.get("losses", 0))
    fees = s.get("total_fee", s.get("fees_paid", 0.0))
    return [
        f"{BOX_S} BALANCE",
        _row("Equity", _money(equity)),
        f"{BOX_L}Used      : {_money(used)}    Open : {open_n}",
        f"{BOX_L}{'Unreal':<10}: {_money(unreal, True)}   Realized : {_money(realized, True)}",
        f"{BOX_S} PORTFOLIO",
        f"{BOX_L}Win rate : {_pct(wr)}  {_bar(wr)}  ({wins}W / {losses}L · {total})",
        _row("Fees paid", f"{fees:.4f}$"),
    ]


def render_open(pair, tf, side, entry, sl, tp, lev, conf,
                fee_usd, notional, stats) -> str:
    direction = "LONG" if side.upper() in ("BUY", "LONG") else "SHORT"
    lines = [
        f"<b>{BOX_T} {direction}  {pair}  {tf}</b>",
        _row("Entry", _price(entry)),
        _row("SL", _price(sl)),
        _row("TP", _price(tp)),
        _row("Size", f"{notional:.2f}$ notional  ({stats.get('size', 0):.4f})"),
        f"{BOX_L}Lev/Conf : {lev}x  ·  {conf:.2f}",
        _row("Fee", f"{fee_usd:.4f}$  (maker 0.02% open)"),
    ]
    lines += _footer_lines(stats)
    lines.append(f"{BOX_B}")
    return "<pre>" + "\n".join(lines) + "</pre>"


def render_close(pair, tf, side, exit_price, reason, pnl_r,
                  gross_usd, fee_usd, net_usd, open_fee, close_fee, stats) -> str:
    result = "WIN" if net_usd >= 0 else "LOSS"
    lines = [
        f"<b>{BOX_T} {result}  {pair}  {tf}  ({reason})</b>",
        _row("Exit", _price(exit_price)),
        _row("PnL", f"{pnl_r:+.2f}R"),
        _row("Gross", _money(gross_usd, True)),
        _row("Fee", f"{fee_usd:.4f}$  (open {open_fee:.4f} + close {close_fee:.4f})"),
        _row("Net", _money(net_usd, True)),
    ]
    lines += _footer_lines(stats)
    lines.append(f"{BOX_B}")
    return "<pre>" + "\n".join(lines) + "</pre>"
```

> The `<b>` only wraps the **title line** (bold header). If you prefer the section labels `BALANCE`/`PORTFOLIO` bold too, wrap them: `f"<b>{BOX_S} BALANCE</b>"`.

---

## 6. Integration — what each bot must pass

Both bots already share the `stats` contract (keys: `equity/balance`, `margin/used`,
`unrealized`, `realized`, `open_n`, `win_rate/wr`, `total_trades/n`, `total_wins/wins`,
`total_loses/losses`, `total_fee/fees_paid`). The footer reads exactly those keys, so:

- **`vaisravana`** — `notify_fill` / `notify_close` already compute `stats` via
  `paper_equity()`. Replace their body with a call to `render_open` / `render_close`,
  mapping `side` → LONG/SHORT and passing `notional = entry * size`, `open_fee`/`close_fee`
  from the fee model. Keep `html_escape` on `reason`/`pair` (reasons are plain words here,
  but escape defensively).
- **`vaisravana-wave`** — `notify_wave_open` / `notify_wave_close` currently emit emoji
  (`🌊 🟢 🔴`) and have **no win-rate / total-trades / fee footer** — this is the exact gap
  reported. Replace them with `render_open` / `render_close`. The wave caller must additionally
  supply the aggregate `stats` (win rate / totals / cumulative fees) from a `wave_log`
  summary at fill/close time — the same dict the main bot builds. Drop the emoji lines.

Caller sketch (wave open):
```python
stats = wave_log_summary(wallet)            # {equity, margin, unrealized, realized,
                                            #  open_n, win_rate, total_trades,
                                            #  total_wins, total_loses, total_fee}
notifier.send_message(render_open(
    wave.pair, wave.tf, wave.side,
    wave.entry_price, wave.sl_price, wave.tp_price,
    lev=getattr(wave, "leverage", 1), conf=wave.confidence,
    fee_usd=open_fee, notional=getattr(wave, "notional", 0.0),
    stats=stats))
```

**Verification (run after any change):** render both cards with a stub client and assert
`not _EMOJI.search(text)` AND `"LONG" in text` AND `"WIN" in text`. The repo already has
`test_notify_fill_and_close_shape` (asserts `LONG`/`WIN`, no ✅/❌) — extend it to assert the
`<pre>` wrapper and the `▓` bar presence so the new format is locked by CI.

---

## 7. Follow-up (other emoji still in the bots, outside trade alerts)

The no-emoji rule is global; these are *not* trade notifications but should be cleaned up
for consistency:
- `notify_decision` — `🟢 👁 ⏭`
- `notify_deploy` — `🚀`
- `notify_status` / `notify_status_30m` — `📊`
- `notify_health` — `🟢 🟡 🔴 📈 📉 📂 ✅ 💾 🎯 🛑 ⏱ 🏁 ❓`
- `notify_db_stats` — `🗄️`
- `build_wave_card` / `build_surf_card` — `🌊 🏄 🟢 🔴`

Recommend porting those to the same text/block-bar language in a follow-up pass.

---

## 8. Files

This spec: `/root/sera-bot-architecture/docs/02-notification-format.md`
Renderer module: `/root/sera-bot-architecture/src/trade_notif_render.py`
