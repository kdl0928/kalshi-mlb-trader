> This document describes the full implementation. The file paths it names refer
> to the private implementation repository, which is available on request.

# Architecture

A research-to-production system for trading Kalshi MLB moneyline markets: it records the
market's raw tick stream, aligns it to what happened on the field, mines that record for
signals under an adversarial research protocol, and runs the surviving strategy as a live
maker ladder against real money.

The system has four layers. Data flows strictly downward — the raw record is sacrosanct,
and every layer above it is rebuildable from it.

```
1. Recording pipeline      raw Kalshi ticks + MLB event logs + aligned joins
2. Research / backtest     delta-built order books, honest fill simulation, hypothesis loop
3. Paper trader            live websocket, simulated fills, N strategy variants side by side
4. Live trader             real orders, one strategy, safety gates and reconciliation
```

**The live trader (`trading/live_trader.py`, on the engine in `trading/live_trader_core.py`) is the only component in the repository that can place an order**,
and only when armed with both `--live` and `--i-understand-real-money-orders`. Everything
else is read-only or simulated by construction.

## 1. Recording pipeline (`pipeline/`)

Three stages plus an orchestrator, run in order:

- **`kalshi_recorder.py`** — streams the Kalshi websocket (`orderbook_delta`, `trade`,
  `ticker`) plus periodic REST book snapshots for a set of market tickers, writing one raw
  JSON object per line. Deliberately dumb: no filtering, no detection. Every line keeps the
  untouched exchange message plus a local receipt time, so nothing is lost and feed latency
  stays measurable offline.
- **`mlb_game_fetch.py`** — pulls the ground truth of what happened on the field from the
  public MLB Stats API and flattens it into a time-sorted per-game event log.
- **`align_kalshi_events.py`** — joins the event log to the recording, emitting one row per
  (event × market): the quote before each play and the price move / prints after it.
  `--broadcast-lag-secs` shifts MLB timestamps forward to model the TV/data-feed delay the
  market actually reacts to.
- **`daily_orchestrator.py`** — stage 0: discovers the day's games and their Kalshi
  markets, starts one recorder per game, stops each when MLB reports the game Final, then
  runs stages 2–3 for everything it recorded. Runs unattended daily under launchd.

Cross-cutting conventions: the join key is `ts_ms` (unix milliseconds, UTC) on both sides;
the Kalshi ticker grammar `KXMLBGAME-<YY><MON><DD><HHMM><AWAY><HOME>-<SIDE>` embeds the
Eastern-time start (doubleheaders append `G1`/`G2` to the team chunk); teams are matched
Kalshi↔MLB by abbreviation.

## 2. Research / backtest layer (`backtest/` + `research/`)

The scaffold contains **no strategy logic** — strategies are plug-in hypothesis modules.

- **`backtest_lib.py`** — parses recordings into a cache of trades and a **delta-built
  order-book BBO series**. This is the honest fill source: the exchange's `ticker` channel
  lags trades/deltas by ~270 ms median, and using it overstates every result. Also holds
  Kalshi fee math (fees are applied both ways, always), the per-game execution simulator
  (BBO-size-capped fills, hold or tp/sl exits), and causal statistics helpers.
- **`run_strategy.py`** — runs one hypothesis module across all cached games; taker-style
  entries after a signal.
- **`run_ladder.py`** — the second runner, for pre-rested, continuously re-pegged maker
  ladders (orders that must already be sitting in the book when a cascade print arrives).
  Fills are strict print-through at the rest price; exits are taker with honest semantics
  (book invalid at exit time → first valid state *after*, never before — the "before"
  variant manufactures phantom edge). Models re-peg latency, write-token billing
  (Kalshi bills 10 tokens per place, 2 per cancel), dead-bands, and staleness pulls.

The research loop itself is documented in [`PROTOCOL.md`](../PROTOCOL.md):
four agent roles (signal-hunter → backtester → skeptic + refiner) cycle over a frozen
TRAIN/HOLDOUT split, with the holdout touched exactly once per hypothesis, at verdict time.
`research/JOURNAL.md` (private repository) is the chronological shared memory of every
round — hypotheses filed, killed, and promoted, with the reasons. `research/hypotheses/`
holds one spec (`.md`) + implementation (`.py`) per idea.

