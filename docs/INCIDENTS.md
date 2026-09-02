> This document describes the full implementation. The file paths it names refer
> to the private implementation repository, which is available on request.

# Incidents

Every regression suite in `tests/` exists because a real-money run lost or mismanaged a
position, or because a measurement showed the system doing something other than what it was
believed to be doing. This file is the map from incident to fix to test. Dates are the day
the incident occurred or was measured.

---

## The lagged private feed (2026-07-29) — `test_live_lagged_fill.py`

The first real-money run got exactly one fill and lost it. The private `fill` websocket
message arrived **8.45 seconds after the fill** (measured across the run: p50 1.8 s, max
10.2 s). In that window the engine re-pegged the rung — cancelling the order — so when the
fill finally arrived there was no rung to attach it to and it was dropped. The reconciler
then correctly halted on the position divergence, but halting is not booking: the contract
sat unmanaged.

Fixes, all of which are rules rather than tuning: a cancel answered with 404 (or accepted
with `reduced_by == 0`) means *already gone, almost certainly filled* — it is evidence, not
an error; a pulled rung is **retained for 60 s** keyed by order id so its own late fill is
still attributable; the hold clock runs from the exchange's timestamp, not receipt, so feed
lag is consumed by the hold rather than extending it; an order id this run placed is ours
forever and can never be classified as an orphan; and a divergence on a market with a live
pending record is *explained* and does not halt.

## Partial fills and sub-contract fragments (2026-08-13) — `test_live_partial_fill.py`

At size 1 an order is wholly consumed by its first fill; at size 15 partial fills are
routine. A rung was modelled as one order that dies on first fill, so a partially-filled
order's residual stayed live at the exchange with the ladder unable to re-peg, cancel or
account for it. Separately, Kalshi reports fixed-point fill counts (1.42 contracts) while
orders post integer counts, and fragments under half a contract were silently rounded to
zero — an unmanaged real position. Rungs now carry open quantity and release collateral in
proportion; fragments accumulate per rung and book as whole contracts as they cross 1; a
fragment that can never complete is escalated loudly, never swallowed.

## Fees: `average_fee_paid` is per contract (2026-08-13) — `test_live_exchange_fees.py`

Kalshi's exit ack reports the *average* fee per contract; booking it as the order total
under-charged every multi-contract exit by a factor of the count, and was the entire
books-vs-exchange discrepancy on the day it was found ($0.66 of a $0.67 gap). Invisible at
size 1, where the two numbers coincide. The general rule the suite pins: book the fee the
exchange charged, never the fee the model computed.

## The sizing model ran as an on/off gate (2026-08-12) — `test_live_h010b_wiring.py`

Raising the per-fill cap from 1 to 15 exposed that the model's committed size was computed
and stored but never put on the wire — every selected rung was posted at the full per-fill
cap. Invisible while cap = model size = 1. The suite pins the whole post-time scoring path:
the placement gate, the committed quantity reaching the order, and the collateral
reservation being trued down from worst case to committed size.

## The write path: queue, batching, conflation (2026-07-31) —
`test_live_async_writes.py`, `test_live_batch_writes.py`, `test_live_shard_priority.py`, `test_live_write_burst.py`

Order writes originally ran synchronously inside the websocket receive loop: a tick
maturing 20 rungs issued 20 serialized ~90 ms round trips and froze the market-data reader
for ~2 s — during which the book, the peg reference and the fill logic were all stale.
Measured on one run: writes blocked the loop 9.6% of total wall time. Made asynchronous, a
second defect appeared: queue waits of p99 9.0 s, max 21.3 s, with the writer pool 22%
utilised — the in-flight gate was the item's hash shard rather than its rung, so ~15
unrelated rungs serialized behind each other. Re-gating per rung, batching writes through
Kalshi's batch endpoints (billed per item — batching is free in tokens and ~5× in round
trips), and conflating still-queued placements superseded by newer decisions took modelled
p99 from ~9 s to ~180 ms at unchanged token spend. Cancels are never conflated — a cancel
is the one write whose omission is unbounded risk.

The invariants the suites hold: per-rung ordering (a rung's cancel is never overtaken by
its own replacement), cancels drain before placements, a batch endpoint failure degrades to
singles without ever stopping the ladder, and a batch's per-order errors are read
individually — a `not_found` inside a 200 envelope is the same "already gone, probably
filled" evidence as a single-order 404 (missing that mapping cost a 4-contract position on
2026-08-14).

## Settlement must not halt the account (ongoing) — `test_live_settle_divergence.py`

A Kalshi market settles at exactly 0 or 100 and its book stops updating, so the normal
taker-exit path can never fire. Settlement is detected from the exchange's own market
status (never from book state — a book can touch 0/100 transiently) with the MLB final
score as fallback, and resolved as bookkeeping, not strategy. The suite pins that a settled
market resolves positions cleanly instead of tripping the divergence halt.

## `repeg_ms` means two different things (2026-07-29) — `test_live_repeg_clock.py`, `test_run_ladder_cycle.py`

The backtest models re-peg lag as a pure information delay with unlimited simultaneous
replacements; the engine throttles to one cancel/replace in flight per rung, discarding
decisions that arrive while one is in flight. At a 500 ms lag the engine drops ~16
decisions per completed re-peg, and the throttle bites hardest during cascades — exactly
where the edge lives. Consequence: a backtest lag sweep does not predict engine behaviour
at high lag, and the two clocks must never be conflated. The suites pin the two semantics
separately.

## Exit off what you hold, not what you recall filling (2026-08-27) — `test_live_v2_flatten.py`

The v1 exit path descends entirely from fill messages: a position the fill feed never
reported is a position it will never exit. The exchange held −420/+300 on one market while
internal state said 0; v1 saw the divergence, halted new orders — and left the position
sitting, because "exit management continues" only covers positions it knows about.
Reconstructing from the fills endpoint does not fix it either (settlement is not a fill, so
settled positions age forever in the reconstruction). `live_trader_v2.py` makes
`GET /portfolio/positions` — the exchange's statement of what is *held* — the sole
authority for the exit path.

---

Recurring themes, for the reader in a hurry: **the exchange is the truth and internal
state is a hypothesis**; **evidence is never swallowed** (a 404, a fragment, an
unattributed fill — each is booked or escalated, never dropped); **feeds lag and
everything downstream must tolerate it**; and **every bug class gets an offline replay
of the real incident before the fix ships**.
