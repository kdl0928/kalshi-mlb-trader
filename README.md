# kalshi-mlb-trader

A research-to-production trading system for Kalshi MLB moneyline markets. It records the
market's live tick stream, aligns it to what happened on the field, tests trading ideas
under an adversarial research protocol, and runs the surviving strategy with real money.

This repository is the public write-up of the project: the system documentation, the
research protocol, and a detailed description of the strategy. The implementation, the
research journal, and the trade records live in a private repository that is available
on request.

The numbers behind the project: roughly 430 recorded game sessions across about 290
markets, more than a dozen strategy ideas tested and mostly killed, one survivor
deployed live, and 12 offline regression suites that each exist because a real-money
run broke in a way worth never repeating.

## The market

Each MLB game has two Kalshi markets, one per team, and each is a binary contract that
settles at $1 if that team wins and $0 if it does not. Prices are quoted in cents, so a
35 cent contract implies a 35 percent win probability. Kalshi charges fees per contract
that scale with P(1-P), about 0.07 times that product for takers and a quarter of that
for makers, so fees peak for coin-flip prices and shrink toward the extremes. On a
market whose spread is often one or two cents, those fees are the dominant cost, and
they shape every design decision in this project.

## Live results

As of August 30, 2026 the deployed strategy is up 25.3 percent on the current era's
starting bankroll, net of all fees, across 205 round trips covering 2,609 contracts.
That works out to 3.8 cents per contract after costs, clearing roughly 2.4 times its
own friction, which matters because friction is what killed every naive idea in the
research phase.

That result is the product of three deployment eras, each one a deployment, a
measurement, and a change:

| Era | Config | Outcome |
|---|---|---|
| Jul 2026 pilot | $50 bankroll, size capped at 1 contract | Down about 1 percent. The goal was to prove the order path, and it exposed a series of real-execution bugs that became permanent fixes and tests. |
| Aug 2026 | $451 bankroll, flat 15 contract orders selected by the model, five ladder depths, 500 ms re-peg | Down 12.6 percent. Diagnosis: the deepest ladder rung lost 5.2 cents per contract, filled overwhelmingly toward the batting team, and consumed about a third of the API write budget while filling least. |
| Aug 23 amendment | Deepest rung removed, re-peg tightened to 350 ms inside the same API budget, drift gate added | Up 25.3 percent on 205 round trips. |

The August 23 amendment changed three things, and each is worth explaining. The
deepest rung was removed because it was a run-catcher: deep orders fill almost
exclusively when the batting team is surging, which is exactly when the move tends to
continue rather than snap back, and the idle rung also consumed about a third of the
API write budget while filling least. Removing it freed enough of the token budget to
tighten the re-peg from 500 ms to 350 ms without changing tiers. And a drift gate was
added: while a batting team's price is climbing, the orders positioned against that
climb are pulled. The thesis is asymmetric. A batting team's upward moves are usually
real rallies that continue, while its sudden drops are the overreactions worth fading.

The loop working end to end is the point: a measured loss was traced to a specific
cause, the fix was made as a deliberate amendment, and the sign flipped.

## The strategy, in detail

Since the code is not public, this section describes what it does.

**The core idea.** When an on-field event moves a game's win probability, the market
often overshoots: a burst of aggressive orders walks the book well past where the price
settles minutes later. Instead of racing to react to the event, the strategy keeps limit
orders already resting deep in the book, so the panicked flow fills them at a discount
and the position profits as the price snaps back.

**Geometry.** Each game has two mirrored markets, and both are quoted on both sides.
Orders rest at four fixed depths below the reference mid for buys and the mirror depths
above it for sells, roughly 15 to 25 cents away from the market, with guard rails that
keep every order between 2 and 98 cents. Each (market, side, depth) slot holds one
resting order, so a full game carries a small ladder of orders on each side.

**Re-pegging.** The reference mid comes from an order book rebuilt in real time from
the exchange's delta feed, not from its lagging quote channel. When the mid moves, each
resting order is cancelled and replaced at its new depth, with one cancel and replace
in flight per order slot and a 350 ms cadence. During a cascade this lag is a feature:
the resting orders are still priced off the pre-cascade mid when the burst arrives,
which is exactly what lets them get filled at a discount.

**Order selection.** Sizing is flat: every order is posted at the same size, currently
15 contracts. What the model controls is which orders exist. At the moment an order
would be posted, the engine computes 32 order-book features (short-horizon drift,
realized volatility, spread windows, traded volume at and through the order's level,
and flow across the whole day's slate of games) and 7 game-state features (score,
inning, baserunners, and related state from a live MLB data poll). An elastic net
predicts the expected profit of a fill at that spot, a quantile regression estimates
the downside tail, and only the spots that clear an edge cutoff are posted at all.
This selection step is what makes the ladder feasible: Kalshi bills every order write
against a token budget, and quoting every depth of every game at all times would blow
through it. Cutting placements down to the spots with predicted edge is how the ladder
fits inside the budget. The fitted coefficients are frozen in a versioned snapshot, so
no research-side change can silently alter what trades.

**Gates.** No order rests before a game's scheduled start. A slate-wide flow floor
stands the ladder down when the whole day's market activity is too thin to support the
edge. A drift gate watches each game's batting team: while that team's price has
climbed more than a couple of cents over the trailing seconds, the orders positioned
against the climb are pulled, on the asymmetric thesis that a batting team's rallies
are usually real and continue, while its sudden drops are the overreactions worth
fading. Total resting collateral is capped at a fixed fraction
of the bankroll, and the ladder covers at most five games at once, admitting new games
as earlier ones settle.

