# Sentiment & Trader Behavior on Hyperliquid — Write-Up

## 1. Methodology

**Data.** `historical_data.csv` is a raw Hyperliquid fill log: 211,224 rows, 16 columns,
32 unique accounts, 246 traded coins, spanning **2023-05-01 to 2025-05-01**. No missing
values and no exact duplicate rows. One data-quality issue was caught and corrected: the
`Trade ID` column is stored in truncated scientific notation (e.g. `8.95E+14`) in the
source file, so it collapses many distinct trades onto the same value — deduping on it
naively would have destroyed ~98% of the real data. We verified zero true duplicates via
a full-row check and used a synthetic row index for counting instead.

`fear_greed_index.csv` is a clean daily series, 2,644 rows, 2018-02-01 to 2025-05-02, no
missing values or duplicates. We aligned it to the trade log's window, leaving **731**
overlapping calendar days across five classes (Extreme Fear → Extreme Greed), which we
also collapsed into a **Fear / Neutral / Greed** binary for the headline comparisons.

**Metrics (per account-day, 2,341 account-day rows after merge).**
- `daily_pnl` — sum of `Closed PnL` for that account/day
- `win_rate` — wins ÷ realized closes, computed **only** over rows with non-zero Closed
  PnL (~104k of 211k rows are opens with PnL = 0 and were correctly excluded)
- `n_trades`, `avg_trade_size_usd`, `total_volume_usd`
- `long_short_ratio` and long/short **share of trades** (from `Direction`, with `Side` as
  fallback)
- **Leverage**: not present in this export (no margin/multiplier field). We used
  **average trade size in USD** as the closest available proxy for sizing behavior and
  flag this substitution explicitly rather than fabricating a leverage number.

**Segmentation** (32 accounts, tercile splits — small n per bucket, treat as directional):
- *Activity*: Infrequent / Moderate / Frequent, by total trade count
- *Size*: Small / Mid / Large, by average trade size
- *Consistency*: Inconsistent / Average / Consistent-winners, by overall win rate

**Stats.** Fear vs. Greed daily PnL was compared with a Mann-Whitney U test (robust to
the fat-tailed PnL distribution rather than assuming normality).

---

## 2. Key Insights

### Insight 1 — Traders take *more* trades and *bigger* size on Fear days, and it shows up as fatter tail risk
| Sentiment | Trades/day (avg) | Avg trade size | 5th-pct daily PnL (tail loss) |
|---|---|---|---|
| Fear | **105.4** | **$8,530** | **–$3,485** |
| Neutral | 100.2 | $6,964 | –$885 |
| Greed | 76.9 | $5,955 | –$174 |

Trade frequency is ~37% higher and average size ~43% higher on Fear days than Greed days
(`04_activity_and_size_by_sentiment.png`). The worst 5% of account-days are roughly
**20x more negative** in Fear than in Greed (`03_tail_risk_by_sentiment.png`). More
trades + bigger size in a worse-known regime is exactly the combination that produces
outsized drawdowns.

### Insight 2 — Mean PnL looks fine on Fear days, but that's a few big winners; the *typical* day is worse
Mean daily PnL is actually highest in Fear ($5,185) vs. Greed ($4,144), but **median**
daily PnL is less than half in Fear ($123) vs. Greed ($265), and the % of profitable
account-days is lower in Fear (60.4%) than Greed (64.3%) (`01_mean_median_pnl_by_sentiment.png`,
`02_winrate_profitable_days_by_sentiment.png`). The Mann-Whitney test on Fear vs. Greed
daily PnL gives **p ≈ 0.062** — a mean/median gap this wide with a borderline
significance test both point the same way: **Fear-day PnL is being carried by a small
number of large wins, while the median trader has a worse day than in Greed.** Averages
alone would have hidden this.

### Insight 3 — Traders flip their directional bias with sentiment, and it's the "wrong" way round
Long share of trades is **63%** on Fear days but only **44%** on Greed days
(`05_long_short_share_by_sentiment.png`) — i.e., traders are net-long into fear and
net-short into greed. This is a mild contrarian/dip-buying pattern rather than
momentum-following, and combined with Insight 1 (bigger size, more trades in Fear) it
means the riskiest directional bet (long, in a fear regime) is also the one taken with
the most size and frequency.