## 3. Paper trader (`trading/paper_trader.py`)

Subscribes to the Kalshi websocket **read-only**, maintains live order books from
`orderbook_delta`, and runs configurable strategy variants side by side, each with its own
paper bankroll. Simulated maker fills are queue-aware at-level (a rest fills once printed
volume at-or-through its level clears the size displayed there at post time). A
`--replay <recording>` mode feeds a recorded tick file through the identical engine —
every change is verified by replaying a real recording and diffing the output byte for byte.

The engine is the shared core: the ladder mechanics (event-driven re-pegs off the
delta-built mid, per-rung cancel/replace lag, episode caps, slate-flow gates, pre-game
gates, settlement resolution via the public market endpoint with an MLB-final-score
fallback), the frozen H010-B state-conditioned placement model (32 order-book features + 7
game-state features, scored at the post instant to decide which rungs are posted at
all — deployed sizing is flat; coefficients are a frozen JSON snapshot so
research-side edits can never silently change what trades live), and write-token accounting
against Kalshi's rate tiers.

## 4. Live trader (`trading/live_trader.py` + `live_trader_core.py`)

Runs the surviving strategy against real orders. It **imports the engine from
`paper_trader.py` — never forks it** — and adds only what real execution requires:

- **Three modes**: `--dry-run` (default, sends nothing), `--demo` (Kalshi's demo
  exchange, separate credentials), `--live` (real money, double-flag armed).
- **A non-blocking write path**: order placement/cancel run on a writer pool gated
  per-rung (a rung's cancel must never be overtaken by its own replacement), batched
  through Kalshi's batch endpoints, with conflation of superseded placements during storms
  and a token governor that always reserves enough to cancel everything resting.
- **Reconciliation**: internal state is a hypothesis; the exchange is the truth. A
  background loop diffs resting orders, positions and balance against internal state —
  phantom orders are dropped, orphans cancelled, unexplained position divergence halts the
  account.
- **Safety gates**: per-fill contract clamp, sticky daily loss limit, refusal to start on
  pre-existing positions/orders, a `HALT` kill-switch file, and unconditional cancel-all on
  shutdown.
- Real fills, exits and fees are booked from what the exchange reports, not what the model
  expected. Output goes to its own `live_trades/` file family so real fills can never
  contaminate the paper research record.

`live_trader_core.py` is the engine — the modes, write path, reconciliation and safety
gates above. `live_trader.py` is the entrypoint: it subclasses the core's reconciler so
that positions are exited off `GET /portfolio/positions` — what the exchange says is
*held* — rather than off an internal ledger reconstructed from fill messages. See its
module docstring for the incident that motivated the split.

`dashboard.py` + `dashboard_ui.html` are a read-only local dashboard: it tails the run
files the traders write (the trades CSV plus the events and prices feeds) and serves
banks, fills and per-market charts for the newest run. It places no orders and reads
only local files.

[`docs/BACKTEST_VS_LIVE.md`](BACKTEST_VS_LIVE.md) is the correspondence map between the
backtest and the live trader: what is literally shared, what was verified to agree
empirically, and the known divergences with their direction of bias.

## Tests (`tests/`)

Twelve self-contained offline regression suites — no network, no credentials, no orders.
Each `test_live_*.py` replays a real production incident against a fake exchange and exits
non-zero on failure; [`docs/INCIDENTS.md`](INCIDENTS.md) maps each suite to the incident
that created it. Run them with any Python that has `requests`:

```bash
for f in tests/test_live_*.py; do python "$f" || echo "FAILED $f"; done
python tests/test_run_ladder_cycle.py   # needs numpy/pandas
```

## What is deliberately not in this repository

The raw tick recordings (~90 GB), the derived caches, per-game backtest result CSVs, the
paper/live trade books, credentials, and logs. The code here regenerates every derived
artifact from a recording; the recordings themselves are the one thing that cannot be
regenerated, and they live outside version control.
