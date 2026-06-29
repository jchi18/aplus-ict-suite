# A+ Rejection Blocks — Backtest Findings

Faithful Python port of `aplus_rejection_blocks.pine` (identical detection + 5-factor
grading), run across 12 instrument/timeframe datasets (crypto 1h+1d, US equities/ETFs
1h+1d) via yfinance. Entry/exit rules added for the test (the indicator itself only
marks zones):

- **Entry:** later bar trades back into the zone → fill at proximal edge.
- **Stop:** distal edge (wick extreme) ± 0.1·ATR buffer.
- **Target:** fixed reward:risk. Same-bar stop+target ambiguity counted as a stop (pessimistic).
- **Invalidation:** close beyond distal edge before entry, or no tap within 40 bars.
- One trade per block. ~4bps commission + slippage modeled in the Pine strategy.

## 1. Does the grade predict edge? (2,896 pooled trades, RR=2)

| Grade bucket | n | Win% | PF | Expectancy |
|---|---|---|---|---|
| C (40–50) | 439 | 33.3 | 1.00 | −0.002R |
| B (50–65) | 1230 | 31.3 | 0.91 | −0.061R |
| A (65–80) | 1099 | 31.4 | 0.92 | −0.058R |
| **A+ (80+)** | **128** | **34.4** | **1.05** | **+0.031R** |

**The scoring engine works directionally:** only the A+ bucket is profitable; lower
grades have no edge. Filtering to A+ is doing real work, not cosmetic.

## 2. A+ blocks are a *reaction* edge, not a runner (RR sweep, A+ only, n=128)

| RR | Win% | PF | Expectancy | Total R |
|---|---|---|---|---|
| **1.0** | **57.8** | **1.37** | **+0.156R** | **+20** |
| 1.5 | 42.2 | 1.09 | +0.055R | +7 |
| 2.0 | 34.4 | 1.05 | +0.031R | +4 |
| 3.0 | 19.5 | 0.73 | −0.219R | −28 |
| 4.0 | 14.8 | 0.70 | −0.258R | −33 |

Price reliably bounces off an A+ block (high hit-rate) but the reaction is shallow —
**RR≈1 is optimal**; demanding 3R+ destroys the edge. Treat these as fade/scalp zones
or scale out near 1R, not as swing-trend entries.

## 3. Opposing-liquidity dynamic target — does it beat fixed RR? (A+ only, n≈128)

Target = nearest unbroken opposing swing (long → nearest swing high above entry; short → nearest swing low below). Tested pure, capped, and as a partial-runner hybrid.

| Exit | Win% | PF | Expectancy |
|---|---|---|---|
| **Fixed RR=1 (baseline)** | **57.8** | **1.37** | **+0.156R** |
| Opposing liquidity (minRR 1.0, cap 12) | 41.4 | 0.95 | −0.029R |
| Partial 0.7 @1R + BE + liq runner | 57.8 | 1.13 | +0.055R |
| Partial 0.7 @1R, no BE + liq runner | 57.8 | 1.04 | +0.017R |

**Verdict: the opposing-liquidity target underperforms.** A+ rejection blocks are a
shallow reaction — price reliably bounces but usually dies before reaching the next
liquidity pool, so runners give back to breakeven while the front-half profit is capped.
The best liquidity variant (+0.055R) is a third of the fixed-RR=1 edge.

The feature is shipped in the strategy as a selectable **Exit mode** (Fixed RR /
Opposing liquidity / Partial + liquidity runner) for your own per-instrument testing,
but the default stays **Fixed RR = 1** because that's what the data supports.

## 4. FVG & Order Blocks — same methodology, different edges

Detection + 5-factor grade ported for Fair Value Gaps (3-candle imbalance) and Order
Blocks (last opposing candle before a BOS). Same trade sim, 12 markets.

**Fair Value Gaps** (17,750 trades) — grade is FLAT across buckets (C/B/A/A+ all
~+0.06–0.08R). "A+ FVG" adds nothing; the edge is the gap itself. FVGs are CONTINUATION
zones — they prefer WIDER targets (RR=3 → +0.102R), opposite of rejection blocks.

**Order Blocks** (742 trades) — grade helps modestly (A+ +0.091R vs C +0.042R). Best
config of all three concepts: **A+ OB at RR=2 → PF 1.36, +0.212R** (54% win at 1R, 40% at 2R).

| Concept | Nature | Best exit | Best config |
|---|---|---|---|
| Rejection Block | Shallow reaction (fade) | RR≈1 | A+ only · PF 1.37 · +0.156R |
| Fair Value Gap | Continuation (robust, frequent) | RR 2–3 | grade-agnostic · PF 1.14 · +0.10R |
| Order Block | Continuation (selective) | RR≈2 | A+/B · PF 1.36 · +0.212R |

Takeaway: grade A+ filtering matters for **rejection blocks and order blocks**, not FVGs;
and the right target is exit-mode-specific per concept. Indicators are calibrated accordingly.

## 5. The combined suite (`aplus_confluence.pine`)

Backtested the suite **as it ships**: every A+ signal (score ≥ 80) from all three
concepts, each exited at *its own* validated RR (RB→1, FVG→2, OB→2), 12 markets ×
timeframes. Two views — pooled (every signal, unlimited capital) and a realistic
**sequential book** (one position at a time per market, later signals skipped while a
trade is open). `scratchpad/suite.py`.

