# PLAN

Status: **DRAFT, awaiting approval (gate 3 of 3)**

Each step starts with a failing test, then the implementation, then a review against SPEC, and ends with a commit. Each step is one commit and can be reverted on its own.

| # | Module | First failing test(s) |
|---|---|---|
| 1 | Scaffold, `config`, secret redaction | a log line containing a key is redacted; `.env` absent → clear error; live endpoint refused |
| 2 | `risk/limits` + `risk/guard` | oversize order vetoed; daily loss breach → halt; DD breach → kill; overrides can only tighten |
| 3 | Kill switch + `exec/router` (paper, mocked broker) | kill → cancel + close + flag; flag present → no orders; duplicate client id rejected |
| 4 | `state/engine` | property test: mutating any data at ≥ t never changes snapshot(t); stale data → None |
| 5 | Data loader (Alpaca historical, cached parquet) | schema/timezone/RTH filtering; gaps flagged |
| 6 | `backtest` engine + cost model | known toy series → exact expected P&L incl. costs |
| 7 | `gate` (walk-forward, metrics, DSR, regimes) | metrics match reference values on fixtures; gate rejects synthetic overfit |
| 8 | Strategy research run → `strategy.md` | gate report is reproducible from seed + data hash |
| 9 | `jev/client` + FakeJev + `questions.yaml` | malformed/timeout response → no trade; outcome sets are validated |
| 10 | `decide/combiner` + `sizing` | any prob below threshold → no signal; quarter-Kelly cap; uncalibrated → min size |
| 11 | `calib` | Brier/ECE against hand-computed values; isotonic map is monotone |
| 12 | Telegram alerts/approval/`/kill` | non-owner chat ignored; >$10k waits; timeout → reject |
| 13 | Dashboard (FastAPI + SSE) | event appears on stream; no secrets in payloads |
| 14 | Daily report | report fields from fixture log |
| 15 | `brain/nightly` (Opus) | diff touching risk/gate/tests rejected; failed gate → discarded |
| 16 | Deploy: Docker, systemd, healthcheck, rollback | container restarts on crash; rollback script restores previous tag |
| 17 | Paper run + kill-switch drill + go-live checklist | — (operational) |

What I need from you, besides approving these docs:
1. An Alpaca **paper** API key/secret → put them in `.env` yourself (do not paste them into chat).
2. The official Jev SDK package/docs and a key.
3. An AgenKit repo/package link, or confirmation to proceed with the in-repo gates.
4. A Telegram bot token + chat ID (they can wait until step 12).
5. An `ANTHROPIC_API_KEY` for the nightly loop (it can wait until step 15).
