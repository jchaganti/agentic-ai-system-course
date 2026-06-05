# v5 Freeze Checklist — Phase 4 Calibration & Validation

> **What the freeze means.** `swing_trader_blueprint_v5.md` is internally consistent and
> runtime-correct. The freeze locks the **structure** — agents, workflows, schema, the *shape* of
> every formula. It does **not** lock the **numbers**. Every constant in v5 is a hand-set seed.
> Phase 4's Strategy Lab is what earns them.
>
> **The one rule this document exists to enforce:** *no seed constant goes live at its seed value.*
> A frozen architecture running un-calibrated constants is a guess with good production hygiene.

---

## 0. Prerequisites — build these BEFORE you calibrate anything

Calibrating against a dishonest backtest is worse than not calibrating — it manufactures false
confidence. Do these first, in order:

- [ ] **Single simulation engine** (§20.6). The Strategy Lab must reuse the *live* trade logic —
      GTT-style entries/exits, the buffered-LIMIT SL, partial exits + SL resize (the C1 fix),
      breakeven/trail/time-stop, portfolio heat, concurrency caps. The only difference is
      historical bars vs live data. If the Lab has its own separate exit logic, you are
      backtesting a different system than you trade.
- [ ] **Fill the cost + slippage constants** (§20.4) — `COST_PCT_PER_SIDE`,
      `ENTRY_SLIPPAGE_PCT`, `EXIT_SLIPPAGE_PCT`. Until these are real, **every** backtest number
      is friction-free fantasy. No result is trustworthy before this box is checked.
- [ ] **Fill the risk config** (§20.2) — `RISK_PER_TRADE_PCT` and the loss caps
      (`DAILY/WEEKLY/MONTHLY_LOSS_CAP_R`). Sizing affects which trades survive the heat/cap gates,
      which changes the trade set you're calibrating against.
- [ ] **Formalize the §20.5 rule specs** per `setupType`. The `WEIGHTS` table in §15 is the
      *draft* of these specs — promote it to explicit trend/pattern/stop/min-R:R rules per setup,
      versioned (`pullback_ema_v1`), so a weight change bumps a version and old backtests stay
      comparable.
- [ ] **Corporate-action-adjusted price series** (§20.4) for every backtest. Splits, bonuses,
      major dividends. SL/target/R metrics computed on adjusted prices, same as live.
- [ ] **Acknowledge survivorship bias** in the UI (§20.4): v1 uses today's NSE 500 membership
      applied historically. Results read better than reality. Label it on every Lab output.

---

## 1. Seed-constant inventory — everything to calibrate

Every value below is a **seed**. Column "Pass criterion" is what must be true before it goes live.

### 1.1 Indicator (Confidence) Score — §15

| Constant | Seed | Calibrate how | Pass criterion |
|---|---|---|---|
| `WEIGHTS` per setup type (5 sub-scores) | hand-set table | Per-setup param sweep, then portfolio walk-forward | Higher-score deciles show higher out-of-sample expectancy (monotonic-ish) |
| Trend sub: ADX cap 40, EMA-stack 40/20 pts, MACD/Supertrend 15 each | seeds | Sweep ADX cap, stack points | Stable across 40↔45 ADX, 35↔45 stack pts (not fragile) |
| Momentum sub: RSI bands (45–55 pullback / 55–68 breakout), RS map `(rs−0.9)/0.3` | seeds | Sweep RSI bands, RS anchor | Band edges don't swing performance > ~10% |
| Volatility sub: `bbWidth < 0.08`, ATR% 1–4 | seeds | Sweep compression threshold | Compression cut tracks real breakout follow-through |
| Volume sub: ratio 1.0/1.3/1.8, delivery 50% | seeds | Sweep ratio bands | Direction holds (expansion helps breakouts, dry-up helps pullbacks) |
| Structure sub: headroom `ratio×30`, `priceVsEma20 < 3%` | seeds | Sweep ratio scale, extension cut | R:R headroom predicts realized R |

### 1.2 Score blend & gates

| Constant | Seed | Location | Pass criterion |
|---|---|---|---|
| Blend weights 0.6 indicator / 0.4 Vision | guess | §7 Agent 2b | The 0.4 only justified once Vision deciles prove out (see §2) |
| Continuous Vision veto threshold 40 | guess | §7 Agent 2b | Sub-40 setups genuinely underperform out-of-sample |
| Min R:R 1.5 (T1, gating) | guess | §20.5 | Trades below it actually have negative expectancy |
| Final-score / MR threshold for placement (50) | guess | Workflow 2 | Placement cut maximizes expectancy net of skipped winners |

### 1.3 Watchlist Score — §15

| Constant | Seed | Pass criterion |
|---|---|---|
| Weights 40 trend / 30 RS / 30 proximity | seeds | Ranking correlates with which watchlist names actually set up next |
| Proximity signals (`bbWidth<0.06`, RSI 45–58, `priceVsEma20<2`), ×33 bucketing | seeds | Replace coarse ×33 with a gradient if buckets lose signal (C3) |
| RS lookback 63 trading days | fixed choice | Confirm 3-month is the horizon that predicts; test 1m/6m |

### 1.4 Morning Readiness penalties — §15

| Constant | Seed | Pass criterion |
|---|---|---|
| Nifty-gap −15 (gap > 0.8%), BSE −30 (hard veto), VIX −10 (move > 15%) | seeds | Penalized setups underperform un-penalized by ≥ the penalty's implied edge |
| Max combined penalty −40 | seed | Cap doesn't mask a genuinely dead setup |

### 1.5 Regime gate — §7 Agent 1

