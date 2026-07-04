# Polymarket Spread Scalper Bot

A separate bot from the delta/momentum strategy bot, with its own logic — but
it has evolved to reuse the same *kind* of signal (price movement magnitude +
momentum agreement) that the delta bot validated, applied here as a
forward-looking predictor rather than a same-window filter.

---

## Read this before running `--live`

This bot's loss shape is **fundamentally different and riskier** than the
delta bot's:

- The delta bot only enters when the market price already strongly agrees
  with the signal (≥0.92), so a loss is bounded and historically rare.
- **This bot buys near a coin-flip price (~50¢) and profits from a modest
  spread (58-60¢), not from being right about the final outcome.** If the
  price never reaches that range and you're forced to exit late, **the loss
  can approach your full stake on that trade** — not a bounded, small
  percentage like the delta bot.
- **Real dry-run testing has already shown this failure mode happening** —
  in an early, small sample, roughly 1 in 4 windows ended in a force-exit
  loss, sometimes on BTC and ETH simultaneously (a real, observed
  correlation — losses are not necessarily independent across assets).
- The entry signal (below) is a genuine attempt to reduce *how often* this
  happens, informed by real validated data from the delta bot — but it does
  **not** change what happens *when* the signal is wrong. Keep testing with
  `--dry-run` before trusting this with `--live` capital.

---

## Strategy

1. Wake 10 seconds before each new 5-minute BTC/ETH Up/Down window opens.
2. As that window is about to close, check two things about the *ending*
   window's own price action (via Binance):
   - **Magnitude** — how far has price moved from that window's own open,
     as a percentage (`ENTRY_MIN_DELTA_PCT`, default `0.06%`)?
   - **Momentum** — does the last ~1-2 minutes of price action agree with
     that same direction?
   Both must agree, and the magnitude must clear the threshold, or **this
   window is skipped entirely** — no trade. This is meant to take fewer,
   more selective trades (rough target: 100-150/day instead of all 288),
   the same way the delta bot skips low-delta windows.
3. If the signal is confident, the instant the window opens, attempt to buy
   the chosen side (Up or Down) as close to $0.50 as possible (ceiling
   $0.52). If unfilled after 2 seconds, cancel.
4. If bought, watch for that side's price to reach $0.58-$0.60. Attempt to
   sell on every such opportunity — there may be more than one per window,
   and each is tracked and logged distinctly.
5. If no opportunity results in a sale by the time 60 seconds remain in the
   window, exit immediately at whatever price is available — no floor, no
   ceiling — to avoid holding into resolution uncertainty.
6. Sleep until 10 seconds before the next window, repeat.

Runs **BTC and ETH in parallel** on independent threads, each on its own
5-minute cycle.

