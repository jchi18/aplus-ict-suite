# A+ ICT Suite — TradingView (Pine v6) indicators

High-end ICT detection tools, **A+ only** by default. Every concept is graded
0–100 by its own validated factor set; only A+ (score ≥ 80) zones are drawn.

## Indicators
| File | What it does |
|---|---|
| **`aplus_confluence.pine`** | **The suite — all three concepts in one indicator**, each distinctly marked (RB = teal, FVG = blue, OB = amber; bull = bright/solid, bear = dark/dashed; ▲/▼ in label). Per-concept toggles, unified dashboard/legend, alerts. |
| `aplus_rejection_blocks.pine` | Rejection Blocks (swing-wick rejection zones) |
| `aplus_rejection_blocks_strategy.pine` | Strategy-Tester version of the RB indicator (Fixed RR / opposing-liquidity / partial-runner exits) |
| `aplus_fvg.pine` | Fair Value Gaps (3-candle imbalance) |
| `aplus_order_blocks.pine` | Order Blocks (last opposing candle before a Break of Structure) |

## How to use
1. TradingView → **Pine Editor** → paste a file's contents.
2. **Add to chart.**
3. Defaults to **A+ only**. Settings let you tune grade threshold, weights, calibration, and display.

## What the backtests say (see [`BACKTEST.md`](BACKTEST.md))
- **Grade matters** for Rejection Blocks and Order Blocks; for FVGs the edge is the gap itself (grade is ~flat), so A+ shows few FVGs by design.
- **Best targets differ by concept:** RB ≈ RR 1 (reaction), FVG & OB ≈ RR 2 (continuation).
- **Confluence (overlapping zones) does NOT help** — it underperforms; grade beats confluence. The overlap marker ships OFF.
- **Timeframe:** edge lives on higher TFs. Daily best, 1h is the practical floor; **below 1h on crypto it's net-negative** — don't run it on 5m/15m crypto.

> Backtests are a faithful Python port of the Pine detection logic, run across 12 markets/timeframes (and deep crypto intraday). They're a realistic baseline for the zones' raw edge — fixed-RR, no trade management. Not financial advice.
