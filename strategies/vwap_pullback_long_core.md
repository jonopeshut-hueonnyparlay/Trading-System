# Trading System — Intraday Long Framework (VWAP Pullback Long module)

Companion documentation for `vwap_pullback_long_core.pine`.

This is a research/backtesting/paper-trading strategy only. It is long-only,
trades whole shares, has no pyramiding, and allows at most one open position
at a time **across the whole script**.

## 0. Architecture

The script is a small multi-strategy **framework** with one active **module**,
VWAP Pullback Long. This is deliberate: it lets future strategies be added as
independent modules without touching or risking the VWAP Pullback logic.

- **Shared framework pieces** (top of the file, usable by any module):
  - `StrategyState` — a Pine v6 user-defined `type`. Each module owns exactly
    one instance of it, holding that module's setup-machine stage, impulse/
    pullback bar tracking, trade bracket levels, and daily counters. Because
    each module's state lives in its own object, two modules can never read
    or overwrite each other's variables.
  - `activeStrategyId` — a single shared marker recording which module's
    setup id currently owns the one open position the whole account is
    allowed to hold. Every module checks this before managing a position.
  - Utility functions: `isAllowedSymbol`, `inBacktestRange`,
    `canOpenNewPosition`, `calcPositionSize`, `minStopBuffer`,
    `etMinutesOfDay`, `etDayId`.
  - Global settings: **Allow Other Symbols** and the backtest date-range
    inputs — these are account/broker-level concerns shared by any module.
    Commission and slippage are set as constants in the `strategy()`
    declaration itself rather than as inputs (Pine requires these two
    specific arguments to be compile-time constants) — see Assumptions.
- **The VWAP Pullback Long module** (setup id `"VWAPPullbackLong"`) owns:
  - its own `Enable VWAP Pullback Long` input (default `true`) and all of its
    own thresholds, each grouped under `VWAP Pullback Long: ...` in Settings;
  - its own indicator calculations (VWAP / 9-20 EMA / ATR);
  - separate functions for trend detection, impulse/pullback setup detection,
    invalidation, confirmation, stop calculation, target calculation, the
    daily/account trading gate, trade execution, setup reset, daily reset,
    closed-trade accounting (filtered to only its own trades — see below),
    exit/session management, and a `runVwapPullbackLong(...)` driver that
    ties the pipeline together for one bar;
  - its own chart markers and plots.
  - **Closed-trade accounting is filtered by setup id**: `strategy.closedtrades`
    is a single list shared by the whole script, so the module's daily-stats
    function only counts a closed trade toward its own counters when
    `strategy.closedtrades.entry_id(i) == state.id`. This means a second
    module's trades can never be miscounted into VWAP Pullback Long's daily
    limits, and vice versa, even while both are enabled and trading.

No second strategy is implemented yet. See the `FUTURE STRATEGY MODULES`
comment block near the bottom of the script for the exact steps to add one
(unique setup id, its own `StrategyState` instance, its own inputs/functions/
driver/visuals) without modifying any VWAP Pullback Long code.

### Pre-test audit findings (fixed)

A final audit before testing found and fixed three definite Pine-specific
bugs. No entry, stop, target, sizing, session, or daily-risk rule changed —
these are state-management/runtime-correctness fixes only:

1. **Stateful `ta.*` calls were being made conditionally.** `ta.falling()`
   and `ta.crossunder()` (used by the invalidation check) were called from
   inside a function that only ran while `setupStage` was 2 or 3. Pine's
   history-dependent built-ins key their internal "previous value" to how
   many times *that call site* has executed, not to the chart's actual bar
   sequence — calling them conditionally means "previous bar" silently
   means "the last bar this branch happened to run," which would have
   produced wrong falling/crossunder signals whenever a setup wasn't
   currently being tracked. Fixed by computing both unconditionally, once
   per bar, in the Indicators section, and only *consuming* the resulting
   booleans conditionally inside the state machine.
2. **Post-window setup-stage flicker.** The session-end guard reset the
   setup stage to 0 whenever `forceCloseTime` was true, but the state
   machine's scanning chain ran immediately afterward in the same bar pass
   and could re-advance stage 0 → 1 right away if trend conditions were
   met — repeating every bar for the rest of the day. No trade could ever
   result (the entry-window check always fails after the window closes),
   but the setup stage would misleadingly flicker between 0 and 1 all
   afternoon. Fixed by also gating the scanning chain on `not
   forceCloseTime`, so scanning genuinely stops for the day once the
   window closes.
3. **New-day reset could desync an open position.** The daily counter
   reset called the generic setup-reset unconditionally, without the same
   "don't touch an active trade" guard the session-end reset already had.
   In the practically-impossible case a position were still open across an
   Eastern Time day boundary (the force-close mechanism makes this
   essentially unreachable in normal use), the setup stage would have been
   wiped to 0 while a real position was still open. Fixed by applying the
   same `setupStage != 4` guard used elsewhere.

