# Metric Triangle: Anti-Gaming Framework

## The Problem

Single metrics can be gamed. Every performance metric yields to a targeted trick:
- **Sharpe**: clip returns tighter → std drops faster than mean → Sharpe rises artificially
- **Lo-adjusted Sharpe**: clip reduces autocorrelation → Lo correction factor shrinks → Lo rises
- **IC**: clip removes rank noise → rank correlation appears cleaner → IC rises
- **Omega**: concentrate on cherry-picked easy days → gain/loss ratio inflated
- **MaxDD**: reduce position size → drawdown falls but so does all signal quality

The core failure: **optimizing one number gives the optimizer a degree of freedom to sacrifice everything else.** The solution is to require simultaneous improvement across three mathematically orthogonal spaces.

## The Solution: Three Orthogonal, Leverage-Invariant Metrics

```
          Lo-adjusted Sharpe
          (optimization target — ratio space)
               /           \
              /             \
     IC (guardrail)    Omega (guardrail)
     (rank space)      (distribution shape space)
              \             /
               \           /
          15-test Validation Gate
          (MaxDD/PnL absolute gates + CPCV/DSR/OOS)
```

## Why These Three

Every metric lives in one of three mathematical spaces. Metrics within the same space are correlated; metrics across spaces are independent.

**Lo-adjusted Sharpe** (ratio space): Sharpe corrected for serial autocorrelation. The Lo correction factor is `sqrt(1/cf)` where `cf = 1 + 2*sum(rho_k)` over significant lags. If a strategy holds positions for days, the correction penalizes the serial correlation. Catches: serial-correlation inflation of raw Sharpe. Blind spot: clipping (still a ratio, still fooled).

**IC** (rank space): Spearman rank correlation between position size and PnL outcome, on active days only. Because IC uses ranks, not magnitudes, leverage scaling doesn't change it. Catches: concentration gaming (zeroing out weak-signal days while doubling on easy days improves Lo but collapses IC because prediction *quality* doesn't improve). Blind spot: systematic overfit where rank order improves in-sample.

**Omega** (distribution shape space): sum of gains / sum of losses on active days. Shape is preserved under uniform leverage scaling but destroyed by clipping (gains are capped harder than losses at any fixed clip boundary). Catches: the clipping trick — clipping daily returns at ±2% truncates the gain tail, lowering Omega even as Sharpe rises. Blind spot: concentrated in a narrow band of gains and losses (but IC catches the rank degradation).

**Why one per space is sufficient:** All metrics within the same space are correlated — if Lo rises, Sharpe necessarily rises (Lo ≤ Sharpe by construction). Adding Sortino when Lo is already present adds no new information. One representative per space is both necessary and sufficient.

## Cross-Check Matrix

| Manipulation | Lo (ratio) | IC (rank) | Omega (shape) | Caught by |
|---|:-:|:-:|:-:|---|
| Clip returns tighter | up | up | **down** | Omega drops — gains clipped harder than losses |
| Serial correlation | stays (Lo corrects) | stays | stays | Lo doesn't rise despite raw Sharpe rising |
| Concentration on easy days | up | **down** | up | IC collapses — prediction quality unchanged |
| Overfit in-sample | up | up | up | 15-test gate (CPCV, DSR, OOS/IS) |
| Position cap (clip to 1.0) | distorted | distorted | distorted | Non-linear break detected across all three |
| Genuine improvement | up | up | not worse | All pass |

**Honest limitation:** In-sample overfitting improves all three triangle metrics simultaneously. The triangle alone cannot catch it. The 15-test validation gate is the required second layer of defense.

## Leverage Invariance

Verified empirically: uniform scaling by 1x, 1.5x, 2x, 3x produces identical triangle values.

| Metric | 1x | 1.5x | 2x | 3x | Invariant? |
|--------|-----|------|-----|-----|:---:|
| Lo-adjusted Sharpe | L | L | L | L | YES |
| IC | I | I | I | I | YES |
| Omega | O | O | O | O | YES |
| PnL | P | 1.5P | 2P | 3P | no |
| MaxDD | D | 1.5D | 2D | 3D | no |

**Why this matters:** Strategy comparisons must measure signal quality, not bet size. MaxDD and PnL scale with leverage and therefore belong in absolute gate thresholds — not in relative experiment comparisons.

**Position cap caveat:** Clipping positions to a maximum (e.g., `min(pos, 1.0)`) is a non-linear transformation that breaks leverage invariance. All three triangle metrics change because the signal's statistical properties are distorted — this is the correct behavior. The triangle detects it.

## The 38-Experiment Proof (Real Gaming Catches)

### Catch 1: Concentration Gaming (xcorr scaling 2.0/0.0)

