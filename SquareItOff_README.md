# Square it Off! — Pine Script v6 implementation

Implementation of **Preet Deep Singh (2015), _"Square it Off!: An intraday short-and-buy strategy"_** (SSRN 3920557).

| File | Type | Use it for |
|---|---|---|
| `SquareItOff_Indicator.pine` | Indicator (separate pane) | Reproducing the paper's **Table 2 (SIF by trend slice)** and **Table 4 (OR/TR correlations)** for any symbol. Works on daily charts. |
| `SquareItOff_Strategy.pine` | Strategy (overlay) | Backtesting the actual rule with fills, costs and filters. **Intraday charts only.** |

## The rule

> Sell (short) the index at the **open** of the regular session, buy it back — *square it off* — at the **close** of the same session. Never hold overnight.

## Paper → code mapping

| Paper | Code |
|---|---|
| `OR_t = ln(open_t / close_{t-1})` — overnight return | `orLog` / `orToday` |
| `TR_t = ln(close_t / open_t)` — trading-time return | `trLog` |
| `SIF = Σ (open_t − close_t)` — Sum of Intraday Falls | `sifPts[0]`, plotted as "SIF — all days" |
| Table 2 slices (overnight increase / decrease, 2-in-a-row, without 2-in-a-row) | `sliceMode` input; all 7 rows computed simultaneously in the indicator's panel |
| Table 4 `OR2TR = corr(OR_t, TR_t)`, `TR2OR = corr(TR_t, OR_{t+1})` | `f_pearson()` over a rolling window (default 252 days) |
| Rogalski (1984) successive vs non-successive days | `succMode` input in the strategy |
| "years where the open barely differs from the previous close" are dropped | `minOrPct` input |
| ~0.015% one-way discount brokerage | `commission_value = 0.015` (percent) |

**SIF is exactly the point-P&L of the strategy** — that identity is why the indicator can evaluate the rule on 20+ years of daily bars without a backtest.

## The paper's own recommendation

The edge is strongest when the overnight trend has **not** been positive two days running (increase-decrease, decrease-increase, or decrease-decrease). That is the `Days without 2 overnight increases` slice.

Also note the paper's own sign warning: **SIF is negative for NASDAQ and the S&P 500** over its 1995–2014 sample — the US regular session *rose*. The strategy has a `Long the open, sell the close (inverse test)` direction for exactly those markets.

## Execution caveat (read before trusting the backtest)

TradingView's broker emulator cannot fill one order at a bar's open and another at the same bar's close. The strategy uses `process_orders_on_close = true`, which means:

- the **square-off fills exactly at the close** of the square-off bar (the paper's hard requirement — square off no matter what the P&L);
- the **entry fills at the close of the first session bar**, not the opening print.

On a **1-minute chart** that is a one-minute lag and the results track the paper closely. On coarse timeframes the error is material. On daily/weekly charts the strategy refuses to trade and prints a warning — use the indicator instead, which measures the open-to-close move directly and has no such limitation.

## Other limitations carried over from the paper

- Gross of slippage and market impact; index level, not a tradeable basket.
- No capital is blocked overnight, but intraday margin still applies (`margin_short = 100` by default — lower it to model leverage).
- Losses are taken unconditionally at the close; the optional stop-loss / take-profit inputs are *extensions*, not part of the paper.
