# Research protocol — how agents collaborate here

Every research agent (signal-hunter, backtester, skeptic, refiner) MUST read
`JOURNAL.md` first and write its findings back before finishing. This directory is
the shared memory between agents; nothing said in a conversation counts until it
is written here.

## The loop (v2, user-specified 2026-07-12)

One round = **hunter** (mines TRAIN data, files Hxxx) -> **backtester**
(implements + sweeps on TRAIN) -> **skeptic** (attacks; judges on HOLDOUT +
placebo) **+ refiner** (salvages what survived, files a `briefs/B###` brief)
-> next round starts from the brief. The loop repeats until a spec meets the
preliminary thresholds, or the refiner declares `PAUSE-FOR-DATA`.

**Data splits (generation 3, ratified 2026-08-26 by user):** an **80/20
CHRONOLOGICAL split over ALL recorded data**, replacing gen-2's freeze-date rule
for every hypothesis judged from this date forward. On the 43 recorded days
(2026-07-06 .. 2026-08-25, 499 market-files) that is **FIT = 07-06 .. 08-12**
(34 days, 388 files) and **HOLDOUT = 08-13 .. 08-25** (9 days, 111 files).
Recompute the boundary when new days are recorded; state the exact boundary used
in every results table, since gen-2 and gen-3 numbers are not comparable.
The discipline is unchanged and non-negotiable: all exploration, feature
derivation, fitting and gate calibration happen on FIT with leave-one-day-out CV
(now 34 folds); HOLDOUT is touched exactly once, by the skeptic, at verdict time;
a spec tuned after seeing HOLDOUT is invalid by construction. 2026-07-14 (the
All-Star Game) remains excluded from formal samples per the 2026-07-13
orchestrator entry.
**Why it changed:** gen-2's FIT set was 15 days, and day concentration — not
sample size in fills — is the failure mode this project keeps hitting (H014's
surviving cell was n=122 attempts over 15 days with its best day carrying ~78% of
the total). 34 FIT days test that weakness far harder.
**The caveat that comes with it:** unlike gen-2's "days recorded after the freeze",
a gen-3 holdout consists of days that ALREADY EXIST when the spec is written. It
is honest only for a mechanism nobody has yet examined on those days, and only if
the split is fixed before looking. For anything touching the H008/H009/H010
ladder family, 07-24..08-25 is already burned by the live/paper streams and the
2026-08-21 skeptic decomposition — gen-3 does NOT un-burn it.
**Live-order contamination (new, applies to every study using days from
2026-07-29 onward):** the live trader has placed real orders since 07-29, so our
own resting orders are inside the recorded book on those days. Any study that
reads the touch must quantify whether our own orders create or destroy the
episodes it counts, and report a figure with them excluded.

**Data splits (generation 2, ratified 2026-07-23 by user; proposed in
briefs/B002):** **FIT set** = all recorded days 2026-07-06 through 2026-07-23,
explicitly declared burned (gen-1 TRAIN was mined by four rounds, 07-10..15
went soft, 07-16..22 was spent on the H008 verdict, and the live paper stream
overlaps 07-16..23). All exploration, feature derivation, curve fitting, and
gate calibration happen on the FIT set with **leave-one-day-out CV mandatory**
(per-fold dispersion reported; a component that flips sign across folds does
not enter a frozen spec). No fresh-data claims may be made from the FIT set.
**HOLDOUT** = days recorded strictly AFTER the hypothesis's spec freeze
(recorded in its .md). Touched exactly once, by the skeptic, at verdict time —
and only after the **verdict-read gate**: (a) >= 8 new game-days, (b) >= 60
frozen-spec fills, (c) >= 2 holdout days in the bottom tercile of slate flow
(tercile boundaries computed on the FIT set, definition frozen with the spec).
A spec whose parameters were chosen after seeing HOLDOUT results is invalid by
construction. (`backtest_lib.root_date` gives the split key; days recorded
after a spec is frozen are automatically honest out-of-sample.)
*(Generation 1 — TRAIN 07-06..09, HOLDOUT 07-10 onward, read once — is
exhausted; H001-H008 were judged under it and their verdicts stand.)*

