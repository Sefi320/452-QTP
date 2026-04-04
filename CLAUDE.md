# QTP — Quantitative Trading Project (FIN 452, UofA Winter 2026)

## Project Overview

Individual assignment worth 20% of grade. Due 2026-04-07 at 6pm EDM with a 10-minute in-class presentation.

**Goal**: Develop and implement a clustering-based pairs trading strategy on S&P 500 Select Sector SPDR ETFs using R + Quarto. Beat SPY on a risk-adjusted basis.

---

## About the Student

- UofA FIN 452 student, Winter 2026
- Comfortable with R and Python; **this project is R only**
- Prior quantitative trading experience from FIN 440
- Familiar with tidymodels, xts, TTR, Quarto
- Uses Obsidian for notes
- Prefers concise, direct responses — no filler, no restating what was said
- Do not add unnecessary rules, parameters, or complexity to the strategy
- Pushes back on bad ideas — engage critically, not sycophantically
- **Do NOT edit, suggest edits to, or rewrite any code unless explicitly asked to do so**

---

## Hard Requirements (from professor)

- **Language**: R only. Any Python code or structure will be penalized.
- **Document format**: `.qmd` + self-contained HTML (all resources embedded). Also submit `context.md`.
- **Data**: Loaded via APIs only — no local files (no CSV, Excel, etc.)
- **Workflow**: Must follow the `tidymodels` workflow
- **Clustering**: Must be the primary method driving signals from the indicator mental model
- **Kalman Filter**: Must be used in conjunction with clustering to trigger trade signals
- **Mental model**: Must be visualized using mermaid.ai charts — code must match the mental model
- **GitHub**: Public repo, push daily. A single commit on submission day = 5-point penalty
- **Benchmark**: SPY (SPDR S&P 500 ETF Trust)
- **Trading frequency**: Daily
- **Portfolio rebalancing**: Monthly
- **Training period**: June 2000 – December 2024
- **Testing period**: January 2025 – March 2026

---

## Asset Universe

| Ticker | Sector | Note |
|--------|--------|------|
| SPY | S&P 500 benchmark + tradeable | — |
| XLB | Materials | — |
| XLC | Communication Services | Available from 2018 only |
| XLE | Energy | — |
| XLF | Financials | — |
| XLI | Industrials | — |
| XLK | Technology | — |
| XLP | Consumer Staples | — |
| XLRE | Real Estate | Available from 2015 only |
| XLV | Health Care | — |
| XLU | Utilities | — |
| XLY | Consumer Discretionary | — |

Total: 12 assets → 66 possible pairs (C(12,2))

XLC and XLRE pairs are excluded before their respective inception dates.

---

## Strategy: Clustering-Based Pairs Trading

### Core Logic
- Go long the undervalued leg, short the overvalued leg of a pair
- Shorting is allowed; no margin call constraints assumed
- Capital is fully invested at all times
- Top 10 pairs traded simultaneously

### Layer 1 — Monthly Pair Selection

1. For each of the 66 pairs, compute spread features using a rolling window (window length determined via optimization):
   - **Half-life** (ln(2)/θ from AR(1) regression on spread): how fast the spread reverts
   - **Hurst exponent**: structure of spread movements — want H < 0.5
   - **Spread volatility**: magnitude of spread movements
2. Cluster all 66 pairs in this 3D feature space using:
   - **k-means clustering**
   - **Hierarchical clustering** (Ward linkage)
   - Validate with **elbow method** + **silhouette score**
   - Search range: k = 2 to k = 12
3. Select tradeable cluster: rank clusters by centroid values — lowest combined (half-life + Hurst) rank wins. Verify spread vol of winning cluster is reasonable post-hoc.
4. Take top 10 pairs from tradeable cluster by silhouette score ranking... wait, use **inverse volatility parity weighting** for position sizing (not silhouette — industry standard).

### Layer 2 — Daily Signal Generation (per active pair)

