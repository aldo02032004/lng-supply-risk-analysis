# Specification

Technical description of `lng_supply_risk_analysis.ipynb`: scope, inputs, method, status rules, outputs and variable names.

## 1. Scope

| Item | Description |
|---|---|
| Subject | Annual LNG cost of one ammonia-urea plant (hypothetical) |
| Risk owner view | Buyer of LNG: higher prices and a weaker Rupiah are bad |
| Framework | ISO 31000 risk management cycle, ISO 22301 for business continuity |
| Horizon | Next 12 months |
| Currency | Cost in Indonesian Rupiah (IDR trillion), market prices in USD |
| Run time | About 10–20 seconds, mostly data download |

## 2. Inputs

### 2.1 Market data (downloaded each run)

| Name in `px` | Description | Source | Unit |
|---|---|---|---|
| `brent` | Brent crude oil spot price, Europe | FRED `DCOILBRENTEU` (CSV endpoint, no API key) | USD/barrel |
| `usdidr` | US dollar to Indonesian rupiah, daily close | Yahoo Finance `IDR=X` via `yfinance` | IDR per USD |

Rules:
- Period: `P["start"]` to the latest available date. FRED usually lags by about one week.
- Each successful download is cached to `data/<name>.csv`. If a download fails, the cache is used.
- FX ticks deviating more than 4% from their centered 11-day median are replaced by the previous value (bad Yahoo ticks).
- Missing dates are forward-filled. Leading rows without both series are dropped.

### 2.2 Parameters (`P`)

| Key | Unit | Default | Role |
|---|---|---|---|
| `urea_cap` | t urea / year | 1,000,000 | Volume |
| `util` | fraction | 0.90 | Volume |
| `gas_per_ton` | MMBtu / t urea | 25 | Volume |
| `slope` | fraction of Brent | 0.12 | Price formula |
| `const` | USD / MMBtu | 1.5 | Price formula |
| `rkap` | IDR trillion | 6.0 | Annual budget |
| `appetite` | score | 6 | Highest green score |
| `tolerance` | score | 12 | Highest yellow score |
| `cost_tol` | fraction | 0.10 | Cost tolerance above budget |
| `n_sims` | scenarios | 10,000 | Monte Carlo size |
| `horizon` | months | 12 | Simulation length |
| `conf` | probability | 0.95 | VaR/CVaR confidence |
| `hedge` | fraction | 0.50 | Hedged share of volume |
| `seed` | integer | 42 | Reproducibility |
| `start` | date | 2015-01-01 | First data date |
| `llm_model` | model id | gemini-3.6-flash | Recommendation model |

Derived volume: `mmbtu = urea_cap × util × gas_per_ton` = 22.5 million MMBtu per year (about 433 thousand tons of LNG at 52 MMBtu/t).

### 2.3 Risk register (`reg`)

| Column | Type | Description |
|---|---|---|
| `id` | text | Risk ID, e.g. R01 |
| `risiko` | text | Risk description |
| `kategori` | text | Category (market, financial, operational, supply, regulatory, credit) |
| `L`, `I` | 1–5 | Current likelihood and impact |
| `L_lalu`, `I_lalu` | 1–5 | Ratings from the previous review |
| `owner` | text | Department that owns the risk |
| `tenggat` | date | Mitigation deadline |

### 2.4 Business continuity (`bcms`)

| Column | Unit | Description |
|---|---|---|
| `id`, `skenario` | text | Linked risk and disruption scenario |
| `estimasi downtime (jam)` | hours | Estimated outage length |
| `RTO (jam)` | hours | Recovery time objective |
| `MTPD (jam)` | hours | Maximum tolerable period of disruption |

### 2.5 Optional

`GOOGLE_API_KEY` from Colab Secrets or the environment, for Gemini recommendations. Without it, the last saved answer in `data/rekomendasi.json` is used.

## 3. Method

### 3.1 Returns
- Daily log return: `r_t = ln(P_t / P_{t-1})`.
- Annual volatility: `std(r_daily) × √252`.

### 3.2 Monte Carlo simulation (`simulate`)
1. Monthly average prices → monthly log returns `X` (one row per month, one column per series).
2. Demean: `X = X − mean(X)`, so the simulation has no directional view.
3. Starting point `spot` = mean of the last 90 daily rows.
4. Joint bootstrap: draw `n_sims × horizon` whole rows of `X` with replacement (`rng.choice`). Drawing whole rows keeps the Brent/FX co-movement and the fat tails of history.
5. Price paths: `path = spot × exp(cumsum(draws))`.
6. LNG price per month: `price = slope × Brent + const` (USD/MMBtu).
7. Hedge: `price = h × price_0 + (1 − h) × price`, where `price_0` uses the starting Brent.
8. Annual cost: `Σ_months (mmbtu / 12) × price × FX / 10^12` → IDR trillion.

The same seed is used for every call, so "without hedge" and "with hedge" see identical market scenarios.

### 3.3 Cost metrics (`metrics`)

