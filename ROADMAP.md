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

### Candidate ideas for SPY/QQQ intraday (unranked, pick one to start)
- **Short-side mirror of VWAP Pullback Long** — the same trend/impulse/
  pullback/confirmation logic, inverted for downtrends. Lowest-effort
  candidate since it reuses almost the entire existing framework and rule
  structure; good complement on bearish/red days where the long-only module
  never fires.
- **Opening Range Breakout (ORB)** — trade a breakout of the first N-minute
  range (commonly 5 or 15) in the breakout direction. Trades right at the
  open (9:30-9:45), a window the current strategy explicitly excludes.
- **VWAP Reversion / Fade** — the mean-reversion counterpart to the current
  trend-continuation approach: fade price back toward VWAP when it's
  extended too far without a clean trend behind it.
- **Power Hour momentum** — trend-continuation in the last hour of the
  session (3:00-4:00 PM ET), a completely different time-of-day exposure
  than the current 9:45-11:15 window.
- **Gap fill / gap-and-go** — rules based on overnight gap behavior in the
  first few minutes of the session.

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