## 1. The code

See [`vwap_pullback_long_core.pine`](./vwap_pullback_long_core.pine) in this
folder. Pine Script v6, single file, no external dependencies.

## 2. Plain-English entry conditions

A trade is only opened when **all** of the following are true, evaluated in
order as a state machine that advances at most one stage per confirmed bar:

1. **Bullish trend — required to start a setup** (stage 0 → 1, and must keep
   holding while waiting for the impulse in stage 1):
   close > VWAP, AND VWAP is higher than the prior bar's VWAP, AND the 9 EMA
   is above the 20 EMA. If any of these breaks while still waiting for the
   impulse, the setup resets back to stage 0.
2. **Impulse** (stage 1 → 2, once trend is established):
   the bar's high reaches at least 0.15% above VWAP, OR at least 0.5×ATR(14)
   above VWAP (either condition, not both).
3. **Pullback** (stage 2 → 3; must occur within 15 bars of the impulse bar):
   a bar's low comes within 0.10% of VWAP (above or below — a small wick
   below VWAP is allowed), AND that same bar's close is not more than 0.05%
   below VWAP.
4. **Confirmation** (stage 3 → trade; must occur within a *separate*,
   independently configurable window — default 5 bars — measured from the
   qualifying pullback bar, not from the impulse):
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
   - The account is flat (no other module currently holds the single open
     position slot).

If confirmation fires but any risk/account filter fails, no trade is taken
and the setup is discarded (it does **not** keep retrying on the same
impulse) — a fresh trend → impulse → pullback → confirmation sequence is
required to try again.

### Once the impulse begins: relaxed trend, explicit invalidation

Between the impulse and the entry (stages 2 and 3), the strategy does **not**
require every original trend condition (close > VWAP, VWAP rising, 9 EMA >
20 EMA) to keep holding on every single bar — a normal pullback necessarily
puts price back near or briefly below VWAP, which would otherwise falsely
break the strict trend test. Instead, the in-progress setup is explicitly
**invalidated and reset** the moment any ONE of these three conditions is
true:

- **VWAP is clearly falling** — a configurable consecutive down-bar streak
  in VWAP itself (`ta.falling(vwap, N)`, default `N = 2` bars).
- **The 9 EMA crosses below the 20 EMA** (`ta.crossunder`).
- **Price closes below the configurable "material VWAP break" threshold**
  (default 0.20% below VWAP — distinct from the pullback's own 0.05%
  close-tolerance, which only governs whether a given bar itself qualifies
  as a valid pullback bar, not whether the whole setup is invalidated).

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

- **"Materially below VWAP" (invalidation)** uses its own configurable
  threshold (default 0.20%), distinct from the pullback's 0.05%
  close-tolerance (see above).