| View | n | Win% | PF | Exp | Total R |
|---|---|---|---|---|---|
| RB alone (A+ @ RR1) | 128 | 57.8 | 1.37 | +0.156R | +20 |
| FVG alone (A+ @ RR2) | 1788 | 36.1 | 1.13 | +0.082R | +147 |
| OB alone (A+ @ RR2) | 99 | 40.4 | 1.36 | +0.212R | +21 |
| **Suite — pooled** | **2015** | **37.7** | **1.15** | **+0.093R** | **+188** |
| **Suite — sequential** | **1717** | **37.7** | **1.16** | **+0.098R** | **+169** |

- Capital contention is mild: 85% of pooled signals still fill one-at-a-time, and the
  sequential book is *slightly better* (+0.098R vs +0.093R) — concurrency isn't carrying
  the result. Median per-market max drawdown 12R, worst 25R.
- The book is dominated by FVGs (~89% of signals) so the blended PF (1.15) tracks FVG's,
  with RB/OB lifting per-trade expectancy. Net: a positive-expectancy, well-diversified
  book — but FVG volume drives it, not the rarer high-quality RB/OB setups.
- Per-market the edge is real but uneven: strong on AAPL 1d (PF 2.15), TSLA 1d (1.96),
  BTC 1d (1.62); negative on MSFT 1d, SPY 1d, QQQ 1h (small samples, 36–91 trades). This
  is expected — A+ density and trend character vary by instrument.

**Confluence (overlap) re-confirmed dead.** Tagging each signal by how many *other*
concept-types overlap it at the same price (same direction, no lookahead) and bucketing:

| Bucket (RR2) | n | Win% | PF | Exp |
|---|---|---|---|---|
| solo (0 overlap) | 16371 | 34.2 | 1.04 | +0.027R |
| +1 type overlap | 1133 | 33.0 | 0.99 | −0.010R |
| +2 types overlap | 77 | 26.0 | 0.70 | −0.221R |
| **A+ grade only** | **2015** | **36.2** | **1.13** | **+0.085R** |

Overlap monotonically *hurts* (more confirmation = worse), while the plain A+ grade
filter is the best single screen. So the suite gates on **grade, not confluence** — the
overlap marker stays OFF by default. "Stacked zones = stronger" is folklore; the data
says the opposite.

## 6. Timeframe study — does the edge survive intraday? (15m / 5m)

Extended the A+ suite below 1h. Crypto (BTC/ETH/SOL) pulled deep from Binance.US
(40k bars each per TF ≈ 140–420 days); stocks/ETFs from yfinance (capped at 60 days
intraday → tiny samples). `scratchpad/run_intraday.py`, `fetch_intraday.py`.

**Crypto is the clean apples-to-apples test** (same 3 assets, all timeframes). The
sequential-book expectancy degrades **monotonically** as the timeframe drops:

| TF | Crypto seq exp/trade | PF | Verdict |
|---|---|---|---|
| 1d | +0.34R (BTC) | 1.62 | best |
| 1h | +0.02 … +0.13R (all 3 positive) | 1.03–1.21 | works |
| **15m** | **−0.11 … −0.13R (all 3 negative)** | **0.82–0.85** | **loses** |
| **5m** | **−0.16 … −0.20R (all 3 negative)** | **0.72–0.78** | **loses badly** |

Pooled across all 15m datasets: PF 0.87, −0.088R (n=3832). At 5m: PF 0.82, −0.124R
(n=4281). Drawdowns also blow out — worst per-market max DD 107R (15m) / 189R (5m) vs
~25R on the daily book. **The zones get chopped to pieces by intraday noise on crypto.**

Stocks/ETFs at 15m/5m printed *positive* (e.g. NVDA 15m +1.19R, QQQ 5m +0.48R) — but
those are **12–50 trades over a single 60-day window**, far too small and too
regime-specific to trust. Suggestive, not evidence.

**Conclusion: the suite's edge lives on higher timeframes.** Daily is best, 1h is the
practical floor. On crypto, anything below 1h is net-negative — do not run the suite on
5m/15m crypto. Whether a *session-filtered* intraday stock variant works is an open
question that needs real multi-year intraday stock data to answer.

## Caveats
- Per-market samples are small by design (A+ is rare: 128 trades across years/12 markets).
- Intraday stock samples (15m/5m) are 60-day, single-regime → directional only, not proof.
- Forex (EURUSD) volume is unreliable → the volume factor is noise there; expect weaker results.
- No regime/session filter, no partial exits, no trade management — a floor, not a ceiling.
- Fixed-RR exit is naive; targeting opposing liquidity/structure likely improves it (parking lot).

## Reproduce
`scratchpad/backtest.py` (per-market), `validate.py` (grade buckets), RR sweep inline.
`scratchpad/suite.py` (combined book + sequential portfolio), `confluence.py` (overlap test).
`scratchpad/run_intraday.py` + `fetch_intraday.py` (15m/5m timeframe study, Binance.US + yfinance).
Use `aplus_rejection_blocks_strategy.pine` in TradingView's Strategy Tester for native,
per-instrument backtesting (default RR=1, longs+shorts).