**Kalman Filter** runs daily for each active pair:
- State: dynamic hedge ratio β_t (random walk transition)
- Observation: price_asset1 = β_t × price_asset2 + spread
- Outputs: β_t (hedge ratio), spread level, spread mean estimate, innovation (prediction error)

**Trade trigger** (clustering + KF in conjunction):
- Pair must be in the tradeable cluster (Layer 1) — necessary condition
- KF spread must deviate significantly from KF mean estimate — sufficient condition
- Both conditions must be true simultaneously to enter a trade

### Entry
- Long undervalued leg, short overvalued leg
- Hedge ratio = β_t from Kalman Filter
- Position size = inverse vol parity weight × total capital allocated to that pair

### Exit Rules (keep it simple — no excessive rules)
1. **Take profit**: Spread returns to KF mean estimate → exit
2. **Time stop**: Days in trade > 2× half-life → exit regardless of P&L

### Override
- Large sustained KF innovations (prediction errors spiking) → force-exit the pair mid-month, structural break detected

### Monthly Rebalancing
- Re-run Layer 1 clustering → update tradeable pair universe
- For each open position:
  - Pair **stays in tradeable cluster** → resize to new inverse vol parity weight
  - Pair **exits tradeable cluster** → force-close position

---

## Position Sizing

- **Across pairs**: Inverse volatility parity weighting — weight = 1/spread_vol, normalized to sum to 1 across top 10 pairs. Lower spread vol = larger position.
- **Within a pair**: KF β_t determines the long/short split

---

## Performance Metrics

Report against SPY benchmark:
- Cumulative returns
- Omega ratio
- Maximum drawdown
- Sharpe ratio (secondary)

---

## Document Structure (.qmd)

1. **Executive Summary** — mermaid mental model diagram
2. **Universe & Data** — assets, API data loading, inception date handling
3. **Layer 1 — Pair Selection** — clustering methodology, feature computation, k selection
4. **Layer 2 — Signal Generation** — Kalman Filter setup and outputs
5. **Entry & Exit Rules** — decision rules, time stop
6. **Monthly Rebalancing** — cluster-conditional rebalancing logic
7. **Performance Results** — vs SPY benchmark, all metrics
8. **Conclusion**

---

## R Packages & Tooling

- **Data**: `tidyquant::tq_get()` for API data loading (Yahoo Finance)
- **Time series**: `xts`, `timetk` for time series manipulation
- **Technical indicators**: `TTR` for indicator computation
- **Performance**: `RTL::tradeStats()`, `PerformanceAnalytics` for metrics
- **Clustering**: `stats` (k-means), `cluster` (hierarchical, silhouette)
- **Kalman Filter**: `dlm` package — use `dlmFilter()` for sequential daily updating. Fall back to `FKF` if performance is an issue.
- **Optimization**: `foreach` + `doParallel` for multi-core parameter grid search
- **Workflow**: `tidymodels` — recipe/prep/bake pipeline
- **Visualization**: `ggplot2`, `plotly` for interactive charts, `lattice::wireframe` for optimization surface
- **Document**: Quarto (`.qmd`), self-contained HTML output

### Course-Specific Coding Conventions (from FIN 452 notes)

**Data loading pattern:**
```r
dat <- tidyquant::tq_get("SPY", from = "2000-01-01") %>%
  dplyr::rename_all(tools::toTitleCase) %>%
  timetk::tk_xts(date_var = Date) %>%
  quantmod::adjustOHLC(., use.Adjusted = TRUE) %>%
  timetk::tk_tbl(rename_index = "Date") %>%
  dplyr::select(-Adjusted) %>%
  dplyr::mutate(across(where(is.numeric), round, 2)) %>%
  dplyr::rename(date = Date)
```

**Return types:**
- `retClCl = Close / lag(Close) - 1` — close-to-close (hold existing position)
- `retOpCl = (Close - Open) / Close` — open-to-close (new position day)
- `retClOp = Open / lag(Close) - 1` — close-to-open (overnight)

**Signal convention:**
```r
signal = dplyr::case_when(condition_long ~ 1, condition_short ~ -1, TRUE ~ 0)
```

