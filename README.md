# Optimal Execution & Market Impact Engine — Crypto L2 Order Book

An end-to-end execution research pipeline built on **raw Binance USD-M ETHUSDT data**. It covers:

- reconstructing the exact L2 limit order book from diff-depth streams;
- measuring **transient market impact** out-of-sample;
- solving for **cost-vs-risk optimal liquidation schedules**;
- backtesting them against **TWAP, VWAP, POV and immediate execution**, with child orders that walk the recorded book.

> **Headline result:** across **4,626 held-out liquidations**, the optimal schedule cut **shortfall volatility by 52% vs TWAP** (27.1 → 12.9 bps) at statistically equal mean cost. VWAP and POV sat *inside* the realized efficient frontier. Immediate execution cost **+5 bps**.

---

## Contents
- [Pipeline](#pipeline)
- [Data](#data)
- [Methodology](#methodology)
- [Results](#results)
- [Repository structure](#repository-structure)
- [How to run](#how-to-run)
- [Validation and robustness](#validation-and-robustness)
- [Limitations](#limitations)
- [Roadmap](#roadmap)

---

## Pipeline

```
 raw trades (CSV/day) ─┐
                       ├─► 01 Data cleaning ──► 02 Microstructure ──► 03 Impact model ──► 04 Optimizer ──► 05 Backtest
 raw L2 diff-depth  ───┘     exact book            OFI, vol curve,       propagator          convex QP,        event-driven
 (parquet/hour)              replay, QC            regimes, splits       kernel, OOS R²      frontier          sim + bootstrap
```

| Notebook | Purpose | Key outputs |
|---|---|---|
| `01_data_cleaning.ipynb` | Order-book reconstruction, trade cleaning, QC | `ladder_1s`, `panel_{100ms,1s,1min}`, `taker_orders`, `qc_summary.json` |
| `02_microstructure.ipynb` | Microstructure features, walk-forward split, order grid | `split.json`, `signature.csv`, `volume_curve_forecasts`, `regimes`, `executions.parquet` |
| `03_impact_model.ipynb` | Transient impact kernel, concavity, OOS validation | `impact_params.json`, `impact_kernel.csv`, `concavity.csv` |
| `04_optimizer.ipynb` | Analytical and transient-impact optimal schedules | `frontier.csv`, `schedules.parquet`, `sensitivity.csv` |
| `05_backtest.ipynb` | Event-driven simulation vs benchmarks | `backtest_results.parquet`, results CSVs |

Each notebook is driven by a single **CONFIG** cell. Change the symbol, dates or paths there and run all cells.

---

## Data

| Stream | Source | Format | Coverage |
|---|---|---|---|
| Trades | Binance USD-M Futures, ETHUSDT | 1 CSV per day | 7–14 Sept 2026 |
| L2 order book | Binance diff-depth stream + periodic snapshots | 24 hourly parquet files per day | 7–14 Sept 2026 |

Raw data is **not included** in this repository because of its size. Expected file names:

```
ETHUSDT-trades-YYYY-MM-DD.csv
ETHUSDT_orderbook_YYYYMMDD_HH.parquet      # HH = 00 … 23
```

**Walk-forward split:** 7–10 Sept is used for calibration and 11–14 Sept is held out for testing. Every test day uses only parameters fitted on data before it.

---

## Methodology

### 1. Exact order-book reconstruction
- **Snapshot plus diff replay** with sequence validation: `pu == previous u`, or `U ≤ lastUpdateId ≤ u` on re-sync.
- **Snapshot re-sync at any position** in the stream, so multiple or mid-hour snapshots are handled correctly.
- **Gap recovery:** when the recorder dropped snapshots, the book is rebuilt from empty, followed by a **10-minute warm-up** before its data counts as usable.
- **Book carry** across hours and days (`book_state_end.npz`). Stamps use the **matching-engine clock** (`transaction_time`).
- **Dense tick-indexed arrays** make replay memory-safe. A column-by-column Arrow loader handles hour files with 15M+ rows.
- Trades: the aggressor side comes from `is_buyer_maker`. **Parent taker orders** are rebuilt by grouping fills with the same millisecond, same side and contiguous IDs.
- Per-hour **QC report**: crossed books, stale gaps, valid-time share.

### 2. Microstructure layer
- **Order-flow imbalance (OFI)** following Cont–Kukanov–Stoikov, plus microprice and depth bands at 10/25 bps.
- Volatility **signature plot**, used to choose a sampling frequency free of microstructure noise.
- **Intraday volume curve** and hour-of-day forecasts, which feed VWAP and POV.
- Kyle λ, Amihud illiquidity and volatility/liquidity **regimes**.

### 3. Transient market-impact model
A propagator model on signed order flow in 10-second bars (~69k bars):

```
r_t = Σ_l G(l) · q_{t-l},      H(l) = H_∞ + A · e^{-l/τ}
```

The parametric kernel is fitted through an exact recursion, `r_t = H_∞ q_t + A(E_t − E_{t−1})`, `E_t = q_t + ρ E_{t−1}`, implemented with `scipy.signal.lfilter`.

- A model-free, log-bucketed kernel is fitted alongside it as a cross-check.
- **HC1 robust standard errors** and a χ²-profile confidence interval for τ.
- **Square-root-law concavity** δ: least squares on all raw windows, not on binned means.
- **Walk-forward out-of-sample R²**.

### 4. Execution optimizer
- **Almgren–Chriss** discrete closed form as the baseline, unit-tested against the reference parameter set.
- **Transient-impact optimizer**: a convex quadratic program combining the decaying-impact kernel, a convex **book-walk cost** (from the immediate cost curve) and a variance penalty λ.
- Solver: **accelerated projected gradient** with exact simplex projection and capped backtracking. Verified against SLSQP and a 30k-iteration reference run.
- Outputs the **efficient frontier** (expected cost vs risk) and parameter sensitivity.

### 5. Event-driven backtest
- Child orders **walk the recorded bid ladder** at each second.
- **Self-impact** is added via FFT convolution with the kernel calibrated for that test day. It is evaluated under four impact scenarios.
- Benchmarks: **TWAP, VWAP, POV, immediate**. Participation is capped at 30%.
- Inference uses **paired differences vs TWAP** with a **cluster bootstrap over (day × hour)**.
- **Arrival-usability filter:** windows whose arrival mid is stale or falls in a book gap are skipped (15 windows).

---

## Results

### Market impact (held-out)
| Quantity | Estimate |
|---|---|
| Instantaneous impact H(0) | **1.10 bps per $1M** signed flow |
| Permanent impact H_∞ | **0.63 bps per $1M** [0.52, 0.74] |
| Permanent share | **57%** |
| Decay τ / half-life | **36 s / 25 s** |
| Concavity δ (cost ∝ size^δ) | **0.61** (1-min) → **0.87** (30-min) |
| Out-of-sample R² | **33.5%** vs 32.3% for a lag-0-only model |
| Ex-ante state dependence | No significant effect |

### Optimizer
- A heuristic risk-aversion choice was **12,868× too small**, which shows why calibration matters.
- The calibrated Almgren–Chriss schedule lands within **0.06 bps** of the full transient-impact optimum at these horizons.

### Backtest: 4,626 held-out sell liquidations
| Strategy | Mean shortfall vs TWAP | 95% CI | Shortfall SD |
|---|---|---|---|
| **Optimal (λ = 0.01)** | **−0.04 bps** | [−2.95, +2.72] | **12.9 bps** (TWAP: 27.1) |
| VWAP | +0.26 bps | — | — |
| POV | +0.96 bps | — | — |
| Immediate | **+4.97 bps** | [+1.7, +8.2] | — |

- **52% lower shortfall volatility** than TWAP, with no statistically detectable difference in mean cost.
- VWAP and POV lie **inside** the realized efficient frontier.
- At equal cost, the optimal schedule cuts risk by a **median 63%** relative to POV.
- Cost decomposition: risk reduction costs about **0.7 bps** of impact plus book walk (7.02 vs 6.31 bps). Any drift advantage is sample luck and is not claimed as alpha.

---

## Repository structure

```
.
├── 01_data_cleaning.ipynb
├── 02_microstructure.ipynb
├── 03_impact_model.ipynb
├── 04_optimizer.ipynb
├── 05_backtest.ipynb
├── requirements.txt
└── README.md

# generated at run time (git-ignored)
clean/ETHUSDT/<date>/        # per-day cleaned panels, ladders, QC
analysis/ETHUSDT/            # model, optimizer and backtest outputs
```

---

## How to run

```bash
git clone https://github.com/<your-username>/<repo-name>.git
cd <repo-name>
pip install -r requirements.txt
jupyter lab
```

1. Put the raw trade CSVs and L2 parquet files in your data folder.
2. In `01_data_cleaning.ipynb`, set `DATA_DIR` and `DATE` in CONFIG. Run it once per day (7–14 Sept). Keep `CARRY_FROM_PREVIOUS_DAY=True` so the book state carries over.
3. In notebooks 02–05, set `ROOT` to the folder containing `clean/`, then run them in order. Each one reads the previous notebook's outputs.

**requirements.txt**
```
numpy
pandas
pyarrow
scipy
matplotlib
jupyterlab
```

---

## Validation and robustness
- **Look-ahead control:** everything is walk-forward. Kernels, volume curves and risk aversion are fitted only on data before each test day.
- **Synthetic recovery test:** a known kernel (H_∞, A, τ) was planted in synthetic data and recovered by notebook 03.
- **Solver checks:** the AC closed form is unit-tested; the projected-gradient solution matches SLSQP.
- **Data integrity:** book-sequence gaps, stale books and warm-up periods are flagged and excluded, never forward-filled into prices.
- **Honest inference:** the cluster bootstrap over (day × hour) accounts for intraday correlation. Confidence intervals are reported, not just point estimates.

---

## Limitations
- **Sell-side only.** Over 4 test days, price drift can favour one side.
- **Short sample:** 8 days in total, 4 out-of-sample.
- **Single asset** (ETHUSDT). Taker-only execution, with no fees and no limit orders or queue position.
- Each schedule is **fixed at arrival**, with no re-optimization during execution.

## Roadmap
- [ ] Buy-side liquidations and a longer sample (months, not days)
- [ ] Exchange fees, plus passive limit orders with queue-position modelling
- [ ] Adaptive schedules that re-optimize on live OFI and spread
- [ ] Multi-asset **portfolio execution** (BTC, ETH, SOL) with a cross-impact model
- [ ] C++ order-book replay engine with latency and throughput benchmarks
- [ ] Paper trading on the Binance testnet

---

## Author
**Nitish Singh** — Dual Degree (Financial Engineering), IIT Kharagpur