- **The two expiration windows are independent and non-overlapping in
  scope**: the pullback window (default 15 bars) is measured only from the
  impulse bar and governs stage 2; the confirmation window (default 5 bars)
  is measured only from the qualifying pullback bar and governs stage 3. A
  setup that finds its pullback on bar 14 of 15 still gets the full 5-bar
  confirmation window starting from that bar, not a truncated remainder of
  the pullback window.
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
- **Bankroll input vs. Initial Capital:** the "Simulated Bankroll ($)" input
  is informational/configurable per the spec, but TradingView's actual
  equity baseline (`initial_capital` in the `strategy()` declaration) is a
  fixed literal `5000` to satisfy Pine's requirement that `strategy()` be
  the first statement in the script (inputs cannot be declared before it,
  so the two can't be wired together automatically). If you change the
  Bankroll input, also update the "Initial Capital" field in the Strategy
  Tester's Properties tab to match.
- **Commission and slippage are not Pine inputs.** `strategy()`'s
  `commission_value` and `slippage` arguments must be compile-time
  constants in Pine — passing an `input.float()`/`input.int()` call there
  is a compile error (`CE10123`), unlike most other `strategy()`
  arguments. They're set as plain constants instead: 0.0% commission
  (reflecting that most US retail brokers no longer charge commissions on
  SPY/QQQ share trades) and 1 tick of slippage. To change them, either
  edit the two literals directly in the `strategy()` declaration near the
  top of the script, or override them per-run in the Strategy Tester's
  **Properties** tab, which doesn't require editing code.
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
   adjust any of the configurable thresholds — the VWAP Pullback Long
   inputs are grouped separately from the shared Global/Backtest settings —
   and the **Properties** tab to confirm Initial Capital, commission, and
   slippage match your intent.

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
8. Toggle **Enable VWAP Pullback Long** to false mid-backtest-range and
   confirm no new setups are scanned, while any trade already open at the
   moment you'd disable it (in a live/paper context) would still be
   managed to its stop/target/force-close.

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
- **State machine granularity:** advances at most one stage per bar, so on
  the exact bar where multiple conditions become true simultaneously (e.g.,
  trend flips true on the same bar an impulse-sized move happens), the
  additional stage(s) are picked up on the following bar rather than the
  same bar.
- **No spread/liquidity modeling, no partial position exits, no
  break-even/trailing logic** — by design, per the "core version" scope.
- **Daily-loss/2-loss/2R-win circuit breakers only stop *new* entries** —
  they do not affect an already-open trade, which will still run to its
  stop, target, or the 11:15 force-close.
- **Single open-position slot is shared account-wide.** Right now only one
  module exists, so this has no visible effect, but once a second module is
  added, only one of the two can hold a position at any given moment —
  whichever fires its entry first. This is a deliberate, documented
  design choice (see Architecture), not a bug.

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
- [x] Session VWAP, EMAs, ATR, and the invalidation checks (`ta.falling`,
      `ta.crossunder`) are all standard non-repainting Pine built-ins, and
      all five are called unconditionally once per bar (see Pre-test audit
      findings above) so their internal history always lines up with actual
      chart bars rather than with how often a conditional branch happened
      to run.
- [x] All session/timezone comparisons use the bar's own `time` value
      converted with an explicit `"America/New_York"` timezone argument —
      never the chart's display timezone, and never a future bar's time.
- [x] `pyramiding = 0` and an explicit `strategy.position_size` check
      (`canOpenNewPosition()`) prevent more than one open position across
      the whole script, and the state machine only permits scanning for a
      new setup once flat.
- [x] Daily counters and the setup state machine reset deterministically at
      the first confirmed bar of each new America/New_York calendar day.
- [x] Closed-trade accounting is filtered by each module's own setup id
      (`strategy.closedtrades.entry_id`), so daily counters can never be
      corrupted by another module's trades.
- [x] `calc_on_every_tick = false` — historical and realtime bars are
      processed identically (once per confirmed bar), so backtest results
      are not an artifact of intrabar recalculation.

## 8. Debug mode (temporary, for testing)

The script includes an optional diagnostic layer, off by default, for
verifying the state machine behaves as documented before you trust backtest
results. It is purely read-only — it never influences any trading decision —
and is safe to leave in place (or delete) once you're done testing.

**To enable it:** open the strategy's **Settings → Inputs** tab, find
**Enable Debug Visuals (temporary, for testing)** under the "VWAP Pullback
Long: Debug" group, and turn it on.

**What you get:**

- Five extra series visible only in the **Data Window** (not drawn on the
  chart itself, so they don't clutter the price pane), scrubbable bar by
  bar through history:
  - **Debug: State Number** — the raw `setupStage` value (0–4) for that bar.
  - **Debug: Impulse Bar Index** — the `bar_index` of the current setup's
    qualifying impulse bar, or blank if no impulse is being tracked.
  - **Debug: Pullback Bar Index** — the `bar_index` of the current setup's
    qualifying pullback bar, or blank if none yet.
  - **Debug: Bars Since Impulse** / **Debug: Bars Since Pullback** — live
    counters against the two expiration windows (15 and 5 bars by default);
    use these to confirm a setup expires on the correct bar and not one off.
- A small **"VWAP Pullback Long - DEBUG"** table in the bottom-right corner
  of the chart, refreshed on the most recent bar, showing:
  - **State** — the stage number with a human-readable label (e.g. "2:
    Waiting Pullback").
  - **Impulse Bar Index** / **Pullback Bar Index** / **Bars Since Impulse**
    / **Bars Since Pullback** — the same values as the Data Window series,
    for a quick at-a-glance snapshot without opening the Data Window.
  - **Last Invalidation Reason** — which of the three post-impulse
    invalidation conditions (VWAP Falling / EMA Cross Down / Material VWAP
    Break) most recently killed a setup. This persists after the reset (it
    is not cleared back to "N/A" until the *next* invalidation), specifically
    so you can see why a setup that just disappeared was killed, since by
    the time the table redraws the stage has already gone back to 0.
  - **Daily Lockout Reason** — which daily circuit breaker, if any, is
    currently blocking new entries for the rest of the trading day ("Big
    Win Reached", "Max Losing Trades", "Max Daily Loss", "Max Trades Per
    Day", or "None").

**Suggested use:** step through a handful of days on a 1-minute SPY/QQQ
chart with debug visuals on, and confirm: the state number advances exactly
one stage at a time and never skips; impulse/pullback bar indices only
change on bars where the corresponding chart marker appears; the bars-since
counters reset to blank immediately after an entry, invalidation, or
expiration; and the daily lockout reason appears exactly on the day/bar you'd
expect given the trade log. Turn the input back off (or leave it off) for
actual backtesting runs — it adds a small amount of per-bar overhead and
visual clutter you don't need once you trust the logic.
