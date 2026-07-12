# Trading System Roadmap

Path from "one backtested Pine Script strategy" to a reliable, monitored,
algorithmic trading system. Written as sequential phases — each phase should
be genuinely done (not just started) before moving to the next. Skipping
ahead is the most common way these projects lose money.

Current position: **Phase 0 complete, Phase 1 in progress.**

---

## Phase 0 — Foundation (done)

- [x] Core strategy logic built: SPY/QQQ VWAP Pullback Long (Pine Script v6,
      `strategies/vwap_pullback_long_core.pine`).
- [x] Modular framework in place — a shared `StrategyState` type, shared
      utilities, and one isolated module — so additional strategies can be
      added later without touching this one (`FUTURE STRATEGY MODULES`
      section in the script).
- [x] Compiles clean, no repainting / no lookahead by design (confirmed-bar
      gating, no `request.security`, realistic order-fill assumptions
      documented in the script header and `strategies/vwap_pullback_long_core.md`).
- [x] Optional debug-visuals layer for verifying state-machine correctness
      bar by bar.
- [x] First backtest run: 18 trades, 83% win rate, profit factor 5.5, 0.24%
      max drawdown over ~3 months of SPY.

**What Phase 0 does *not* mean:** 18 trades on one symbol over one window is
not a validated edge. It means the logic isn't obviously broken.

---

## Phase 1 — Validation (current phase)

Goal: earn the right to trust the backtest numbers before building anything
else on top of them.

- [ ] Manually spot-check 5+ individual trades against the debug panel and
      chart markers — confirm entries/stops/targets are mechanically correct.
- [ ] Extend the backtest date range as far as your TradingView plan's
      intraday data allows.
- [ ] Re-run on QQQ, same settings — an edge should show on both symbols.
- [ ] Re-run on both 1-minute and 5-minute — should hold on both, not just one.
- [ ] Set commission/slippage in Properties to match a realistic broker,
      re-run, confirm the edge survives realistic costs.
- [ ] Stress-test edge cases: a session with a gap/holiday, a full trading
      day with debug on to confirm force-close and daily resets fire
      correctly.
- [ ] Forward paper-test: leave it running live (untouched) on a chart for
      several weeks of real, out-of-sample time. Compare forward results to
      the backtest — large divergence means overfitting or regime change.

**Exit criteria for Phase 1:** the edge holds up (not necessarily identical
stats, but the same *character* — positive expectancy, small drawdowns) across
symbol, timeframe, cost assumptions, and forward time. If it doesn't, go back
to design, not to tweaking thresholds against the same 3 months of data.

**Do not skip ahead from here without meeting the exit criteria.**

---

## Phase 2 — Observability inside TradingView

Goal: make the strategy's decisions externally visible, not just internal to
the Pine script.

