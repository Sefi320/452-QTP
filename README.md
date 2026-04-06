# 452-QTP — Clustering-Based Pairs Trading

**FIN 452 Quantitative Trading Project | University of Alberta | Winter 2026**

---

## Strategy Overview

This project implements a two-layer pairs trading strategy on the 11 S&P 500 Select Sector SPDR ETFs plus SPY as both benchmark and tradeable asset.

The core idea: sector ETFs are highly correlated over the long run, but diverge in the short run. When a pair's spread deviates far enough from its historical norm, it is likely to revert — that reversion is the trade.

---

## Layer 1: Monthly Pair Selection (Clustering)

Every month, three mean-reversion features are computed for each of the 66 possible pairs:

- **Half-life** — how quickly the spread reverts to its mean (from an AR(1) regression)
- **Hurst exponent** — structural persistence of the spread (H < 0.5 = mean-reverting)
- **Spread volatility** — magnitude of spread movements

All 66 pairs are clustered in this 3D feature space using k-means and hierarchical clustering (Ward linkage). The optimal number of clusters is selected by maximizing the average silhouette score across both methods. The tradeable cluster is the one with the lowest combined rank of half-life and Hurst exponent — i.e., pairs that revert fast and have mean-reverting structure.

The top 10 pairs from the tradeable cluster are selected, weighted by inverse volatility parity (lower spread vol = larger position).

---

## Layer 2: Daily Signal Generation (Kalman Filter)

For each active pair, a Kalman Filter tracks the dynamic hedge ratio between the two legs in real time. Unlike OLS, the KF allows the relationship to evolve daily. It outputs:

- A time-varying hedge ratio
- A dynamic spread estimate
- An innovation (prediction error) that flags structural breaks

A trade fires when two conditions are simultaneously true:
1. The pair is in the current month's tradeable cluster
2. The KF spread deviates beyond 1.5 standard deviations from its rolling mean

Long the undervalued leg, short the overvalued leg.

---

## Exit Rules

- **Take profit**: spread returns to KF mean estimate
- **Time stop**: position open longer than 2x the pair's half-life
- **Innovation override**: KF prediction errors spike for 5+ consecutive days (structural break)

---

## Rebalancing

Monthly. Pairs that leave the tradeable cluster are force-closed. Pairs that remain are resized to new inverse vol parity weights.

---

## Benchmark

SPY (SPDR S&P 500 ETF Trust). Primary performance metrics: cumulative return, Omega ratio, maximum drawdown, Sharpe ratio.

---

## Files

| File | Description |
|------|-------------|
| `QTP-Soumer.qmd` | Main Quarto document — full strategy implementation |
| `QTP-Soumer.html` | Self-contained HTML output (all resources embedded) |
| `assets/mental_model.svg` | Strategy mental model diagram |
| `mental_model.mmd` | Mermaid source for the mental model |
| `claude-macro-rotation-experiment.qmd` | Side experiment — macro regime rotation strategy. Generated entirely by Claude AI, not part of the graded submission. |
| `claude-macro-rotation-experiment.html` | HTML output of the Claude-generated experiment |

---

## Training / Test Split

- **Training**: June 2000 – December 2024
- **Testing**: January 2025 – March 2026
