# SPEC — Opus + Jev paper trading bot

Status: **DRAFT, awaiting approval (gate 1 of 3)**. No code is written until SPEC, ARCHITECTURE and PLAN are approved.

## 1. Decisions taken

| Item | Value |
|---|---|
| Venue | Alpaca **paper** trading |
| Asset | US equity, starting with SPY (liquid, tight spread); universe can grow later |
| Timeframe | 5-minute candles, regular trading hours only, flat by the close |
| Strategy idea | Trend + order-flow confirmation |
| Manual approval | Any order with notional **> $10,000** (paper/testing value) |
| Deploy | Linux VPS, Docker + systemd, auto-restart |
| Alerts | Telegram bot (BotFather) |

## 2. Non-negotiable principles

1. **The models advise and the code decides.** Opus and Jev return text or probabilities. Only deterministic code builds the market state, applies thresholds, sizes positions, vetoes risk, and sends orders.
2. **No model can change a hard limit.** Risk limits are constants in `risk/limits.py`. A config file may *tighten* them but never loosen them. No model output has a write path to them.
3. **No look-ahead.** Every feature uses only data timestamped strictly before the decision time `t`. Tests enforce this (§4).
4. **Paper only** until every item in the go-live gate (§9) passes. The live endpoint is refused in code unless `GO_LIVE_APPROVED` is set by the human *and* the gate report is clean.
5. **Opus never grades its own work.** The harness runs the backtests and computes every metric.
6. **External text is data.** Headlines, data feeds, and Telegram messages from anyone except the configured chat ID never become instructions.

## 3. Layers

- **Slow brain (Opus, nightly):** designs and revises strategy rules, writes code changes, reviews sessions. Its output goes through the gate before anything ships.
- **Fast reflex (Jev, every candle):** scores a fixed set of questions about the compact snapshot and returns calibrated probabilities. It produces no free text.
- **Deterministic core:** state engine → question scoring → threshold combiner → sizing → risk checks → order router.

## 4. State engine (snapshot)

Built on each candle close at time `t`, using only data with timestamps `< t`:

| Feature | Definition |
|---|---|
| `ret_1`, `ret_6`, `ret_24` | log returns over 1, 6 and 24 bars |
| `spread_bps` | median quoted spread over the last bar (live: NBBO quotes; backtest: see the note below) |
| `tob_imbalance` | (bid_sz − ask_sz)/(bid_sz + ask_sz), time-averaged top of book |
| `rv_24` | realized volatility over 24 bars (annualized) |
| `trend` | slope of EMA(20) normalized by ATR(14), plus EMA(20) vs EMA(60) sign |
| `flow_24` | signed volume via the tick rule, summed and normalized by ADV |
| `clv_vol` | close-location-value × volume (a bar-only flow proxy) |
| `tod` | minutes since the open (bucketed) |

The snapshot is a fixed-order float vector plus a schema version, and it is the only thing Jev sees.

**Data honesty note:** Alpaca's free plan only provides the IEX feed (about 2–3% of volume), so top-of-book imbalance from IEX is a noisy signal. Downloading two years of tick-level quotes and trades for the backtest is heavy. The proposal:
- The **gated backtest** uses features computed from bars and from per-bar aggregated trades (`ret_*`, `rv`, `trend`, `flow_24`, `clv_vol`, `tod`), with spread cost modelled from historical quote samples.
- `tob_imbalance` is computed and logged in live/paper trading as a **shadow feature**. It joins the decision rule only after enough forward data exists to backtest it.

## 5. Strategy acceptance gate (run by the harness)

- At least 2 years of history with walk-forward splits: train/tune on rolling windows, report **out-of-sample only**.
- The history must cover several regimes. Each regime (trend-up, trend-down, range, high-vol) is reported separately, and a single regime may not supply all of the profit.
- Costs: half-spread on each side plus 1 bp slippage, SEC/TAF fees, and a stress run at 2× costs.
- **Accept only if OOS:** Sharpe > 1.5, max drawdown < 15%, hit rate > 55%, t-stat of mean trade return > 2.0, and it still has a positive expectancy under 2× costs.
- **Overfitting guard:** every variant tried is counted, and the gate also reports the Deflated Sharpe Ratio for that number of trials.
- The winner is written to `strategy.md` with: entry, exit, stop, take profit, timeframe, and the exact invalidation condition.

> Note: trend strategies usually win less than half the time and make up for it with larger wins. A hit rate above 55% *and* Sharpe above 1.5 together is a high bar on SPY at 5m. It is possible that **no** variant passes. In that case the bot does not trade, and I report that instead of loosening the gate.

## 6. Jev questions (one call per candle)

