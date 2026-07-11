# SPY/QQQ VWAP Pullback Long — Core Version

Companion documentation for `vwap_pullback_long_core.pine`.

This is a research/backtesting/paper-trading strategy only. It is long-only,
trades whole shares, has no pyramiding, and allows at most one open position
at a time.

## 1. The code

See [`vwap_pullback_long_core.pine`](./vwap_pullback_long_core.pine) in this
folder. Pine Script v6, single file, no external dependencies.

## 2. Plain-English entry conditions

A trade is only opened when **all** of the following are true, evaluated in
order as a state machine that advances at most one stage per confirmed bar:

1. **Bullish trend** (must be true to start a setup):
   close > VWAP, AND today's VWAP is higher than the prior bar's VWAP, AND
   the 9 EMA is above the 20 EMA.
2. **Impulse** (must occur after the trend is established):
   the bar's high reaches at least 0.15% above VWAP, OR at least 0.5×ATR(14)
   above VWAP (either condition, not both).
3. **Pullback** (must occur within 15 bars of the impulse bar):
   a bar's low comes within 0.10% of VWAP (above or below — a small wick
   below VWAP is allowed), AND that same bar's close is not more than 0.05%
   below VWAP.
4. **Confirmation** (the next bar, or a later bar, while still within the
   15-bar window from the impulse):
   the bar closes above its own open, closes above the *previous* bar's
   high, closes above VWAP, and its body is at least 50% of its high-low
   range.
5. **Risk and account filters**, checked only at the moment confirmation
   fires:
   - Symbol is SPY or QQQ (or "Allow Other Symbols" is enabled).
   - Chart is intraday (warns otherwise; will not trade a non-intraday chart).
   - Current time is inside the 9:45–11:15 AM America/New_York entry window.
   - Current bar time is inside the configured backtest date range.
   - Daily limits have not been hit yet (see below).
   - The computed stop is below entry, risk per share > 0, and stop distance
     ≤ 0.30% of entry price.
   - Position size (see sizing formula) rounds to at least 1 share.

If confirmation fires but any risk/account filter fails, no trade is taken
and the setup is discarded (it does **not** keep retrying on the same
impulse) — a fresh trend → impulse → pullback → confirmation sequence is
required to try again.

**Entry price:** the confirmation candle's own close (filled via
`process_orders_on_close = true`, so no next-bar-open slippage into the
signal — see order-fill assumptions in the script header).

**Stop:** the lowest low observed during the pullback/confirmation phase,
minus a buffer equal to the greater of 1 minimum tick or 0.01% of the entry
price.

**Target:** entry + 2.0 × (entry − stop), i.e. a fixed 2R target
(configurable).

**Exit:** whichever of the stop or target is touched first; the whole
position exits at once. No trailing stop, no breakeven logic. Any trade
still open at 11:15 AM ET is force-closed at market.

## 3. Assumptions made

- **"Materially below VWAP" (invalidation)** is implemented as a separate,
  configurable threshold (default 0.20%) distinct from the pullback's
  0.05% close-tolerance. A close between −0.05% and −0.20% below VWAP is
  simply not counted as a valid pullback bar (no state change); a close
  beyond −0.20% invalidates the whole setup.
- **The 15-bar pullback window** is measured from the impulse bar and is
  enforced both while waiting for the pullback *and* while waiting for
  confirmation (i.e., confirmation must also occur within 15 bars of the
  impulse). The spec only explicitly stated the 15-bar limit for the
  pullback itself; extending it to confirmation prevents a setup from
  waiting indefinitely.
- **"Lowest low of the pullback"** for stop placement is the running
  minimum low from the first qualifying pullback bar through the
  confirmation bar (inclusive), not just the single pullback bar's low.
- **A "losing trade"** for the daily loss-count rule is any completed trade
  with profit ≤ $0 (breakeven counts as a loss). This only matters if
  commission/slippage produce a small non-zero result at an otherwise flat
  exit, which shouldn't normally occur given this strategy's bracket exits.
- **"One completed trade reaches at least +2R"** is evaluated using the
  trade's actual realized profit divided by the dollar risk taken on that
  trade (risk-per-share × shares), compared to the Target R Multiple input.
  Because the target is itself fixed at that R multiple, in practice this
  flag is set whenever a trade exits via the target rather than the stop.
- **Trend must hold to *start* a setup** (state 0 → 1). Once impulse
  tracking begins (state 1+), the trend condition is not re-checked every
  bar — only the pullback/impulse-specific invalidation and expiration
  rules apply, per the spec's explicit invalidation criteria. If trend
  breaks while still waiting for the impulse (state 1), the setup resets.
