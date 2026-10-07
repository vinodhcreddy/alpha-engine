Build me a live, auto-refreshing web dashboard for the Indian stock market (NSE F&O segment) that scans all stocks with active futures contracts and highlights intraday opportunities. Use Python (Flask backend) + a single self-contained HTML/JS frontend (no separate build step, no frontend framework).

## Universe
Scan the official NSE F&O stock-futures list (~213 symbols as of late 2025/2026 — e.g. RELIANCE, TCS, HDFCBANK, INFY, ICICIBANK, SBIN, plus smaller/newer additions like ADANIPOWER, COCHINSHIP, HYUNDAI, VMM, GVT&D, SWIGGY, etc.). Don't hand-guess this list — fetch the real current list from a reliable source (e.g. search for an up-to-date NSE F&O derivatives symbol list / GitHub mirror of NSE's fo_mktlots data) so it's accurate and not missing recent additions or including delisted/options-only names.

Use Yahoo Finance tickers (`SYMBOL.NS`) via the `yfinance` Python library to pull data — no broker API/keys needed. Note: Yahoo tickers sometimes escape special characters differently than NSE's raw symbol (e.g. `M&M`, `BAJAJ-AUTO`, `GVT&D`) — verify these resolve before trusting the list.

## Data & indicators
For each stock, pull 5-minute intraday candles (last ~5 days, for indicator warm-up) and compute:
- **VWAP** (volume-weighted average price, resets each trading day)
- **RSI(14)** on candles **resampled to 10-minute** bars (Yahoo has no native 10m interval — build it by resampling the 5m data: `resample("10min").agg({Open:first, High:max, Low:min, Close:last, Volume:sum})`)
- **Day change %** — ⚠️ compute this against **yesterday's actual closing price**, not today's first intraday candle. (Easy mistake: grabbing `today_candles.Close.iloc[0]` instead of the previous trading day's final close understates/distorts every number.)
- **VWAP gap %** (current price vs VWAP)
- **ATR**-based suggested stop-loss (entry − 1.5×ATR) and target (entry + 2×ATR)

Classify each stock into a bucket: OVERSOLD (RSI<20), NEAR_OVERSOLD (20-30), NEUTRAL (30-70), NEAR_OVERBOUGHT (70-80), OVERBOUGHT (RSI>80).

## Dashboard UI (dark theme, like a trading terminal)
- **Header**: live clock, market open/closed badge (NSE hours: Mon-Fri 9:15-15:30 IST), scan status indicator, a thin progress bar showing countdown to next data refresh
- **Stat cards row**: counts of stocks in each RSI bucket + today's single Top Gainer / Top Loser (symbol, %, price)
- **Market heatmap**: tile per stock, with a toggle between two coloring modes:
  1. **Price Change %** (default) — sort gainers→losers, green/red gradient scaled to the day's max move
  2. **RSI Buckets** — colored by oversold/overbought zone
- **Two side-by-side panels**: "Oversold Watch" (RSI<20) and "Overbought Watch" (RSI>80), each listing CMP, suggested SL/target, RSI trend arrow
- **Full sortable table**: every stock with all metrics
- Click any tile/row → popup with full detail breakdown
- **UI polls a `/api/state` JSON endpoint every 2 seconds** (cheap, just re-renders from cache)
- **Backend re-scans Yahoo Finance every ~90 seconds** in a background thread (don't hammer the API every 2s — only the UI render is that fast, not the actual data fetch)

## Practical gotchas to avoid (learned the hard way)
1. **Pick a safe port.** Chrome hard-blocks certain ports as "unsafe" (`ERR_UNSAFE_PORT`) — notably **5060** (SIP) and others like 6000, 389, etc. Use something like 8000/8050/8060/5050.
2. **Don't rely on `pip`/`python` being on PATH on Windows** — on a fresh machine these often fail with "not recognized" even after install, because Windows' App Execution Aliases intercept the bare `python` command. Find the real interpreter via `where.exe python` / checking `%LOCALAPPDATA%\Programs\Python\Python3*\python.exe`, or just reopen the terminal after install.
3. **Run the server as a detached/background process**, not tied to a single terminal session, so it survives — e.g. `Start-Process` on Windows with hidden window + redirected stdout/stderr log files. Save a `start_dashboard.ps1` / `stop_dashboard.ps1` pair (stop script should find the process by whatever is listening on the chosen port, not a hardcoded PID).
4. **This is equity/cash-market price data, not true futures contract data.** Real "futures heatmaps" (like stockmojo.in or research360.in) use the actual futures contract's price + **Open Interest** to classify Long Buildup / Short Buildup / Short Covering / Long Unwinding — that requires NSE's authenticated API (session cookies, anti-bot headers) or a paid data feed, which Yahoo Finance does not provide. Be upfront that this dashboard approximates with cash-market RSI/VWAP/price-change, which tracks closely for liquid stocks but isn't identical.
5. Handle missing/delisted tickers gracefully (skip with try/except) — 1-2 symbols out of ~213 may not resolve on Yahoo at any given time.

## Important disclaimer
This is a technical-analysis visualization tool only — not financial advice. Label it clearly as such in the UI footer.