**Preliminary success thresholds (measured on HOLDOUT, net of fees):**
median trade > 0 **or** win rate >= 50%, **and** mean/contract > 0, **and**
the matched placebo increment > 0. A spec meeting these graduates to paper
trading (status: promising). The mean guard exists because win rate and median
are trivially gameable with tight take-profits (measured: H004's mtp exits
raised win rate to 0.6-0.8 while turning the mean more negative).

**Loop control:** the refiner ends every round with `LOOP: CONTINUE (brief)`
or `LOOP: PAUSE-FOR-DATA`. Two consecutive rounds with no component surviving
the skeptic forces PAUSE-FOR-DATA — re-mining the same days past their
information content only manufactures overfits.

## Layout

```
analysis/research/
  JOURNAL.md          running log, newest entry FIRST; hard facts + verdicts only
  PROTOCOL.md         this file
  hypotheses/         one Hxxx_slug.md spec (+ Hxxx_slug.py implementation) per idea
  results/            per-game backtest CSVs written by analysis/run_strategy.py
```

## Hypothesis lifecycle

`proposed → implemented → tested → killed | promising → validated → paper-traded`

A hypothesis file is `hypotheses/Hxxx_slug.md` (next free number) with fields:

```markdown
# Hxxx — one-line name
status: proposed | implemented | tested | killed | promising | validated
owner: signal-hunter | backtester | skeptic
## Rationale        (why this could be a real, fee-beating edge — cite data)
## Spec             (precise, causal trigger + exit; every threshold named)
## Results          (filled by backtester: run command, results CSV, summary table)
## Verdict          (filled by skeptic: splits/bootstrap/sensitivity + kill or pass)
```

The matching `Hxxx_slug.py` implements the spec for `analysis/run_strategy.py`:
`NAME`, `PARAMS`, and `generate_signals(md, params) -> [(ts_ms, side)]` with
side +1 = buy YES, -1 = buy NO. See `hypotheses/H000_template.py`.

## Rules of evidence (non-negotiable)

1. **Causality.** A signal at time t may use only data with ts <= t. No
   full-sample statistics inside `generate_signals` (the `med` field in `md` is
   full-sample — use `backtest_lib.expanding_median` for causal versions).
   **Fill windows too (skeptic amendment, 2026-07-12):** any fill simulation
   must source fills strictly AFTER the trigger cell/second closes, and every
   hypothesis spec must state its fill-window start relative to the trigger
   timestamp. (A ~1s fill look-ahead in exploration manufactured 2/3 of H003's
   claimed edge.)
2. **Fees always.** Never quote a result without Kalshi taker fees both ways
   (`run_game` applies them). Taker 0.07*P*(1-P)/contract rounded up per order;
   maker 0.0175*P*(1-P).
3. **Fills = delta-built book.** All backtests fill via the `bbo` cache
   (delta-built order book), never the `ticker` channel (lags ~270ms median).
   Default latency assumption 100ms; report 50/100/200ms sensitivity for
   anything promising.
   **Test taker AND maker execution as first-class variants (user rule,
   2026-07-12).** Maker entries are cheaper but rest on fill-model assumptions
   (print-through); taker fills are simulated on firmer ground. Report both
   side by side wherever the strategy admits both, and quantify the gap.
4. **A result is per-game first.** Total P&L hides one lucky game. Always report
   games, pos_games (share of profitable games), worst_game, and trade count.
5. **The skeptic gates promotion.** Nothing moves past `tested` without:
   train/test split by date (`backtest_lib.root_date`), bootstrap over games
   (resample per-game totals, 1000x, report P(total>0)), latency sensitivity,
   and parameter-neighborhood check (does +-1 unit on each threshold hold up?).
6. **Killed is a result.** Record WHY it died (fees? no signal? one-game fluke?)
   in the hypothesis file and one line in the journal — it stops the next agent
   from re-proposing it.
7. **Small data honesty.** ~6 days of recordings exist. Anything that survives
   the skeptic is still only `promising`; `validated` requires holding up as new
   days arrive, and real belief requires the live paper trader.

## Journal entry format (prepend to JOURNAL.md)

```markdown
## 2026-07-12 — agent-name
- Hxxx moved proposed->tested: one-line result (total $, pos_games, n trades).
- Fact: <any new reusable fact about the data/market, with numbers>.
- Next: <what the following agent should pick up>.
```