**Exits.** Every fill is held for 30 seconds and then exited aggressively at the best
available price, accepting the taker fee in exchange for certainty. Markets that
settle mid-hold are resolved from the exchange's own settlement status with the MLB
final score as a fallback. A separate reconciliation layer treats the exchange as the
sole authority on what is held: it polls open positions directly and force-flattens
anything unmanaged, so a dropped fill message can never leave a position without an
owner.

**Execution engineering.** Kalshi bills order writes against a token budget, so the
engine runs a token governor that always reserves enough to cancel everything resting,
batches placements and cancels through the exchange's batch endpoints, drops superseded
placements that are still queued when a newer decision arrives, and keeps order writes
off the market-data path entirely so a burst of re-pegs can never stall the book.

## The data pipeline

Everything starts from a faithful record of the market. A recorder streams live order
book deltas, trades, and quotes over the Kalshi websocket API, writing every message
untouched with a local receipt timestamp. Nothing is filtered at capture time, so every
later idea can be tested against the same ground truth. A second stage pulls the
play-by-play record from the MLB Stats API, and a third joins the two, showing exactly
how prices moved around each on-field event. A daily orchestrator runs all of it
unattended, one recorder per game, from first pitch to final out.

## The backtest

The first research finding was about our own measurements. Early backtests used the
exchange's ticker channel for fills. That channel lags real trades by about 270 ms at
the median, which quietly inflates every result. The backtest library instead rebuilds
the order book from raw deltas and fills against that reconstruction, with fees charged
both ways on every simulated fill.

Ideas were then tested under a protocol designed to kill them. Train and holdout days
are frozen, the holdout is touched exactly once per idea at verdict time, and a skeptic
role attacks each surviving result before anything is promoted. The full rules are in
[PROTOCOL.md](PROTOCOL.md).

## How the research progressed

The earliest ideas tried to ride short-term momentum after on-field events: detect a
move, chase it or fade it, and get out quickly. Those died to two hard limits. Fees and
a 1 to 2 cent spread consumed the edge, and reacting from a home connection meant
always arriving after the professionals. The turning point was inverting the latency
problem instead of fighting it: rather than racing to react, rest deep orders in the
book ahead of time and let overreactions come to them.

From there the work became about selectivity. The resting ladder was first conditioned
on simple signals like spread and order flow, and then the hand-tuned rules were
replaced by the fitted selection model described above.

## From backtest to live

The signal logic is shared between the backtest and the live system, and the feature
engine was verified to match the research code to floating point precision on real
fills. The execution logic cannot be fully shared, because a simulator idealizes things
the real exchange does not forgive.
[docs/BACKTEST_VS_LIVE.md](docs/BACKTEST_VS_LIVE.md) maps every known divergence and
the direction of its bias.

Live trading surfaced limitations the research never modeled, and each one became a
permanent part of the system: a write queue that once stalled the market-data loop for
seconds, an API token budget that drove real strategy decisions, a private fill feed
measured arriving up to 10 seconds late, fractional fill counts, per-contract fees, and
markets that settle without a final trade. Every one of those facts broke the system
once. [docs/INCIDENTS.md](docs/INCIDENTS.md) is the full postmortem catalog, and every
incident is pinned in the private repository by an offline test that replays it against
a fake exchange.

## Open limitations

Two known weaknesses in the current results, each with work in progress against it.

- **The backtest idealizes execution in the strategy's favor.** It reprices every order
  continuously with unlimited simultaneous replacements, while the live engine allows
  one cancel and replace in flight per order and discards decisions that arrive in the
  meantime. The gap is largest during cascades, which is exactly where the edge lives.
  A throttle-aware mode in the ladder runner and replays through the real engine are
  closing this gap.
- **The live sample is small.** The current era covers five trading sessions, one
  strong day carries much of the total, and daily variance is the same order as the
  daily mean. The result is promising rather than proven, and the fix is simply more
  trading days under the unchanged configuration.

## Future directions

- **Close the last modeling gaps.** Teach the backtest the placement gate the live
  trader applies.
- **Scalability.** The sizing analysis shows the edge term, not capital, is the binding
  constraint, so growth comes from covering more markets rather than placing bigger
  orders. Higher API tiers allow more simultaneous games.
- **Faster microstructure.** Each game has two complementary markets, and a filed
  research idea explores cross-market lock arbitrage between them at sub-10 ms latency
  achievable via a virtual machine located near Kalshi's servers.
- **Maker exits.** Taker fees can be expensive, so a future research angle will be
  testing maker exit strategies to recover PnL from taker fees.
- **Other sports.** The pipeline is sport-agnostic outside the ticker format and the
  event feed. NFL markets are the natural next target: fewer games, far more volume per
  game, and a different microstructure worth the same treatment end to end.

## What is in this repository

| File | Contents |
|---|---|
| [docs/ARCHITECTURE.md](docs/ARCHITECTURE.md) | The full system map: recording pipeline, backtest scaffold, paper trader, live trader |
| [docs/BACKTEST_VS_LIVE.md](docs/BACKTEST_VS_LIVE.md) | What the backtest and live trader share, what was verified to agree, and every known divergence |
| [docs/INCIDENTS.md](docs/INCIDENTS.md) | Postmortems of every production incident and the regression test that pins each one |
| [PROTOCOL.md](PROTOCOL.md) | The adversarial research protocol the strategy survived |

The private implementation repository contains the recording pipeline, the backtest
library and runners, every hypothesis spec with its implementation, the full research
journal, the paper trading engine, the live trader, and the 12 offline regression
suites. Access is available on request.
