# QTP Design Spec — Clustering-Based Pairs Trading Strategy
**Date**: 2026-04-04
**Course**: FIN 452, UofA Winter 2026
**Due**: 2026-04-06 at 6pm EDM

---

## 1. Problem Statement

Design and implement a quantitative trading strategy in R that:
- Uses clustering as the primary signal driver
- Uses the Kalman Filter in conjunction with clustering to trigger trades
- Trades S&P 500 Select Sector SPDR ETFs + SPY
- Beats SPY on a risk-adjusted basis
- Is fully implemented in Quarto (.qmd) with daily trading and monthly rebalancing

---

## 2. Asset Universe

- **12 assets**: SPY + XLB, XLC, XLE, XLF, XLI, XLK, XLP, XLRE, XLV, XLU, XLY
- **66 possible pairs**: C(12,2)
- **Data caveat**: XLC available from 2018, XLRE from 2015 — pairs involving these assets excluded before their respective inception dates
- **Data frequency**: Daily
- **Data source**: Yahoo Finance via `tidyquant::tq_get()` — no local files
- **Training period**: June 2000 – December 2024
- **Testing period**: January 2025 – March 2026

---

## 3. Strategy Architecture

### Mental Model Summary

The strategy has two layers and an override mechanism:

1. **Monthly (Layer 1)**: Cluster all 66 pairs by spread mean-reversion properties → identify tradeable pairs → size positions
2. **Daily (Layer 2)**: Kalman Filter tracks each active pair's spread → combined with cluster membership to trigger trade entry/exit
3. **Override**: KF innovation spikes trigger mid-month force-exit

Trade fires only when **both** conditions are true:
- Pair is in the tradeable cluster (Layer 1)
- KF spread has deviated significantly from KF mean estimate (Layer 2)

---

## 4. Layer 1 — Monthly Pair Selection

### Feature Computation
For each of the 66 pairs, compute the following over a **rolling trailing window** (window length determined via optimization on training data):

| Feature | Description | Direction |
|---------|-------------|-----------|
| **Half-life** | ln(2)/θ from AR(1) regression on spread: `ΔS_t = α + β·S_{t-1} + ε`. Half-life = -ln(2)/β | Lower = faster reversion = better |
| **Hurst exponent** | Measures autocorrelation structure of spread. H < 0.5 = mean-reverting, H = 0.5 = random walk, H > 0.5 = trending | Lower = more mean-reverting = better |
| **Spread volatility** | Rolling standard deviation of spread returns | Sweet spot — not too low (unprofitable), not too high (too noisy) |

### Clustering Methodology
1. Scale all three features before clustering
2. Run **k-means clustering** with k search range k = 2 to k = 12
3. Run **hierarchical clustering** with Ward linkage
4. Validate k selection with **elbow method** (inertia vs k) and **silhouette score**
5. If k-means and hierarchical clustering agree on cluster assignments → high confidence in groupings
6. Select optimal k as the value just before diminishing returns on the elbow curve

### Tradeable Cluster Selection
1. For each cluster, compute centroid values on all three features
2. Rank clusters by combined (half-life rank + Hurst rank) — lowest total rank wins
3. Verify post-hoc that the winning cluster's spread vol centroid is reasonable (not near zero, not extreme)
4. Winning cluster = tradeable pair universe for the coming month

### Position Sizing
- Take top 10 pairs from the tradeable cluster
- Size using **inverse volatility parity**: weight_i = (1/spread_vol_i) / Σ(1/spread_vol_j)
- Lower spread volatility = larger position (explicit risk management)
- Weights normalized to sum to 1 across all 10 pairs (fully invested at all times)
- Within each pair: KF β_t determines the long/short split

### Rolling Window Optimization
- Window length is a parameter optimized on training data
- Use `expand.grid()` to generate candidate window lengths
- Use `foreach` + `doParallel` for multi-core grid search
- Select window length that maximizes OmegaSharpeRatio on training set

---

## 5. Layer 2 — Daily Signal Generation

### Kalman Filter Setup (per active pair)

**State equation** (hedge ratio evolves as random walk):
```
β_t = β_{t-1} + η_t,   η_t ~ N(0, Q)
```

**Observation equation**:
```
price_asset1_t = β_t × price_asset2_t + α + ε_t,   ε_t ~ N(0, R)
```

**Daily outputs**:
- `β_t`: dynamic hedge ratio (sizes the short leg)
- `spread_t = price_asset1_t - β_t × price_asset2_t`: current spread level
- `spread_mean_t`: KF-estimated mean of the spread
- `innovation_t`: prediction error (how surprised the model was today)

### Trade Trigger (Clustering + KF in conjunction)

**Entry condition** — BOTH must be true:
1. Pair is currently in the tradeable cluster (Layer 1 assignment)
2. `|spread_t - spread_mean_t|` exceeds a threshold (spread has deviated from KF mean)