| Constant | Seed | Pass criterion |
|---|---|---|
| `classifyRegime`: niftyVs200ema ±1%, ADX ≥ 20, VIX < 18 | seeds | risk_off periods really do have worse long expectancy |
| `REGIME_POLICY`: riskMultiplier 0.5 (neutral), maxOpen 5/8, mrThreshold 50/60 | seeds | **Gating reduces max drawdown** without killing total return (the whole point) |

### 1.6 Exit policy — §15

| Constant | Seed | Pass criterion |
|---|---|---|
| `breakevenAtR` 1.0 | seed | Breakeven-at-1R improves expectancy vs no-breakeven (compare both) |
| `partialAtR` 1.6 (= T1), `partialFraction` 0.5 | seeds | 50%-at-T1 beats all-out and let-it-ride on *your* trade set |
| `atrTrailMult` 2.5 | seed | Trail captures trend without premature shake-out; sweep 2.0–3.5 |
| `timeStopSessions` 10 | seed | Dead-trade exit frees capital without cutting slow winners |

### 1.7 Execution safety — §14

| Constant | Seed | Pass criterion |
|---|---|---|
| SL buffer `max(2.5·ATR, 1%)` below trigger | seed | Fills through realistic gap-downs in the historical set; not so wide it invites bad fills |
| Stop-distance floor 1.5·ATR / ceiling 8% | seeds | Floor stops the "tight stop → oversized → shaken out"; ceiling rejects non-swings |

### 1.8 Risk & portfolio — §20.2 / §20.3

| Constant | Seed | Pass criterion |
|---|---|---|
| `MAX_PORTFOLIO_HEAT_PCT` 6 | seed | Stress-test under a **correlated gap-down** (A4) — real worst case, not the independent 6% |
| `MAX_OPEN_POSITIONS` 8, `MAX_ACTIVE_SETUPS` 10 | seeds | Cognitive cap holds; concurrency doesn't degrade win rate |
| Loss caps 3R/6R/10–12R | seeds | Caps cut tail loss without clipping normal variance |
| Sector cap 25% + theme/correlation cap (D1 refinement) | seed + TODO | Define correlated theme clusters; cap the cluster, not just the GICS sector |

---

## 2. Open design assumptions — validate the SHAPE, not just the numbers

These are bets baked into the architecture. Numbers alone won't tell you if they're right.

- [ ] **Vision's role.** v5 keeps Vision **veto-first** (the continuous veto) and flags the 40%
      blend as un-earned. Phase 4 test: bucket realized R by **Vision score decile**. Only promote
      Vision from veto to a positive score contributor if high-Vision deciles out-perform out-of-
      sample. If they don't, keep it veto-only and set the blend weight toward 0.
- [ ] **Entry mechanism per setup (B4).** v5 still uses one `price ≥ entryZoneHigh` buy-stop for
      all setups and only *notes* it should be per-type. Backtest buy-stop-through-zone vs
      buy-limit-at-support **per setup type**; pick the mechanism per type and write it into the
      §20.5 spec.
- [ ] **Regime hard-gate actually helps.** Compare full-system DD/return **with vs without** the
      gate. If risk_off blocking doesn't cut drawdown, the thresholds are wrong — not the idea.
- [ ] **Breakeven timing.** Strict-EOD vs intraday-touch (News-Agent 30-min). Which has better
      expectancy on your trade set? v5 recommends intraday; prove it.
- [ ] **Partial fraction.** 50% at T1 is a default. Test 33% / 50% / 67% and all-out.
- [ ] **Heat independence (A4).** Build the correlated-gap-down stress scenario explicitly and
      confirm the buffered SL + regime gate keep realized loss inside your hard drawdown line.

---

## 3. Calibration order (dependency-first — don't calibrate out of order)

1. **Costs, slippage, risk config** (§0) — everything downstream is fantasy without these.
2. **§20.5 rule specs** per setup — defines the trade set you calibrate against.
3. **Indicator sub-scores + weights** (1.1) — single-instrument first, then portfolio.
4. **Vision deciles** (§2) — decide veto-only vs contributor *before* trusting the blend.
5. **Regime thresholds + policy** (1.5 / §2) — does gating cut DD?
6. **Exit config** (1.6 / §2) — breakeven / partial / trail / time-stop.
7. **Score blend + placement thresholds** (1.2) — last, since they sit on top of all the above.
8. **Whole-system walk-forward** — in-sample vs out-of-sample, vs the performance targets.

Each stage feeds the next. Calibrating exits (6) before fixing costs (1) means re-doing 6.

---

## 4. Validation guardrails — the go-live red flags

A constant is **not** ready for live if **any** of these is true:

- [ ] It's still at its seed value.
- [ ] Its backtest was run **before** costs + slippage were filled.
- [ ] It was tuned on **< 100 trades** (§20.8 minimum sample).
- [ ] It was validated **in-sample only** (no walk-forward / out-of-sample).
- [ ] It's **fragile** — a small parameter change produces a large performance swing (§20.6
      heatmap flags "likely overfit").

---

## 5. Go-live gate (Definition of Done for Phase 4)

Ship the live constants only when, on **out-of-sample / walk-forward** segments, after costs:

- [ ] Win rate **40–50%**, average **~2R** on winners (§20.8).
- [ ] Max drawdown **≤ 10%**; never breaches the **≤ 15%** hard line.
- [ ] **≥ 100 trades** per configuration evaluated.
- [ ] Regime gate demonstrably **reduces drawdown** vs the un-gated system.
- [ ] Exit policy demonstrably **beats** the naive single-trail it replaced.
- [ ] No parameter flagged fragile.
- [ ] Vision's weight matches its proven decile edge (veto-only if unproven).

Until every box is checked, treat each v5 number as a **hypothesis under test** — exactly what the
freeze made it. Phase 5 (Ch.21) is where these stop being hand-tuned and start re-calibrating
themselves from accumulated live outcomes.