| Metric | Formula | Reading |
|---|---|---|
| Mean, Median, Std | of the 10,000 costs | Expected cost and spread |
| P5 | 5th percentile | A good year |
| VaR 95 | 95th percentile | A bad but plausible year |
| CVaR 95 | mean of costs ≥ VaR 95 | Average of the worst 5% |
| CaR vs RKAP | VaR 95 − budget | Budget overrun in a bad year |
| CaR vs mean | VaR 95 − mean | Reserve needed above expected cost |
| P(> RKAP) | share of costs > budget | Probability of exceeding the budget |

### 3.4 Monitoring
- **KRI:** 60-day rolling volatility, annualized (`× √252`), compared with its own historical 75th and 90th percentiles.
- **BCMS:** estimated downtime compared with RTO and MTPD.
- **Previous period:** `simulate` and `metrics` are re-run on data up to the end of the previous month (`to_period("M")`; use `"Q"` for quarters).

### 3.5 Gemini recommendations
All yellow and red risks are sent in one prompt. The model must return JSON `{"risk id": "recommendation"}`. Green risks get "-". Each answer is saved to `data/rekomendasi.json`; when the call fails, that saved copy is used.

## 4. Status rules

One helper, `status(x, green_limit, yellow_limit)`, is used everywhere:

| Check | HIJAU (green) | KUNING (yellow) | MERAH (red) |
|---|---|---|---|
| Register score `L × I` | ≤ `appetite` | ≤ `tolerance` | > `tolerance` |
| LNG cost (VaR 95) | ≤ `rkap` | ≤ `rkap × (1 + cost_tol)` | above |
| Volatility KRI | < P75 | P75 – P90 | ≥ P90 |
| BCMS downtime | ≤ RTO | ≤ MTPD | > MTPD |

Standard actions: green "accept, monitor routinely", yellow "monitor closely, prepare mitigation", red "mitigate now".

## 5. Outputs (section 6)

| Section | Output |
|---|---|
| 6.1 | Slide `01_ringkasan`: four headline numbers (VaR 95, probability above budget, hedge effect, red risks) and four status cards (cost, two volatility KRIs, worst BCMS scenario) |
| 6.2 | Cost table without vs with hedge (mean, VaR, CVaR, CaR, probability above budget) with a plain-language meaning per row; slide `02_biaya_hedging`: both cost distributions with budget, tolerance and VaR lines |
| 6.3 | Full register sorted by score (score change, status, recommendation, owner, deadline); slide `03_register`: 5×5 risk matrix and score change per risk |
| 6.4 | Table of changes since last month; slide `04_pemantauan`: 60-day volatility with P75/P90 thresholds and BCMS downtime vs RTO/MTPD |
| 6.5 | Slide `05_data_pasar`: price history and daily return distributions with volatility, largest moves and correlation |

Every slide is a 13.33 × 7.5 inch (16:9) figure whose title states the main message.

## 6. Checks

- `assert CVaR 95 ≥ VaR 95 ≥ Median`: the metric order must hold.
- `assert std(cost with 100% hedge) < std(cost without hedge)`: hedging must narrow the distribution.
- The notebook stops if either check fails, which signals a bug in the cost formula.

## 7. Assumptions and limitations

- Hypothetical plant, contract, budget, register and BCMS values.
- Brent is a proxy for LNG prices (JKM is licensed).
- Bootstrapped months are independent: no seasonality, autocorrelation or mean reversion.
- Hedge has no premium, basis risk, margin or credit risk; FX is not hedged.
- Volume is constant; demand and outage-driven volume changes are not simulated.

## 8. Dependencies

Python 3.10+, `numpy`, `pandas` ≥ 2.2, `matplotlib`, `seaborn`, `yfinance`; optional `google-genai`. See `requirements.txt`.

## Variable glossary

Code, comments and printed labels are in Indonesian. Main names:

| Name | Meaning |
|---|---|
| `px` | Daily prices (`brent`, `usdidr`) |
| `ret` | Daily log returns |
| `reg` | Risk register |
| `skor`, `skor_lalu` | Current and previous risk score |
| `status` | Status (HIJAU/KUNING/MERAH = green/yellow/red) |
| `rekomendasi`, `sumber_rek` | Mitigation recommendation and its source |
| `owner`, `tenggat` | Risk owner and deadline |
| `cost`, `cost_h` | 10,000 simulated annual costs without and with hedge |
| `m`, `m_hedge`, `m_lalu` | Cost metrics without hedge, with hedge, and for last month |
| `batas`, `cost_status` | Cost tolerance limit and cost status |
| `kri`, `vol60`, `q` | KRI table, 60-day volatility, P75/P90 thresholds |
| `bcms` | Business continuity table |
| `lalu` | Price data up to the end of last month |
| `rkap` | Annual budget (Rencana Kerja dan Anggaran Perusahaan) |
| `rp()`, `pct()`, `idn()`, `tgl()`, `bln()` | Format IDR trillion, percent, Indonesian number notation, and Indonesian dates |
| `slide()`, `tampil()` | 16:9 chart canvas with message title; table with title and reading note |
| `pendek()` | Short risk name (text before the first bracket or comma) |
| `nama`, `warna` | Display names and colors per series |
| Biaya, Rata-rata, Anggaran | Cost, average, budget |
| Terburuk, Terbaik | Worst, best |
| Penurunan, Perubahan | Reduction, change |
| Bulan lalu, Sekarang | Last month, now |
