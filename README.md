# Hyperliquid Sentiment Analysis

Analysis of how Fear/Greed sentiment relates to trader behavior and performance on
Hyperliquid, using the Bitcoin Fear & Greed Index and a Hyperliquid historical trade log.

## Repo structure

```
.
├── README.md                    <- this file
├── REPORT.md                    <- full write-up: methodology, insights, strategy rules
├── data/
│   ├── historical_data.csv      <- Hyperliquid trade log (32 accounts, 211,224 fills)
│   └── fear_greed_index.csv     <- Bitcoin Fear & Greed Index (daily, 2018–2025)
├── notebook/
│   ├── hyperliquid_sentiment_analysis.ipynb   <- main notebook (run this)
│   ├── 01_prep_and_metrics.py   <- Part A: load, clean, align, build metrics
│   ├── 02_analysis.py           <- Part B: sentiment vs performance/behavior, segments
│   ├── 03_charts.py             <- chart generation
│   └── build_notebook.py        <- assembles the .py scripts into the .ipynb
└── outputs/
    ├── tables/                  <- all intermediate & summary CSVs (+ text logs of stdout)
    └── charts/                  <- all PNG charts (01–10)
```



## How to run

**Option 1 — open the notebook:**
```bash
jupyter notebook notebook/hyperliquid_sentiment_analysis.ipynb
```
Run all cells top to bottom. It reads from `data/`, writes tables to `outputs/tables/`
and charts to `outputs/charts/`.

**Option 2 — run the scripts directly (same logic, no notebook needed):**
```bash
cd notebook
python3 01_prep_and_metrics.py
python3 02_analysis.py
python3 03_charts.py

```

## Data notes (important for reproducibility)

- **`historical_data.csv`**: 211,224 rows, 16 columns, 32 unique accounts, 246 coins,
  trade window **2023-05-01 → 2025-05-01**. No missing values, no exact duplicate rows.
- **The `Trade ID` column in the source file is stored in truncated scientific notation**
  (e.g. `8.95E+14`), so its precision is already lost at export — it is **not** usable as
  a unique row key. Deduplicating on it would incorrectly discard ~98% of genuine,
  distinct trades. We verified there are zero full-row duplicates and used a synthetic
  `row_id` (row index) for counting instead.
- **`fear_greed_index.csv`**: 2,644 rows, daily, 2018-02-01 → 2025-05-02, no missing
  values, no duplicates. We aligned it to the trade log's date range (731 overlapping days).
- **Leverage**: the trade log has no explicit leverage/margin-multiplier column. We used
  **trade size (USD)** as the closest available proxy for position-sizing behavior and
  say so explicitly in the report rather than inventing a leverage figure.
- **Win rate / realized PnL**: most rows have `Closed PnL == 0` because they are
  position-opening fills, not closes. Win rate is computed only over the ~104k rows with
  non-zero Closed PnL (i.e., actual realized closes).

See `REPORT.md` for the full methodology, insights, and strategy recommendations.