### Insight 4 — Segments respond very differently to sentiment (n is small — directional, not proof)
- **Consistent-winners** (top win-rate tercile) are resilient across regimes but still
  do best in Fear ($9,578 mean daily PnL vs. $6,135 in Greed) and post their highest win
  rate in Greed (96.9%) — they seem to both size up profitably in Fear *and* pick their
  spots well in Greed.
- **Inconsistent traders** barely move with sentiment (≈$2,900 in both Fear and Greed)
  but are **negative** on Neutral days (–$290) — sentiment extremes may be easier to
  read than range-bound/neutral markets for this group.
- **Moderate-frequency traders** show the widest swing of any segment: $11,099 mean
  daily PnL in Fear vs. just $1,175 in Greed — the single biggest sentiment-sensitivity
  in the dataset (`06_activity_segment_pnl_by_sentiment.png`).
- **Large-size traders** earn most in Fear ($10,452) and least in Greed ($3,729) —
  the opposite pattern from mid-size traders, who do best in Greed.

---

## 3. Bonus work

**Predictive model.** A Random Forest predicting next-day account profitability
(profit/loss) from today's sentiment + behavior scored **62.6% accuracy**, *below* the
**71.3%** naive baseline of always predicting "profitable" (the majority class, since
~71% of account-days are profitable). **Honest takeaway: sentiment + same-day behavior
do not carry enough next-day signal to beat the base rate** with this sample size (32
accounts, ~1,240 usable rows after the next-day shift). The two most important features
were `n_trades` and `daily_pnl`, i.e. recent behavior matters more than the sentiment
label itself — worth revisiting with more accounts/history rather than trusting this
model in production.

**Clustering (KMeans, k=3)** on total trades, avg trade size, win rate, and PnL
volatility surfaced three rough archetypes:
- **Cluster 0** (n=15) — high-frequency, small-size, modest win rate (75%), most
  volatile daily PnL relative to its size — "grinders."
- **Cluster 1** (n=13) — moderate frequency, small size, very high win rate (96%),
  lower volatility — "precise/selective."
- **Cluster 2** (n=4) — high frequency, **large size** ($15k avg), by far the highest
  PnL volatility (std ≈ $87k/day) and total PnL (~$953k) — "whales," carrying most of
  the portfolio's tail risk.

---

## 4. Strategy Recommendations

1. **Cap size and trade count on Fear days for everyone except proven "consistent
   winners."** The data shows the whole account base collectively trades bigger and
   more often exactly when tail losses are worst (Insight 1) — this is the single
   clearest, most actionable pattern. A concrete rule: *on Fear/Extreme-Fear days, cap
   position size at the account's Neutral-day average and set a daily trade-count
   limit*, unless the account is in the "consistent winners" segment, which
   historically handles Fear-day size well.

2. **Treat sentiment-driven directional bias as a flag to check, not follow.** Since
   the base rate is net-long in Fear and net-short in Greed (Insight 3), and mid-size /
   moderate-frequency traders do *worse* in Greed despite this being the "calmer"
   regime, a rule-of-thumb is: *before adding to a position on a Fear day, explicitly
   confirm the trade thesis independent of the fear signal* — the data doesn't show
   fear-buying paying off better than greed-buying; if anything the median outcome is
   worse.

---

## 5. Caveats

- 32 accounts is a small sample for segment-level (tercile) conclusions — treat
  segment findings as directional hypotheses to re-test as more data arrives, not firm
  rules.
- No true leverage/margin field exists in the export; "size" conclusions use trade
  notional (USD) as a proxy, not leverage multiple.
- The Fear-vs-Greed PnL difference is borderline (p ≈ 0.06), not a clean statistically
  significant result — we present it as a real signal worth monitoring, not a settled
  finding.
- The next-day predictive model underperforms the naive baseline; it's included for
  completeness and to show the honest limits of the signal, not as a usable
  prediction tool.