- [ ] Add `alert()` / `alertcondition()` calls at each decision point: entry
      filled, stop hit, target hit, force-close. (Deliberately not in the
      script yet — added once Phase 1 passes, not before, so alerts aren't
      built around logic that's still changing.)
- [ ] Design the alert payload (JSON) with everything a downstream system
      would need: symbol, side, quantity, entry/stop/target price, setup id
      (`VWAPPullbackLong`), timestamp. The setup id matters even more here —
      it's how a downstream bridge will eventually tell which module/strategy
      an alert came from once more than one exists.
- [ ] Confirm your TradingView plan supports real-time alert delivery on
      strategies (this is a separate limit from the indicator-count cap you
      hit earlier — check before assuming it's covered).

---

## Phase 3 — Execution bridge

Goal: turn an alert into a real (paper, first) order.

- [ ] Choose a broker with API access that supports the order types this
      strategy needs: market entry + bracket (stop + limit) exit. Common
      choices for this kind of retail algo work: Alpaca, Interactive Brokers.
- [ ] Choose how alerts reach the broker:
  - **No-code**: a bridge service (e.g. TradersPost or similar) that
    connects TradingView webhooks directly to a broker.
  - **Custom**: a small server you own that receives the webhook and calls
    the broker API directly — more work, more control, no third-party
    dependency.
- [ ] Handle idempotency: a duplicate/retried webhook must not place a
      duplicate order. This is a real failure mode, not a hypothetical one.
- [ ] Map the strategy's daily circuit breakers (max trades/day, max losing
      trades, max daily loss, big-win lockout) into the bridge/broker layer
      too — don't rely on Pine's internal state alone once real order
      rejections and partial fills are possible.

---

## Phase 4 — Broker paper trading (second, more realistic test)

Goal: test the *pipeline*, not just the strategy logic.

- [ ] Run the full alert → bridge → broker pipeline against your broker's own
      paper-trading account (not TradingView's backtester — a different,
      more realistic simulation of real order flow and timing).
- [ ] Compare actual fills/timing/slippage against what the Pine script
      assumed. Adjust the commission/slippage inputs to match reality, and
      re-run Phase 1's validation with the corrected assumptions.
- [ ] Monitor for pipeline failures specifically: missed alerts, webhook
      downtime, malformed payloads, order rejections. Log everything.

---

## Phase 5 — Reliability & risk-control layer

This is the "boring" layer, and it's the one that actually determines
whether this is safe to run unattended. Do not shortcut it.

- [ ] Structured logging of every alert received, every order sent, every
      fill/rejection, with timestamps.
- [ ] Health monitoring on the bridge/server itself — you need to know
      immediately if it goes down during market hours, not find out after
      the fact.
- [ ] A manual kill switch: one action that flattens any open position and
      stops the pipeline from taking new signals, reachable from your phone.
- [ ] Daily reconciliation: compare what the broker's account actually shows
      (positions, today's trades, today's P&L) against what the system
      *thinks* happened. Alert on any mismatch.
- [ ] Alerting to yourself (push/email/SMS) on: pipeline errors, an
      unexpected position, a daily circuit breaker tripping, the kill switch
      firing.

**Exit criteria for Phase 5:** you're comfortable being asleep or away from
a screen while this runs, because you trust it will either work correctly or
fail safely (flat, not silently broken) and page you either way.

---

## Phase 6 — Small real-money pilot

- [ ] **Check Pattern Day Trader (PDT) rules before starting.** In the US,
      a margin account under $25,000 equity is restricted to 3 day trades
      per rolling 5 business days. This strategy's "max 2 trades per day"
      cap can easily exceed that limit across a week — with a ~$5,000
      account this is a real, concrete blocker, not a formality. Resolve it
      (larger account, cash account with T+1 settlement awareness, a
      broker/account structure that fits) before sending a single live
      order — this needs to happen before Phase 6, not be discovered during it.
- [ ] Start with real capital meaningfully smaller than you're ultimately
      willing to risk, specifically so early pipeline mistakes are cheap.
- [ ] Define kill criteria *in writing, before starting*: e.g. "stop if
      realized slippage exceeds assumption by 3x," "stop if any duplicate
      order occurs," "stop if daily loss limit is breached without the
      system halting itself." Decide these when you're calm, not mid-drawdown.
- [ ] Run live and forward-paper in parallel for comparison.

---

## Phase 7 — Scale and expand

- [ ] Increase capital gradually, tied to a track record, not to impatience.
- [ ] Add further strategy modules using the existing framework pattern —
      each gets its own unique setup id, its own `Enable` toggle, its own
      state object, with the shared account-level position slot and daily
      accounting already designed to keep modules from interfering with
      each other.
- [ ] Set a recurring re-validation cadence (e.g. monthly) — re-run Phase 1's
      checks against fresh data to catch when an edge decays or the market
      regime shifts. An edge that worked last year isn't guaranteed to keep
      working.

---

## Strategy research track (runs in parallel, starting now)

Finding and defining new strategy *candidates* doesn't require live capital
or a finished pipeline, so this track can run alongside Phase 1-5 instead of
waiting for them. What it feeds into is Phase 7 — a candidate only actually
gets built as a module, and only actually goes live, once it clears the same
gate the core strategy is going through right now. **No new module gets
implemented until one of these is picked and confirmed.**

### What makes a candidate worth researching
- **Mechanical, not discretionary** — expressible as explicit if/then rules,
  the same way VWAP Pullback Long is. If it can't be written as precise
  conditions, it can't be coded without repainting or ambiguity.
- **Diversifying, not duplicating** — ideally trades a different time-of-day,
  a different market condition (trend vs. range), or the opposite direction,
  so it doesn't just win and lose on the same days as the existing module for
  the same reason. A second copy of the same edge isn't diversification.
- **Frequent enough to validate** — needs enough historical occurrences to
  get a meaningful sample size in a reasonable testing window (the core
  strategy's ~1 trade/3-4 days is already on the sparse side).

### Full backlog: it's ~20-25 core edges, not ~400 strategies

A full brain-dump of candidate strategy names was collected (preserved in
full further down). Read literally it's 400+ items, but almost all of that
is the same underlying handful of edges repeated across three axes:

- **Instrument**: the identical pattern traded on SPY vs. QQQ vs. ES vs. NQ
  vs. gold vs. a single stock is one edge, not five.
- **Direction**: a "long" version and its "short"/"put" mirror are one edge
  expressed two ways, not two edges.
- **Expression vehicle**: the same directional or volatility view expressed
  as stock, futures, or one of a dozen options structures is one view, not
  a dozen views.

Collapsing along those three axes, the backlog is really this set of core
edges:

**Directional / continuation (equities, ETFs, futures)**
VWAP pullback/reclaim/rejection, opening-range breakout/breakdown/retest/
failure (incl. Initial Balance variants), moving-average pullback, trend-day
continuation, flag/pennant/triangle/wedge continuation, higher-low /
lower-high continuation, level breakout-and-retest (HOD/LOD, prior day
high/low, premarket high/low, horizontal S/R, overnight high/low, Globex
high/low), gap-and-go.

**Mean reversion / fade (equities, ETFs, futures)**
VWAP mean reversion, anchored VWAP reversion, Bollinger Band reversion, RSI
overbought/oversold reversion, extension fade, failed-breakout/failed-
breakdown fade-and-reclaim, exhaustion/climactic reversal, stop-hunt /
liquidity-sweep reversal, bull-trap fade / bear-trap reclaim, gap fade/fill/
hold/reclaim, prior-close and range-midpoint reversion.

**Relative value / momentum (needs multiple symbols)**
Relative strength/weakness (single name or sector vs. index), SPY-vs-QQQ or
ES-vs-NQ divergence, sector/breadth confirmation (TICK, ADD, VOLD), pairs
trading, cointegration, index/ETF/futures arbitrage, merger/convertible
arbitrage, cash-and-carry / basis trade.

**Market/volume profile (needs profile charting)**
Value area breakout/rejection/rotation, point-of-control reversion/
rejection, high/low-volume node reversion/breakout, single prints, poor
high/low reversal, balanced-day vs. trend-day rotation.

**Order flow / tape (needs Level 2 / DOM / footprint data)**
Absorption, iceberg detection, cumulative delta divergence, footprint/
stacked imbalance, large-lot tracking, DOM and time-and-sales scalping.

**Futures-specific structure**
Per-product directional (ES/NQ/RTY/YM and micros, metals, energy, grains,
livestock, rates, FX, crypto, VIX), intramarket/intermarket/intercommodity
spreads (crack, crush, spark, TED, yield-curve, calendar), contango/
backwardation, session-based (Globex/London/NY-open/Power-Hour), and macro
event-driven (FOMC, CPI, NFP, inventory reports).

**Options structures (full spectrum)**
Directional (long call/put, stock+option combos), income (covered call,
cash-secured put, the wheel), protective (protective put/call, collars),
vertical spreads (bull/bear, debit/credit), straddles/strangles/guts,
calendars and diagonals (incl. poor man's covered call), butterflies and
condors (incl. iron variants and broken-wing), ratio spreads/backspreads,
synthetics and parity trades (conversion, reversal, box spread), LEAPS
strategies, position-rolling techniques, volatility/greeks trades (vega,
theta, delta-neutral, gamma scalping, skew/term-structure, IV rank), and the
earnings-specific and 0DTE versions of most of the above.

### Practical note on tooling (not a restriction — just what tests where)
- Directional/mean-reversion/level/profile edges on **equities, ETFs, and
  futures** (continuous contracts like `ES1!`, `NQ1!`) can all be prototyped
  in Pine Script the same way the current module was, using
  `request.security()` for the relative-value ones that need a second symbol.
- **Options structures** generally can't be realistically backtested inside
  a Pine `strategy()` — Pine has no native options-chain, greeks, or IV
  surface data. Researching these means either paper-tracking them manually,
  or using an options-specific backtesting platform/broker analytics tool
  outside this repo.
- **Order-flow/DOM/footprint** edges need Level 2 or tick-level data Pine's
  standard feeds don't expose — these need a different platform (e.g. a DOM/
  footprint tool) to research at all, not just to trade.
- None of this changes what gets *traded* — it's just about which edges this
  repo's current toolchain can actually validate versus which need a
  different tool to even research properly.

<details>
<summary>Full raw list as originally provided (unedited, for reference)</summary>

VWAP Pullback, VWAP Reclaim, VWAP Rejection, VWAP Mean Reversion, Anchored
VWAP Strategy, Opening Range Breakout, Opening Range Breakdown, Opening
Range Retest, Opening Range Failure, Trend Pullback, Moving Average
Pullback, Breakout Pullback, Higher Low Continuation, Lower High
Continuation, Horizontal Level Breakout, High-of-Day Breakout, Low-of-Day
Breakdown, Pre-Market High Breakout, Pre-Market Low Breakdown, Range
Breakout, Consolidation Breakout, Triangle/Pennant Breakout, Resistance
Break and Retest, Support Break and Retest, Previous Day High Retest,
Previous Day Low Retest, Previous Close Reclaim/Reject, Support Bounce,
Resistance Rejection, Range Trading, Previous Day Level Bounce, Premarket
Level Bounce, VWAP Reversion, Bollinger Band Reversion, RSI Overbought/
Oversold Reversion, Gap Fill, Extension Fade, Failed Breakout Fade, Gap and
Go, Gap Fade, Gap Hold, Gap Reclaim, Relative Strength Long, Relative
Weakness Short, Sector Relative Strength, SPY vs QQQ Divergence, Pair
Relative Value, Intraday Trend Following, Moving Average Trend, Higher High
/ Higher Low Trend, Lower Low / Lower High Trend, Trend Day Strategy, High
Relative Volume Momentum, News Momentum, Earnings Momentum, Analyst
Upgrade/Downgrade Momentum, Sector Momentum, Stop Hunt Reversal, Liquidity
Sweep Long, Liquidity Sweep Short, Failed Breakdown Reclaim, Failed Breakout
Rejection, Level 2 Scalping, Time & Sales Momentum, Absorption, Iceberg
Detection, Cumulative Delta Divergence, Breadth Confirmation, Tick Index
Strategy, VOLD / ADD Confirmation, Sector Confirmation, Pairs Trading, ETF
Component Arbitrage, Index Futures vs ETF Arbitrage, Mean Reversion Basket,
Cointegration Strategy, Latency Arbitrage, Merger Arbitrage, Convertible
Arbitrage, Options Put-Call Parity Arbitrage, Crypto Exchange Arbitrage,
Long Call, Long Put, Buying Index Calls, Buying Index Puts, Long Stock +
Long Put, Short Stock + Long Call, Covered Call, Covered Put, Cash-Secured
Put, Cash-Backed Call, Naked Call, Naked Put, Protective Put, Protective
Call, Protective Collar, Collar, Zero-Cost Collar, Ratio Collar, Put Spread
Collar, Call Spread Collar, Bull Call Spread, Bull Put Spread, Bear Call
Spread, Bear Put Spread, Debit Call Spread, Debit Put Spread, Credit Call
Spread, Credit Put Spread, Vertical Call Spread, Vertical Put Spread, Long
Vertical Spread, Short Vertical Spread, At-The-Money Vertical, In-The-Money
Vertical, Out-Of-The-Money Vertical, Broken-Wing Vertical, Long Straddle,
Short Straddle, Long Strangle, Short Strangle, Covered Strangle, Covered
Combination, Guts, Short Guts, Long Guts, Long Call Calendar Spread, Long
Put Calendar Spread, Short Call Calendar Spread, Short Put Calendar Spread,
Double Calendar Spread, Calendar Spread, Calendar Straddle, Calendar
Strangle, Weekly Calendar Spread, Monthly Calendar Spread, Earnings Calendar
Spread, Reverse Calendar Spread, Call Diagonal Spread, Put Diagonal Spread,
Double Diagonal Spread, Diagonal Spread, Poor Man's Covered Call, Poor Man's
Covered Put, Diagonal Covered Call, Diagonal Put Spread, Reverse Diagonal
Spread, Long Call Butterfly, Long Put Butterfly, Short Call Butterfly, Short
Put Butterfly, Long Iron Butterfly, Short Iron Butterfly, Iron Butterfly,
Broken-Wing Butterfly, Call Broken-Wing Butterfly, Put Broken-Wing
Butterfly, Unbalanced Butterfly, Skip-Strike Butterfly, Christmas Tree
Butterfly, Ratio Butterfly, Long Call Condor, Long Put Condor, Short Call
Condor, Short Put Condor, Long Iron Condor, Short Iron Condor, Iron Condor,
Narrow Iron Condor, Wide Iron Condor, Unbalanced Iron Condor, Broken-Wing
Condor, Reverse Iron Condor, Call Ratio Spread, Put Ratio Spread, Call Ratio
Backspread, Put Ratio Backspread, 1x2 Call Ratio Spread, 1x2 Put Ratio
Spread, 1x3 Call Ratio Spread, 1x3 Put Ratio Spread, Covered Ratio Spread,
Uncovered Ratio Spread, Ratio Spread, Backspread, Frontspread, Synthetic
Long Stock, Synthetic Short Stock, Synthetic Long Call, Synthetic Short
Call, Synthetic Long Put, Synthetic Short Put, Synthetic Covered Call,
Synthetic Cash-Secured Put, Conversion, Reversal, Box Spread, Long Box
Spread, Short Box Spread, Jelly Roll, Put-Call Parity Arbitrage, LEAPS Long
Call, LEAPS Long Put, LEAPS Covered Call, LEAPS Poor Man's Covered Call,
LEAPS Diagonal, LEAPS Protective Put, Stock Replacement With LEAPS, Covered
Call Roll, Cash-Secured Put Roll, Vertical Spread Roll, Calendar Roll,
Diagonal Roll, Iron Condor Roll, Straddle Roll, Strangle Roll, Collar Roll,
Rolling Up, Rolling Down, Rolling Out, Rolling In, Rolling Up and Out,
Rolling Down and Out, Stock Repair Strategy, Married Put, Covered Call
Income, Buy-Write, Overwrite Strategy, Put-Write, Wheel Strategy, Covered
Wheel, Dividend Capture With Options, Early Assignment Strategy, Long
Volatility Trade, Short Volatility Trade, Vega Trade, Theta Decay Trade,
Delta-Neutral Trade, Gamma Scalping, Delta Hedging, Dynamic Hedging,
Volatility Skew Trade, Volatility Smile Trade, Term Structure Trade, IV Rank
Trade, IV Percentile Trade, Volatility Crush Trade, Volatility Expansion
Trade, Long Vega Calendar, Short Vega Calendar, Variance Risk Premium
Strategy, Earnings Long Call, Earnings Long Put, Earnings Long Straddle,
Earnings Long Strangle, Earnings Short Straddle, Earnings Short Strangle,
Earnings Iron Condor, Earnings Iron Butterfly, Earnings IV Crush,
Post-Earnings Drift Options, Pre-Earnings Run-Up Options, Post-Earnings Gap
Fill Options, 0DTE Long Call, 0DTE Long Put, 0DTE Debit Spread, 0DTE Credit
Spread, 0DTE Iron Condor, 0DTE Iron Butterfly, 0DTE Straddle, 0DTE Strangle,
0DTE Gamma Scalping, 0DTE Momentum Scalping, 0DTE Mean Reversion, 0DTE VWAP
Reclaim, 0DTE Opening Range Breakout, 0DTE Power Hour Options, Index Options
Scalping, SPX Options Scalping, SPY Options Scalping, QQQ Options Scalping,
NDX Options Scalping, XSP Options, RUT Options, VIX Options, VIX Call
Spread, VIX Put Spread, VIX Calendar Spread, VIX Futures Options Strategy,
Sector ETF Options, Single-Stock Options, ETF Options, Index Options, Weekly
Options, Monthly Options, Quarterly Options, AM-Settled Index Options,
PM-Settled Index Options, Long Call Momentum, Long Put Momentum, Call
Breakout, Put Breakdown, Call Pullback, Put Pullback, Call VWAP Pullback,
Put VWAP Rejection, Call Opening Range Breakout, Put Opening Range
Breakdown, Call Break and Retest, Put Support Break and Retest, Call Trend
Continuation, Put Trend Continuation, Call Relative Strength, Put Relative
Weakness, Credit Spread Income, Iron Condor Income, Iron Butterfly Income,
Covered Call Income, Cash-Secured Put Income, Calendar Income, Diagonal
Income, Theta Portfolio, Premium Selling Basket, Defined-Risk Premium
Selling, Undefined-Risk Premium Selling, Long Futures, Short Futures,
Outright Futures Trade, Directional Futures Trade, Trend-Following Futures,
Countertrend Futures, Momentum Futures, Mean-Reversion Futures, Breakout
Futures, Pullback Futures, Range Futures, Scalping Futures, Swing Futures,
Position Futures, Macro Futures, ES Long, ES Short, NQ Long, NQ Short, YM
Long, YM Short, RTY Long, RTY Short, MES Long, MES Short, MNQ Long, MNQ
Short, MYM Long, MYM Short, M2K Long, M2K Short, ES VWAP Pullback, NQ VWAP
Pullback, MES VWAP Pullback, MNQ VWAP Pullback, ES VWAP Reclaim, NQ VWAP
Reclaim, ES VWAP Rejection, NQ VWAP Rejection, ES VWAP Mean Reversion, NQ
VWAP Mean Reversion, Anchored VWAP Futures Strategy, 5-Minute ORB, 15-Minute
ORB, 30-Minute ORB, Initial Balance Breakout, Initial Balance Rejection,
Initial Balance Rotation, Initial Balance Extension, Opening Drive, Opening
Reversal, Open Test Drive, Open Rejection Reverse, Previous Close Reclaim,
Previous Close Rejection, Overnight High Breakout, Overnight High Rejection,
Overnight Low Breakdown, Overnight Low Reclaim, Globex High Retest, Globex
Low Retest, Regular Trading Hours High Breakout, Regular Trading Hours Low
Breakdown, 9 EMA Pullback, 20 EMA Pullback, 50 SMA Pullback, Trend Day
Continuation, Trend Day Pullback, Trend Day Late-Day Continuation,
Two-Legged Pullback, ABC Pullback, Flag Continuation, Bull Flag Futures,
Bear Flag Futures, Range Breakdown, Triangle Breakout, Wedge Breakout,
Pennant Breakout, Inside Bar Breakout, Outside Bar Continuation, Failed
Breakout, Failed Breakdown, Breakdown Pullback, Prior High Bounce, Prior Low
Bounce, Prior Close Bounce, Overnight Midpoint Bounce, Range Support Long,
Range Resistance Short, VWAP Support Bounce, VWAP Resistance Rejection,
Prior Close Reversion, Overnight Midpoint Reversion, Value Area Mean
Reversion, Point of Control Reversion, Opening Range Midpoint Reversion,
Exhaustion Fade, Climactic Reversal, Blow-Off Top Short, Panic Flush Long,
Buy-Side Liquidity Sweep, Sell-Side Liquidity Sweep, Swing Failure Pattern,
High Sweep Reversal, Low Sweep Reversal, Trap Long, Trap Short, Bull Trap
Fade, Bear Trap Reclaim, Market Profile Value Area Breakout, Market Profile
Value Area Rejection, Market Profile Value Area Rotation, Value Area High
Rejection, Value Area Low Reclaim, Point of Control Rejection, Volume
Profile High-Volume Node Reversion, Volume Profile Low-Volume Node
Breakout, Single Prints Continuation, Poor High Reversal, Poor Low Reversal,
Excess High Fade, Excess Low Fade, Balanced Day Rotation, Trend Day Profile
Continuation, Double Distribution Day Strategy, Order Flow Absorption, Bid
Absorption, Ask Absorption, Delta Divergence, Cumulative Delta Reversal,
Footprint Imbalance, Stacked Imbalance Continuation, Order Flow Reversal,
Order Flow Momentum, Iceberg Absorption, Tape Reading Scalp, Large Lot
Tracking, Market Buy Imbalance, Market Sell Imbalance, DOM Scalping, Level 2
Futures Scalping, Time and Sales Scalping, Breadth Confirmation Futures,
NYSE TICK Futures Strategy, ADD Confirmation, VOLD Confirmation, Market
Internals Trend Confirmation, ES/NQ Divergence, NQ Leading ES, ES Leading
NQ, RTY Risk-On Confirmation, Dow Confirmation, Semiconductor Confirmation
for NQ, Treasury Yield Confirmation, Dollar Confirmation, VIX Confirmation,
ES/NQ Relative Strength, ES/YM Spread, NQ/ES Spread, RTY/ES Spread, RTY/NQ
Spread, Dow/Nasdaq Rotation, Small Cap vs Large Cap Futures, Growth vs Value
Futures Proxy, Risk-On / Risk-Off Futures Pair, Futures Spread Trading,
Intramarket Calendar Spread, Intermarket Spread, Intercommodity Spread,
Crack Spread, Crush Spread, Spark Spread, TED Spread, Treasury Yield Curve
Spread, Commodity Calendar Spread, Bull Calendar Spread, Bear Calendar
Spread, Nearby vs Deferred Spread, Front-Month / Back-Month Spread, Roll
Yield Strategy, Contango Strategy, Backwardation Strategy, ES Calendar
Spread, NQ Calendar Spread, Crude Oil Calendar Spread, Natural Gas Calendar
Spread, Gold Calendar Spread, Silver Calendar Spread, Copper Calendar
Spread, Corn Calendar Spread, Soybean Calendar Spread, Wheat Calendar
Spread, Live Cattle Calendar Spread, Treasury Futures Calendar Spread, Crude
Oil Futures Long, Crude Oil Futures Short, Crude Oil Breakout, Crude Oil
Inventory Report Strategy, Crude Oil Gap Fill, Crude Oil Calendar Spread,
Crude Oil Crack Spread, Brent/WTI Spread, Heating Oil Spread, RBOB Gasoline
Spread, Natural Gas Futures Long, Natural Gas Futures Short, Natural Gas
Weather Trade, Natural Gas Inventory Strategy, Natural Gas Calendar Spread,
Gold Futures Long, Gold Futures Short, Silver Futures Long, Silver Futures
Short, Copper Futures Long, Copper Futures Short, Gold/Silver Ratio Trade,
Gold/Copper Ratio Trade, Metals Breakout, Metals Mean Reversion,
Dollar-Confirmed Gold Trade, Rate-Confirmed Gold Trade, Corn Futures Long,
Corn Futures Short, Soybean Futures Long, Soybean Futures Short, Wheat
Futures Long, Wheat Futures Short, Soybean Crush Spread, Corn/Wheat Spread,
Corn/Soybean Spread, Old Crop / New Crop Spread, Weather-Driven Grain Trade,
USDA Report Strategy, Crop Progress Strategy, Seasonal Grain Spread, Live
Cattle Futures, Feeder Cattle Futures, Lean Hog Futures, Cattle Crush
Spread, Livestock Seasonal Strategy, Livestock Report Strategy, Feed Cost
Hedge, Livestock Calendar Spread, Treasury Futures Long, Treasury Futures
Short, 2-Year Treasury Futures, 5-Year Treasury Futures, 10-Year Treasury
Futures, 30-Year Bond Futures, Ultra Bond Futures, Yield Curve Steepener,
Yield Curve Flattener, 2s10s Futures Spread, 5s30s Futures Spread, Duration
Hedge, Rate Cut Trade, Rate Hike Trade, FOMC Futures Strategy, CPI Futures
Strategy, NFP Futures Strategy, Currency Futures Long, Currency Futures
Short, Euro FX Futures, British Pound Futures, Japanese Yen Futures, Swiss
Franc Futures, Canadian Dollar Futures, Australian Dollar Futures, Dollar
Index Futures, FX Carry Futures, FX Breakout, FX Mean Reversion, Central
Bank Futures Strategy, Interest Rate Differential Strategy, Bitcoin Futures
Long, Bitcoin Futures Short, Micro Bitcoin Futures, Ether Futures, Micro
Ether Futures, Crypto Futures Breakout, Crypto Futures Mean Reversion,
Crypto Basis Trade, Spot/Futures Basis, Cash-and-Carry Crypto Futures,
Perpetual Futures Funding Strategy, Crypto Calendar Spread, Volatility
Futures, VIX Futures Long, VIX Futures Short, VIX Calendar Spread, VIX
Contango Short Vol Strategy, VIX Backwardation Long Vol Strategy, Volatility
Hedge, Equity Hedge With VIX Futures, Event Volatility Futures Strategy,
Weather Futures, HDD Futures, CDD Futures, Temperature Hedge, Energy Demand
Weather Hedge, Weather Derivatives Strategy, Hedging With Futures, Portfolio
Hedge With ES, Portfolio Hedge With MES, Nasdaq Hedge With NQ, Nasdaq Hedge
With MNQ, Commodity Producer Hedge, Commodity Consumer Hedge, Currency
Hedge, Interest Rate Hedge, Inflation Hedge, Cross Hedge, Beta Hedge, Delta
Hedge With Futures, Cash-and-Carry Arbitrage, Reverse Cash-and-Carry
Arbitrage, Basis Trade, Basis Convergence Trade, Globex Session Breakout,
Globex Session Fade, London Session Breakout, London Session Reversal, New
York Open Breakout, New York Open Reversal, Power Hour Futures, End-of-Day
Futures Rebalance, Overnight Futures Trend, Overnight Futures Mean
Reversion, Asia Session Futures Strategy, Europe Session Futures Strategy,
News Futures Strategy, Fed Day Futures Strategy, PPI Futures Strategy, ISM
Futures Strategy, Retail Sales Futures Strategy, Treasury Auction Futures
Strategy, Oil Inventory Futures Strategy, EIA Natural Gas Futures Strategy,
OPEC Meeting Futures Strategy, Futures Scalping, One-Tick Scalping, Two-Tick
Scalping, Order Flow Scalping, Momentum Scalping, Reversal Scalping, VWAP
Scalping, Opening Range Scalping, Micro Pullback Scalping, Breakout
Scalping, Liquidity Sweep Scalping, Market Maker Style Scalping,
Algorithmic Futures Trend Following, Algorithmic Futures Mean Reversion,
Algorithmic Futures Breakout, Statistical Arbitrage Futures, Machine
Learning Futures Strategy, Reinforcement Learning Futures Strategy,
Volatility Targeting Futures, Risk Parity Futures, Managed Futures CTA
Strategy, Time-Series Momentum, Cross-Sectional Momentum, Trend-Following
Basket, Commodity Momentum Basket, Macro Futures Basket, Long Call on
Futures, Long Put on Futures, Short Call on Futures, Short Put on Futures,
Covered Futures Call, Covered Futures Put, Protective Put on Futures,
Protective Call on Futures, Bull Call Spread on Futures, Bull Put Spread on
Futures, Bear Call Spread on Futures, Bear Put Spread on Futures, Debit
Spread on Futures, Credit Spread on Futures, Vertical Spread on Futures,
Long Straddle on Futures, Short Straddle on Futures, Long Strangle on
Futures, Short Strangle on Futures, Iron Condor on Futures, Iron Butterfly
on Futures, Call Butterfly on Futures, Put Butterfly on Futures, Calendar
Spread on Futures Options, Diagonal Spread on Futures Options, Ratio Spread
on Futures Options, Backspread on Futures Options, ES Options on Futures,
MES Options on Futures, NQ Options on Futures, MNQ Options on Futures, Crude
Oil Options on Futures, Natural Gas Options on Futures, Gold Options on
Futures, Silver Options on Futures, Copper Options on Futures, Corn Options
on Futures, Soybean Options on Futures, Wheat Options on Futures, Treasury
Options on Futures, Currency Options on Futures, Bitcoin Options on
Futures, Ether Options on Futures, VIX Options on Futures, Futures Options
Delta Hedge, Futures Options Gamma Scalping, Futures Options Vega Trade,
Futures Options Theta Trade, Futures Options Skew Trade, Futures Options
Volatility Crush, Futures Options Volatility Expansion, Futures Options
Event Trade, Futures Options Report Trade, Futures Options Macro Event
Trade

</details>

### Instrument universe

Notes on which instruments to work with, as originally scoped:

- **Core index ETFs**: SPY, QQQ, IWM, DIA
- **Sector ETFs**: XLK, XLF, XLV, XLE, XLY, XLP, XLI, XLU, XLC, XLB, XLRE, SMH
- **Macro ETFs**: TLT, HYG, LQD, UUP, GLD, SLV, USO
- **Major stocks**: AAPL, MSFT, AMZN, META, GOOGL, NVDA, TSLA, AMD
- **Leveraged/inverse ETFs**: TQQQ, SQQQ, SPXL, SPXS, SOXL, SOXS, UPRO, UVXY
- **Futures**: /ES, /MES, /NQ, /MNQ, /RTY, /M2K, /YM, /MYM, /CL, /MCL, /GC,
  /MGC, /ZN, /ZB
- **Options underlyings**: SPY, QQQ, IWM, AAPL, MSFT, NVDA, TSLA, AMD, META,
  AMZN, GOOGL, TLT, SMH, GLD, USO
- Also noted: random gappers, small caps, low-float stocks, and meme stocks
  as a distinct, higher-risk category of their own.

### Research workflow for a chosen candidate
1. Write the rules out precisely first, in plain English, the same level of
   detail the original VWAP Pullback Long spec had — entry, stop, target,
   sizing, session window, invalidation, daily limits.
2. Build it as its own module following the `FUTURE STRATEGY MODULES`
   pattern in the script (own id, own state, own inputs) — never touching
   the existing module's code.
3. Run it through the *same* Phase 1 validation checklist independently.
4. Only after it passes on its own, evaluate it *combined* with the existing
   module (shared account, shared single-position slot) to confirm it
   doesn't just cannibalize the existing module's trades.

---

## Cross-cutting, all phases

- **Secrets**: broker API keys and webhook secrets never belong in this
  repo or in Pine code. Keep them in whatever secret store your bridge/server
  uses.
- **Version discipline**: once something is live, changes to its logic go
  through Phase 1's validation again before replacing what's running — don't
  hot-patch a live strategy.
- **This stays research/paper-trading until Phase 6.** Nothing before that
  point should touch a real order, on purpose.
