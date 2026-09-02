> This document describes the full implementation. The file paths it names refer
> to the private implementation repository, which is available on request.

# Does the backtest match what trades live?

The question a reviewer should ask of any research-to-production system is whether the
backtest and the live trader are the same logic or two implementations that merely claim
to agree. This file is the map: what is literally shared, what was verified to agree
empirically, and where the two are known to diverge — each divergence stated with its
direction of bias.

## One strategy definition, consumed by both sides

The strategy is not written twice. Both layers consume the same two artifacts:

- **The hypothesis module** — `research/hypotheses/H011_state_conditioned_ladder.py`
  supplies `rest_levels()` (the rung geometry) and the feature/scoring definitions. The
  backtest runner (`backtest/run_ladder.py`) imports it directly.
- **The frozen coefficients** — `trading/H010B_model_lag100_fit.json` is a byte-for-byte
  snapshot of the research-side fit `research/hypotheses/H011_model_lag100_fit.json`
  (verify: `shasum -a 256` both — identical). The trading engine loads its own copy by
  absolute path, so a research-side edit can never silently change what trades live; the
  matching hash is the proof the freeze held.

## Verified parities (measured, not asserted)

**The feature engine.** `Engine.post_features` in `trading/paper_trader.py` is a stated
*port* of `run_ladder`'s post-time feature block — window semantics, log transforms, the
signed-drift convention, and the NaN policy are copied, not re-derived (see the block
comment above `post_features` and the `MarketHistory` docstring, which commits to
reproducing `backtest_lib._parse_one`'s BBO cache row for row, including the
trade-price-rounding boundary where `0.56*100 = 56.00000000000001` would otherwise slip a
strict through-comparison). Verified empirically on 78 matched fills across two games:
**all 32 order-book features agree to ≤1.3e-8** and the 7 game-state features are exact.

**The fill source.** Both sides price off a **delta-built order book** — the backtest from
its cached BBO series, the engine from live `orderbook_delta` messages. Neither uses the
exchange's `ticker` channel, which lags trades/deltas by ~270 ms median and overstates
every result.

**Exit semantics.** Both sides use the same honest rule: if the book is invalid at exit
time, the exit fills at the first valid state *after*, never before. (The "before" variant
was measured to manufacture +2–4 ¢/contract of phantom edge.)

**Fees.** Charged both ways, always, on both sides — maker on the rest, taker on the exit.

**Replay equivalence.** The live trader's `--replay` mode feeds a recording through the
real engine with no writer attached; it was proved to reproduce the paper twin account
**byte for byte** on the reference recording (identical fills across all CSV columns,
identical P&L, identical write-token counts). This is the standing regression: any engine
change is re-verified by replaying the same recording and diffing the output.

**Pinned by tests.** `tests/test_run_ladder_cycle.py` pins the backtest's peg-schedule and
token-billing semantics; `tests/test_live_repeg_clock.py` pins the engine's re-peg clock;
`tests/test_live_h010b_wiring.py` pins the live scoring path end to end, loading the same
frozen research-side JSON the backtest uses.

## Known divergences, each deliberate and each with a stated bias

1. **`repeg_ms` means two different things.** The backtest models re-peg lag as a pure
   information delay — a rung's level at *t* is `floor(mid(t − lag)) − d`, continuously
   repriced, implicitly permitting unlimited simultaneous replacements
   (`run_ladder.peg_schedule`; `billable_repegs` never sees the lag). The engine throttles:
   one cancel/replace in flight per rung, discarding decisions that arrive while one is in
   flight — at 500 ms that drops ~16 decisions per completed re-peg, hardest during
   cascades, which is where the edge lives. **Bias: the backtest is optimistic at high
   re-peg lag**, and a backtest lag sweep must not be read as predicting engine behaviour
   there. The two suites above pin the two semantics separately so neither can silently
   drift into the other.

2. **Fill model.** The backtest fills a rest only on **strict print-through** at the rest
   price. The paper engine is **queue-aware at-level** — a rest fills once printed volume
   at-or-through its level clears the size displayed there at post time. Live fills are
   whatever the exchange reports. The backtest's rule is the stricter one at the level
   itself; the paper rule is the causal, implementable analog; both are documented at the
   fill machinery in `trading/paper_trader.py`.

3. **When size is decided.** (The deployed configuration sizes every order flat, so
   this divergence is latent machinery rather than live behaviour.) The backtest and
   paper engine can evaluate sizing at the
   *fill* instant; a real resting order cannot — its quantity is committed when the order
   is POSTED. The live path therefore sizes at placement time and enforces fill-instant
   guards by cancelling (the staleness pull) rather than by declining a fill it already
   has.

4. **The placement gate the research never modelled.** The backtest rests every rung
   unconditionally and scores only rungs that were actually hit; the live trader evaluates
   the model cutoff at post time on every rung it might place, so live also uses the
   cutoff as a *placement* gate — and with flat sizing deployed, that gate is the
   model's entire job. The `place_thr`/`thr` split makes the two jobs explicit;
   with both unset the frozen spec is reproduced byte for byte.

5. **Rung geometry at the cent boundary.** The live event-driven re-peg occasionally pegs
   a rung 1¢ away from the backtest's continuous-re-peg idealisation. This affects exactly
   one feature (`p_thru60s`, the only rung-level-dependent one); recomputing the backtest's
   own definition *at the live rung's level* reproduces the live value to ≤2e-10, which is
   what isolates the divergence to geometry rather than feature code.

6. **Slate universe.** Live slate-flow features are computed over the *subscribed* slate
   with causal minute buckets; the backtest uses the day's *cached* markets and can
   include prints made after the evaluation instant in its last bucket. Same definitions —
   verified to ≤2e-9 when fed the identical print universe.

7. **Fees, precision.** The backtest's fee helpers round each order up to a whole cent;
   Kalshi bills to four decimals. **Bias: the backtest overstates fees** (~1 ¢/contract at
   size 1). The live trader books the exchange's own reported fee, never the model's.

8. **Model fit geometry vs. operative cadence.** The frozen coefficients were fitted on
   features computed at a 100 ms post-lag (`*_lag100_fit`); the live account currently
   re-pegs at 350 ms, so the model is applied at a slower cadence than it was fitted at.
   This was a measured decision — the backtest lag sweep found model-sized P&L roughly
   flat from 100→500 ms — subject to caveat 1 above: that sweep is the idealisation, so
   the true high-lag behaviour is likely somewhat worse than it shows.

## How to review this in an afternoon

Read `research/hypotheses/H011_state_conditioned_ladder.py` (the strategy: geometry,
features, scoring) and `backtest/run_ladder.py`'s module docstring (the execution
idealisation and its honesty rules). Then read the `MarketHistory` and `post_features`
blocks in `trading/paper_trader.py` — the port and its stated irreducible differences —
and the `SPECS` block in `trading/live_trader_core.py` for the operative live
configuration and the reasoning behind each amendment. Finally run the suites in
`tests/`: they are offline, take minutes, and each one pins one of the correspondences or
divergences above.
