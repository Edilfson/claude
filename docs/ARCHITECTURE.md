# ARCHITECTURE

Status: **DRAFT, awaiting approval (gate 2 of 3)**

## 1. Stack

Python 3.12 · `alpaca-py` · numpy/pandas · SQLite (WAL) for the decision log · FastAPI + Server-Sent Events for the dashboard · Telegram Bot HTTP API · `anthropic` SDK for the nightly Opus loop · pytest + hypothesis · Docker + systemd.

## 2. Layout

```
bot/
  config.py          # loads .env; secrets wrapped in SecretStr; log redaction filter
  data/              # Alpaca bars/quotes/trades: historical loader + live stream
  state/engine.py    # Snapshot builder (pure function of data < t)
  jev/client.py      # JevClient protocol + real adapter + FakeJev for tests/backtest
  jev/questions.yaml # fixed-outcome questions (compiled from strategy.md)
  decide/combiner.py # thresholds + weights from strategy.md -> Signal | None
  decide/sizing.py   # capped fractional Kelly
  calib/             # decision log, Brier/ECE, reliability, isotonic recal
  risk/limits.py     # HARD constants (tighten-only overrides)
  risk/guard.py      # pre-trade checks, daily loss, drawdown, kill switch
  exec/router.py     # idempotent order placement, approval hook, paper-only lock
  approve/telegram.py# alerts + /kill + inline approve/reject (owner chat only)
  backtest/          # event-driven engine reusing state/decide/risk code paths
  gate/              # walk-forward, metrics, deflated Sharpe, regime split, report
  brain/nightly.py   # Opus proposes diffs -> gate -> tag/promote or discard
  dash/              # FastAPI + SSE + single HTML page
  report/daily.py
strategy.md
tests/
deploy/ (Dockerfile, docker-compose.yml, tradingbot.service, healthcheck)
```

## 3. Candle loop (live and backtest share it)

```
bar close t
  └─ state.engine.build(data < t)            -> Snapshot (stale? -> skip)
  └─ jev.score(snapshot, questions)          -> raw probs   (timeout 800ms -> skip)
  └─ calib.recalibrate(raw)                  -> probs
  └─ decide.combiner(probs, strategy)        -> Signal | None
  └─ decide.sizing(signal, probs, equity)    -> qty
  └─ risk.guard.check(order, portfolio)      -> Approved | Veto(reason)
  └─ approval if notional > $10,000          -> Telegram, 60s timeout = reject
  └─ exec.router.submit(order)               -> fill events
  └─ calib.log(decision); dash.publish(event); telegram.alert(fill)
```

The backtest swaps only the data source, the Jev client (rule-only baseline, or FakeJev), and the broker (a simulated fill model with costs). Every other module is the exact live code.

## 4. Trust boundaries

- `risk/` and `exec/` never import anything from `brain/` or `jev/`. A test checks the import graph.
- Opus can modify only `strategy.md` and `jev/questions.yaml` and only on a staging branch. A CI check rejects any diff that touches `risk/`, `gate/`, or `tests/`.
- Telegram commands are accepted only from `TELEGRAM_CHAT_ID`. Every other inbound text is ignored and logged as data.

## 5. Kill switch

A halt flag is stored both in SQLite and as the file `/var/lib/tradingbot/HALT`. Triggering it runs `cancel_all_orders()` → `close_all_positions()` → verifies that positions = ∅ (retrying with backoff) → raises a Telegram alert. On startup the bot refuses to trade while the flag exists.

## 6. Rollback

Each promoted strategy/code version is a git tag `release-YYYYMMDD-N`. Running `deploy/rollback.sh` checks out the previous tag and restarts the service. The decision log records the release tag for every decision.

## 7. Open items before build

- **AgenKit (agenkit.xyz):** I could not reach the site from this sandbox (the proxy returned 403), so I could not verify what its install command would run. I will not pipe-run an unverified installer. Until you share its repo/package or allow the domain, the six phases (brainstorm → architecture → plan → test-first build → review → ship) run as in-repo gates: these docs + TDD + review + tagged releases.
- **Jev SDK:** the only public description I found is secondary (it mentions a "TypeSafe" Python SDK, `TYPESAFE_API_KEY`, and model `jev-latest`). I need the official package name/docs from you before installing. All code depends on the `JevClient` protocol, so this does not block the build.
- **Data plan:** free IEX vs paid SIP feed (affects the quality of the spread/imbalance features).