Zeroed positions on low-correlation days, doubled on high-correlation days. Capital concentrated on "easy" days.

| Scale | Lo-adj | IC | Sharpe | Verdict |
|-------|--------|----|--------|---------|
| 1.25/0.75 | 2.302 | 0.441 | 4.218 | Baseline |
| 1.75/0.25 | 2.367 | 0.569 | 4.414 | Optimal |
| **2.00/0.00** | **2.375** | **0.403** | **4.425** | **IC crashed — CAUGHT** |

Lo and Sharpe both went up. IC collapsed 29% because prediction quality didn't improve — only capital concentration changed. The triangle caught what Lo and Sharpe both missed.

### Catch 2: Over-Aggressive Contrarian (RSI scaling 0.50/1.50)

| Scale | Lo-adj | IC | Sharpe | Verdict |
|-------|--------|----|--------|---------|
| 0.60/1.40 | 2.482 | 0.563 | 4.289 | Baseline |
| **0.50/1.50** | **2.488** | **0.568** | **4.221** | **Sharpe dropped — CAUGHT** |

Lo and IC both rose slightly. Raw Sharpe dropped 0.068, exceeding the 0.05 tolerance. The guardrail caught what the optimization target and IC both missed.

## KEEP/DISCARD Rule

```
KEEP if:
  Lo-adjusted Sharpe improved (vs baseline)
  AND IC >= baseline - 0.005
  AND Omega >= baseline - 0.10
  AND 15-test PASS
  AND PnL > floor (100% crypto / 30% equity)
  AND Sharpe/Lo ratio < 2.5
DISCARD: everything else
```

After a KEEP, the new metrics become the baseline for subsequent experiments.

## Additional Guards

**PnL floor:** Total cumulative PnL > +100% (crypto) / +30% (equity). A strategy with Sharpe 5.0 and +2% total return is worthless. This is an absolute gate, not a relative comparison.

**Sharpe/Lo ratio cap:** `raw_sharpe / lo_adjusted_sharpe < 2.5`. When this ratio is large, raw Sharpe is inflated by serial correlation. The strategy is holding positions too long, not generating real alpha.

**IC stability floor:** Monthly IC positive fraction >= 50%. Prevents strategies where IC is driven by a few extreme months rather than consistent prediction quality.

**Relative PnL drop guard:** If the current best KEEP has PnL X, the next experiment must produce at least `0.80 * X`. Prevents death-by-a-thousand-cuts where successive "improvements" each stay above the absolute floor but erode total return toward zero.

## Formulas (Implement Locally)

```python
import numpy as np
from scipy.stats import spearmanr

def lo_adjusted_sharpe(daily_pnl, periods_per_year=252):
    """Sharpe corrected for serial autocorrelation (Lo 2002)."""
    n = len(daily_pnl)
    sharpe_raw = daily_pnl.mean() / daily_pnl.std() * np.sqrt(periods_per_year)
    # Compute correction factor from significant autocorrelations
    from statsmodels.stats.stattools import durbin_watson
    rho_sum = 0.0
    for lag in range(1, min(n // 5, 12)):
        rho_k = np.corrcoef(daily_pnl[lag:], daily_pnl[:-lag])[0, 1]
        # Include lag if |rho_k| > 2/sqrt(n) (approx significance threshold)
        if abs(rho_k) > 2 / np.sqrt(n):
            rho_sum += rho_k * (1 - lag / n)  # Newey-West weighting
    cf = 1 + 2 * rho_sum
    if cf <= 0:
        return sharpe_raw  # Correction undefined, fall back
    return sharpe_raw * np.sqrt(1 / cf)

def information_coefficient(positions, daily_pnl):
    """Spearman rank corr between |position| and PnL sign on active days."""
    active = positions != 0
    if active.sum() < 10:
        return np.nan
    ic, _ = spearmanr(np.abs(positions[active]), daily_pnl[active])
    return ic

def omega_ratio(daily_pnl, threshold=0.0):
    """Gain/loss asymmetry on active days (Shadwick & Keating 2002)."""
    gains = daily_pnl[daily_pnl > threshold]
    losses = daily_pnl[daily_pnl < threshold]
    if len(losses) == 0 or losses.sum() == 0:
        return np.inf
    return gains.sum() / abs(losses.sum())
```

**Notes:**
- `daily_pnl` = strategy returns (not clipped); clipping before this computation is the gaming vector
- IC uses `abs(positions)` on active days: large positions should predict large gains (not losses)
- Omega at threshold=0 is the standard form; some profiles use a hurdle rate instead
- The Lo correction is sensitive to the number of lags included — use significance testing, not a fixed window