**Trade & position tracking:**
```r
trade = tidyr::replace_na(dplyr::lag(signal) - dplyr::lag(signal, n = 2L), 0)
pos   = cumsum(trade)
```

**P&L computation (three cases):**
```r
ret_new    = ifelse(pos == trade, pos * retOpCl, 0)           # new position
ret_exist  = ifelse(pos != 0 & trade == 0, pos * retClCl, 0)  # unchanged position
ret_others = case_when(
  (pos - trade) != 0 & trade != 0 ~
    (1 + retClOp * (pos - trade)) * (1 + retOpCl * pos) - 1,
  TRUE ~ 0
)
ret    = ret_new + ret_exist + ret_others
cumeq  = cumprod(1 + ret)
```

**Strategy as a function (required for optimization):**
```r
strategy <- function(data = train, par1 = 50, par2 = 200) {
  data %>% dplyr::mutate(...)
}
```

**Parameter optimization (multi-core):**
```r
out <- expand.grid(par1 = seq(5, 50, 5), par2 = seq(55, 200, 5))

library(foreach); library(doParallel)
cl <- makeCluster(detectCores() - 1)
registerDoParallel(cl)
res <- foreach(i = 1:nrow(out), .combine = "cbind",
               .packages = c("tidyverse", "RTL", "timetk", "tidyquant")) %dopar% {
  as.numeric(RTL::tradeStats(strategy(train, out[i,"par1"], out[i,"par2"]) %>%
    dplyr::select(date, ret)))
}
stopCluster(cl)
```

**Performance metrics:**
- `RTL::tradeStats(data %>% select(date, ret))` — comprehensive stats
- `PerformanceAnalytics::OmegaSharpeRatio()` — preferred over raw Omega (comparable to Sharpe)
- `PerformanceAnalytics::table.Drawdowns()` — drawdown table (works best with xts)
- `PerformanceAnalytics::SharpeRatio()` — Sharpe ratio

**Omega ratio (non-parametric Sharpe alternative):**
```r
MAR <- 0  # minimum acceptable return
OmegaSharpeRatio(as.ts(ret), MAR = MAR)
```

### Data Splitting
- Training: June 2000 – December 2024
- Testing: January 2025 – March 2026
- Time-series split — use manual date cutoff (NOT random split):
```r
cutoff <- "2025-01-01"
train <- dat %>% dplyr::filter(date < cutoff)
test  <- dat %>% dplyr::filter(date >= cutoff)
```
- For tidymodels: use `rsample::initial_time_split()` (preserves time order)

### tidymodels Workflow (from FIN 450 notes)
```r
# 1. Split
split      <- rsample::initial_time_split(data, prop = 0.80)
df_train   <- rsample::training(split)
df_test    <- rsample::testing(split)

# 2. Recipe (feature engineering)
recipe_pipeline <- recipes::recipe(target ~ ., data = df_train) %>%
  recipes::step_rm(date) %>%
  recipes::step_normalize(all_numeric()) %>%
  recipes::prep()

train_baked <- recipes::bake(recipe_pipeline, df_train)
test_baked  <- recipes::bake(recipe_pipeline, df_test)

# 3. Model
model <- parsnip::linear_reg(mode = "regression") %>%
  parsnip::set_engine("lm") %>%
  parsnip::fit(target ~ ., data = train_baked)

# 4. Evaluate
model %>%
  stats::predict(new_data = test_baked) %>%
  dplyr::bind_cols(truth = test_baked$target) %>%
  yardstick::metrics(truth = truth, estimate = .pred)
```

---

## How Claude Should Behave in This Project

- Be a critical thinking partner — push back on bad ideas, don't just agree
- Keep responses concise and direct
- Do not add unnecessary complexity, parameters, or rules to the strategy
- When writing R code: follow tidymodels workflow, use xts/TTR conventions, load data via API only
- When asked to implement something, read existing code first before modifying
- The mental model is fixed — code must match it exactly
- Flag any deviations from the professor's hard requirements immediately
- Do not propose Python solutions under any circumstances