**Direction**:
- `spread_t > spread_mean_t`: asset1 overvalued → SHORT asset1, LONG asset2 (hedge ratio β_t units)
- `spread_t < spread_mean_t`: asset1 undervalued → LONG asset1, SHORT asset2 (hedge ratio β_t units)

---

## 6. Exit Rules

Two rules only — no additional thresholds or percentage targets:

| Rule | Condition | Action |
|------|-----------|--------|
| **Take profit** | `spread_t` returns to KF mean estimate (`spread_t ≈ spread_mean_t`) | EXIT position |
| **Time stop** | Days in trade > 2 × half-life of the pair | EXIT regardless of P&L |

**Rationale**: If the spread hasn't reverted within 2× its estimated half-life, the model's statistical assumptions are violated. The time stop is motivated directly by the OU model, not an arbitrary threshold.

---

## 7. Override Mechanism

**Condition**: KF innovations (`innovation_t`) spike and remain elevated for N consecutive days

**Action**: Force-exit the pair's position immediately, regardless of monthly cluster assignment or spread state

**Purpose**: Handles mid-month structural breaks (sector-specific shocks, macro regime changes) that the monthly clustering cannot anticipate in real time.

---

## 8. Monthly Rebalancing

Occurs at month-end. Two-step process:

**Step 1 — Update pair universe:**
- Re-run Layer 1 clustering on updated rolling window data
- For each open position:
  - Pair **stays in tradeable cluster** → resize to new inverse vol parity weight
  - Pair **exits tradeable cluster** → force-close position

**Step 2 — Position sizing refresh:**
- Recompute inverse vol parity weights for the updated top 10 pairs
- Adjust notional allocations accordingly

---

## 9. Performance Metrics

Report all metrics against SPY buy-and-hold benchmark:

| Metric | Function |
|--------|----------|
| Cumulative returns | `cumprod(1 + ret)` |
| OmegaSharpeRatio | `PerformanceAnalytics::OmegaSharpeRatio()` (preferred over raw Omega — comparable to Sharpe) |
| Sharpe ratio | `PerformanceAnalytics::SharpeRatio()` |
| Maximum drawdown | `PerformanceAnalytics::table.Drawdowns()` |
| Full trade stats | `RTL::tradeStats()` |

---

## 10. Document Structure (.qmd)

1. **Executive Summary** — mermaid mental model diagram (strategy overview)
2. **Universe & Data** — assets, API data loading, inception date handling for XLC/XLRE
3. **Layer 1 — Pair Selection** — feature computation, clustering methodology, k selection, tradeable cluster identification
4. **Layer 2 — Signal Generation** — Kalman Filter setup, outputs, trade trigger logic
5. **Exit Rules & Override** — time stop, take profit, KF innovation override
6. **Monthly Rebalancing** — cluster-conditional rebalancing logic, position sizing refresh
7. **Parameter Optimization** — rolling window optimization, grid search methodology, visualization
8. **Performance Results** — all metrics vs SPY, training and testing periods separately
9. **Conclusion** — strategy assessment, limitations, potential improvements

---

## 11. Key Design Decisions & Rationale

| Decision | Rationale |
|----------|-----------|
| Cluster pairs (not ETFs) | Pairs have measurable mean-reversion properties; clustering on those properties directly identifies tradeable pairs |
| Half-life + Hurst + vol as features | Each captures a different dimension: speed, structure, profitability. Orthogonal features. |
| No ADF pre-filter | ADF has low power in finite samples and is redundant with Hurst + half-life. Clustering handles pair selection. |
| KF for hedge ratio | Dynamically adapts β_t as sector relationships evolve. Compensates for monthly feature staleness. |
| Clustering + KF in conjunction | Clustering gates which pairs are tradeable; KF determines when to execute. Both required to fire a trade. |
| Inverse vol parity sizing | Industry standard. Equalizes dollar risk contribution across pairs. No single pair dominates P&L. |
| Time stop (2× half-life) | Statistically motivated by OU model. If spread hasn't reverted in 2× expected time, model assumptions are violated. |
| No % stop-loss | Stop-losses are counterproductive in mean-reverting strategies — they exit trades right before reversion. KF innovation override handles structural breaks instead. |
| Monthly rebalancing conditional on cluster | Only force-close if pair exits the tradeable cluster. Avoids interrupting profitable live trades unnecessarily. |
| SPY as both benchmark and tradeable asset | SPY pairs with defensive sectors (XLU, XLP, XLV) may show strong mean-reversion. Let clustering decide. |

---

## 12. Assumptions

- **No transaction costs** — no bid/offer spreads, commissions, or slippage
- **No margin calls** — short positions are unconstrained
- **Full shorting availability** — all 12 assets can be shorted at any time

## 13. Open Implementation Questions

- KF package: **`dlm`** (primary) — `dlmFilter()` for sequential daily updating. Fall back to `FKF` if performance becomes an issue across many pairs.
- Exact KF innovation threshold for override — to be calibrated on training data