- **Bankroll input vs. Initial Capital:** the "Simulated Bankroll ($)" input
  is informational/configurable per the spec, but TradingView's actual
  equity baseline (`initial_capital` in the `strategy()` declaration) is a
  fixed literal `5000` to satisfy Pine's requirement that `strategy()` be
  the first statement in the script (inputs cannot be declared before it,
  so the two can't be wired together automatically). If you change the
  Bankroll input, also update the "Initial Capital" field in the Strategy
  Tester's Properties tab to match.
- **Commission default is 0.0%**, reflecting that most US retail brokers no
  longer charge commissions on SPY/QQQ share trades; adjust to your actual
  broker's rate for a more conservative test.
- **EMA lengths (9/20) and ATR length (14)** are exposed as inputs even
  though the spec didn't explicitly ask for them to be configurable, purely
  for convenience — defaults match the spec exactly.

## 4. Adding the script to TradingView

1. Open [tradingview.com](https://www.tradingview.com), load an SPY or QQQ
   chart, and set the timeframe to 1 minute or 5 minutes.
2. Open the **Pine Editor** tab at the bottom of the screen.
3. Click **New** → **Blank script**, then delete the placeholder code.
4. Copy the entire contents of `vwap_pullback_long_core.pine` and paste it
   into the editor.
5. Click **Add to Chart** (or **Save**, then **Add to Chart**).
6. Open the strategy's **Settings (gear icon) → Inputs** tab to review or
   adjust any of the configurable thresholds, and the **Properties** tab to
   confirm Initial Capital, commission, and slippage match your intent.

## 5. Testing it on SPY and QQQ

1. Load SPY on a 1-minute chart, apply the strategy, and open the
   **Strategy Tester** panel (bottom of screen) → **Performance Summary** /
   **List of Trades** tabs.
2. Set a realistic date range using the "Backtest Start Date" / "Backtest
   End Date" inputs (or the Strategy Tester's own range control) — a few
   months of 1-minute data is a reasonable starting sample.
3. Repeat on a 5-minute SPY chart, then on QQQ at both timeframes, to
   compare behavior across symbols/timeframes.
4. In **List of Trades**, spot-check a handful of trades against the raw
   chart: confirm the entry marker sits on a confirmation candle that
   closed above its open/previous high/VWAP, the stop line sits just below
   the pullback low, and the target line is exactly 2× the entry-to-stop
   distance above entry.
5. Try setting "Allow Other Symbols" to false on a non-SPY/QQQ chart and
   confirm no trades are taken (a warning label should appear).
6. Try a non-intraday timeframe (e.g. daily) and confirm the orange warning
   background/label appears and no trades are taken.
7. Sanity-check the daily limits by finding a day with 2 completed trades,
   2 losing trades, a −$50 day, or a trade that hit target, and confirming
   no further entries occur that day in the List of Trades.

## 6. Known limitations

- **Force-close timing granularity:** the 11:15 AM ET force-close (and the
  end of the entry window generally) is evaluated on confirmed bars only.
  On a 5-minute chart this means the actual close can occur up to ~5
  minutes after 11:15, at the close of whichever bar contains/crosses that
  time — not exactly at 11:15:00.
- **No partial fills / no slippage-adjusted stop realism beyond the ticks
  input:** stops and targets are assumed to fill at their exact price once
  touched (aside from the configured slippage on market orders), which is
  optimistic for the stop side during fast moves (real gaps/slippage
  through a stop are not modeled beyond the flat ticks setting).
- **Single VWAP definition:** uses Pine's built-in session-anchored
  `ta.vwap`, which resets on the chart's own session boundaries. Extended
  hours data, if present on the chart, will affect the session VWAP anchor
  the same way it would on the raw chart.
- **State machine granularity:** advances at most one stage per bar (see
  Assumptions), so on the exact bar where multiple conditions become true
  simultaneously (e.g., trend flips true on the same bar an impulse-sized
  move happens), the additional stage(s) are picked up on the following
  bar rather than the same bar.
- **No spread/liquidity modeling, no partial position exits, no
  break-even/trailing logic** — by design, per the "core version" scope.
- **Daily-loss/₂-loss/₂R-win circuit breakers only stop *new* entries** —
  they do not affect an already-open trade, which will still run to its
  stop, target, or the 11:15 force-close.

## 7. Repainting and lookahead-bias checklist

- [x] All trading decisions are gated behind `barstate.isconfirmed` — no
      decision is made from an unconfirmed (still-forming) bar.
- [x] Entries fill at the confirmation bar's own close
      (`process_orders_on_close = true`), never at a price the strategy
      could not have known at decision time.
- [x] Stops and targets are filled by TradingView's broker emulator using
      each *subsequent* bar's high/low — never the bar that generated the
      signal, and never a future bar's data used to make the signal itself.
- [x] No `request.security()` / higher-timeframe calls anywhere in the
      script — eliminates the entire class of daily-high/low-before-it-
      happened lookahead bugs.
- [x] Session VWAP, EMAs, and ATR are all standard non-repainting Pine
      built-ins (`ta.vwap`, `ta.ema`, `ta.atr`) computed causally bar by
      bar.
- [x] All session/timezone comparisons use the bar's own `time` value
      converted with an explicit `"America/New_York"` timezone argument —
      never the chart's display timezone, and never a future bar's time.
- [x] `pyramiding = 0` and an explicit `strategy.position_size` check
      prevent more than one open position, and the state machine only
      permits scanning for a new setup once flat.
- [x] Daily counters and the setup state machine reset deterministically at
      the first confirmed bar of each new America/New_York calendar day.
- [x] `calc_on_every_tick = false` — historical and realtime bars are
      processed identically (once per confirmed bar), so backtest results
      are not an artifact of intrabar recalculation.