**Note on the `ENTRY_MIN_DELTA_PCT` default (0.06%):** this value is
informed by real data — it's the exact threshold that, on the delta bot,
eliminated its one real loss across 107 trades while preserving all 68 wins
above that line. But that was answering a different question (filtering
entries within the same window) than this bot asks (predicting the *next*
window from the *previous* one's move). It's a reasonable starting point,
not a value validated for this specific use yet — treat it as a hypothesis
to test with your own dry-run data, not a settled number.

---

## Requirements

- Python 3.10+
- A Polymarket account with USDC on Polygon (for `--live` mode)
- No Binance account needed — momentum/magnitude data comes from Binance's
  public API (no auth required), same source the delta bot uses

---

## Installation

```bash
git clone https://github.com/kabeer-artistic-joy/PolyDrunk_Spreat_Bot.git
cd PolyDrunk_Spreat_Bot
python -m venv .venv
source .venv/bin/activate   # Windows: .venv\Scripts\activate
pip install -r requirements.txt
cp .env.example .env
# Edit .env — only required for --live mode
```

---

## Usage

```bash
# Dry run — real order book + real Binance data, no real orders, no funds at risk
python spread_bot.py --dry-run

# Live — real orders, real funds
python spread_bot.py --live --amount 2
```

Press **Ctrl+C** to stop and print the session summary.

---

## Configuration

Key constants are defined at the top of `spread_bot.py`:

| Constant | Default | Description |
|---|---|---|
| `ENTRY_MIN_DELTA_PCT` | `0.06` | Minimum % move (from the ending window's own open) required to trust its direction as a signal |
| `BUY_TARGET_PRICE` | `0.50` | Price we hope to pay |
| `BUY_CEILING_PRICE` | `0.52` | Max we're willing to pay (actual limit order price) |
| `BUY_TIMEOUT_SEC` | `2.0` | Cancel the buy if unfilled after this long |
| `SELL_TARGET_PRICE` | `0.60` | Price we hope to sell at |
| `SELL_FLOOR_PRICE` | `0.58` | Minimum acceptable sell price that triggers a sell attempt (not a ceiling — a better price will be taken if available) |
| `FORCE_EXIT_SECONDS_LEFT` | `60` | Exit at any price if still holding with this much time left |

---

## Order mechanics — exactly what type of order is used where

| Action | Order type | Why |
|---|---|---|
| Buy at window open | **Limit, GTC** | Needs to rest for up to 2 seconds in case liquidity appears a moment after open — a market/FAK order only checks the book once, right now, and won't wait |
| Sell at 58-60¢ opportunity | **Limit, FAK** | Instant — check the book now, fill what's available, cancel any leftover |
| Force-exit (last 60s) | **Market, FAK** | Deliberately no price limit — the goal here is guaranteed exit, not price protection |

---

## What `--dry-run` actually measures

Dry-run mode polls Polymarket's real, live public order book and Binance's
real price data throughout each window — it is not a simulation with assumed
prices. It records:

- Whether the entry signal fired at all, and the real delta% behind that
  decision (logged per-trade, so you can review after the fact whether
  `0.06%` is actually the right bar)
- Whether a real ask at or below the buy ceiling existed within the timeout
- Every distinct point where the real bid reached the sell floor, and
  whether there was enough real depth to fill your position size
- What would have happened at forced exit if no opportunity worked out

---

## Project Structure

```
spread-bot/
├── spread_bot.py        # Main bot
├── requirements.txt      # Python dependencies
├── .env.example          # Environment variable template
├── .gitignore
└── README.md
```

Trade-by-trade results are logged live to `trades_log.csv` in this folder,
including the signal reasoning (`target_side`, `signal_delta_pct`,
`signal_reason`) for every window — whether it was skipped, bought, sold, or
force-exited.

---

## Known uncertainties in this implementation

Documented honestly rather than glossed over:

- **`ENTRY_MIN_DELTA_PCT = 0.06%` has not been validated for this specific
  use case yet.** It's an informed starting point borrowed from the delta
  bot's real results, not a number proven correct here.
- **Order-fill detection in live mode** relies on `get_order()` returning
  `None` once an order leaves the open-orders index — inferred behavior, not
  something confirmed in Polymarket's documentation for this exact case.
- **The delayed-buy-fill price is not fully tracked.** If a buy order rests
  and fills sometime within the 2-second window (rather than matching
  instantly), the logged price falls back to the ceiling price, which may
  understate actual profit if the real fill was better. The immediate-match
  case (the common one) does correctly parse the real fill price.
- **Winning the race for the first 1-2 seconds of liquidity** depends on
  network latency and how fast market makers seed that opening liquidity —
  a real competitive factor this code cannot fully control for.
- **No hard-confirmed minimum order size exists in this codebase.** An
  earlier assumption of "5 shares minimum" had no verified source and has
  been removed — if Polymarket enforces some real minimum, you'll discover
  it directly via a live order response rather than a guess baked into the
  code.
- **Order-book polling volume during the sell-watch phase (~1 request/sec
  for up to ~4 minutes, across two assets continuously) has not been
  checked against Polymarket's actual public API rate limits.**

---

## Disclaimer

> This software is for educational and experimental purposes only. This is
> **not financial advice**. This strategy's loss shape can approach full
> stake per losing trade — materially riskier than a bounded-entry strategy.
> Early real testing has already shown a meaningful force-exit-loss rate in
> a small sample. Always start with `--dry-run` and accumulate real evidence
> before considering `--live`. Never risk money you cannot afford to lose.