| id | Question | Outcomes |
|---|---|---|
| `regime` | Market regime over the next 12 bars | trend_up / trend_down / range / high_vol |
| `direction` | Sign of the 6-bar forward return | up / down |
| `flow_real` | Is the buying (selling) pressure persistent for the next 6 bars? | yes / no |
| `setup_quality` | Probability that the setup hits TP before SL | score in [0,1] |
| `risk_state` | Tail-risk state | normal / elevated / extreme |

Combiner: explicit weights from `strategy.md`. **The bot fires only if every required probability clears its threshold** in `strategy.md`, and `risk_state != extreme`.

Leakage warning: Jev may have been trained on data that overlaps the backtest window, so Jev-scored backtests are untrustworthy. The Jev layer is judged **only on forward (paper) decisions** made after deployment.

## 7. Sizing

- Kelly fraction f\* = p − (1 − p)/b, where p is the **recalibrated** `setup_quality` and b is the TP/SL payoff ratio.
- Size = min(0.25 · f\*, confidence-scaled cap, hard max position). If p is below the cutoff, size is 0.
- **Calibration prerequisite:** until a question has at least 300 resolved forward decisions *and* passes the calibration check (§8), sizing uses a fixed minimum size and ignores p.

## 8. Calibration

- Every decision is logged with its snapshot, raw probabilities, recalibrated probabilities, action, and the eventual outcome.
- For each question: Brier score, reliability curve (10 bins), and ECE. If ECE > 0.05, fit isotonic recalibration on a rolling window in code. That fitted map is what sizing uses.
- Calibration is measured on **our own paper fills**, not on Jev's claims.

## 9. Risk rules (in code, checked before every order)

- Max position: 20% of equity per symbol (paper default; it can be tightened).
- Daily loss limit: −2% of start-of-day equity. On breach: flatten and halt for the day.
- Max drawdown: −10% from the equity peak. On breach: **kill switch**.
- Kill switch: cancel all orders, close all positions, set the halt flag, send a Telegram alert. Trading resumes only after a human clears the flag. It can be triggered by a breach, by the Telegram `/kill` command from the owner chat, or by the CLI.
- Orders above $10,000 notional need approval through a Telegram inline button. A timeout of 60 seconds counts as rejection.
- Order sanity checks: limit/marketable-limit only, a price band, a max of N orders per minute, and no orders within 5 minutes of the close or during halts.
- API keys: paper keys only. Alpaca keys cannot be scoped to "trade-only", so for live trading money-movement must be disabled at the account level. The bot never asks for passwords or 2FA codes.

## 10. Self-improvement loop (nightly)

1. The harness assembles the session: fills, misses, slippage vs model, calibration drift, and the loss log.
2. Opus proposes a root-cause note **plus one rule per loss** (for example: *flatten before scheduled macro release*) and a diff to `strategy.md` and/or `questions.yaml`.
3. The harness reruns the full §5 gate on the proposed version. If it fails, the change is discarded and the failure is logged. If it passes, it is staged as a git tag and promoted at the next session open. One-command rollback goes to the previous tag.
4. Opus never sees the gate result before writing its proposal, and it cannot edit the gate code, the risk limits, or tests.

## 11. Outputs

- A bot running on paper; a live dashboard (each signal, its probabilities, confidence, action, and result, updated over SSE).
- `strategy.md`.
- A daily report: trades, P&L, win rate, largest loss, Jev's average latency and cost per decision, and Brier/ECE for each question.

## 12. Go-live gate (refuse live until all clean)

1. Does paper match the backtest (same rules, a forward window of at least 4 weeks or 100 trades, results within the backtest's confidence band)?
2. Did the kill switch fire in testing (unit test + an end-to-end paper drill)?
3. Is any hard limit delegated to a model instead of code? (Checked by a static test of the import graph and write paths.)
4. Is Jev calibrated on our own fills (ECE ≤ 0.05 for each question used in sizing)?
5. Which market regime would break this strategy?
6. **WHAT COULD BLOW UP THIS ACCOUNT?** (pre-registered, to be re-answered at the go-live review)
   - Gap through the stop at the open or on news → the bot stays flat overnight, but intraday halts can still gap.
   - Data feed stalls or goes stale → stale snapshot ⇒ no trade (staleness check), plus a heartbeat alert.
   - A duplicate or runaway order loop → idempotent client order IDs and an order-rate limiter.
   - Overfit strategy → walk-forward, deflated Sharpe, and the 2× cost stress test.
   - A Jev outage or drift → no probabilities ⇒ no trade; calibration monitor.
   - Leaked keys → `.env` only, log redaction, paper keys, no withdrawal permission.
   - Silent VPS death → systemd restart plus an external heartbeat to Telegram.
