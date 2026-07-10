# Polymarket Wallet ROI Project

This repository reflects the results from an observational study of a single Polymarket prediction market, *"Will there be a US
Government shutdown before 2025?"*, asking whether traders who entered the market early earned
a different mean return on investment (ROI) than traders who entered late. The market ran from
September 3 to December 31, 2024, traded roughly $54M in volume, and attracted 5,080 unique wallets.
Each wallet was classified as early or late based on whether its first trade occurred on or
before December 17, 2024 at 7:00 PM EST, a hand-picked timestamp just before public sentiment
rapidly flipped the market's expected outcome. From each group, two independent simple random
samples of 250 wallets were drawn. The code in this repository identifies and samples
those wallets via an SQL query on Dune, reconstructs each sampled wallet's trade history (on this market) through
Polymarket's public API, and computes per-wallet ROI to be analyzed and commented on
in the accompanying report.

## Findings at a glance

A two-sample t-test and a 95% two-sample t-interval for the difference in mean ROI between
early and late traders were computed in the report, but no formal conclusion was drawn from
either. The early group failed the 10% condition: 250 wallets were sampled from an early-trader
population of only 502. Sampling that large a share of a finite population
means the observations may be correlated with one another, which distorts the standard error
and makes the t-statistic, p-value, and confidence interval unreliable. This was addressed in the limitations section of the report, which is attached as a PDF in this repository.

## How it works

1. **Classify and sample wallets: `Dune/dune_market_query.sql`.** A single Dune query that goes through a chain of CTEs over Polymarket's on-chain trade data table: it
   timestamps every wallet's first trade in the market (whether as maker or taker), categorizes each
   wallet as either early (`A`) or late (`B`) relative to the configurable splitting timestamp, then uses
   `ROW_NUMBER() OVER (PARTITION BY group_label ORDER BY RANDOM())` to draw a configurable amount of randomly selected wallets
   from each group. If either group falls below a configurable minimum, the
   query falls back to a second output: daily counts of new traders from the market creation to resolution instead of wallet
   addresses (providing the user with more information that can be useful to find a better splitting timestamp).

2. **Download the results: `Dune/fetch_query_results_and_save_as_csv.py`.** A short script
   that requests the query's results from Dune's API (query ID & API key need to be set) and saves them
   locally as a CSV. The sampled wallets used in the study are saved as either
   `Dune/output_A.csv` or `Dune/output_B.csv` depending on whether the conditions for output A were met.

3. **Reconstruct trades and compute ROI: `Polymarket/roi_calculator.py`.** For each sampled
   wallet, the script pages through Polymarket's public data API to rebuild that wallet's full
   trade history in this market, retrying with exponential backoff on rate limits and network
   errors. Raw trades are cached to disk (one JSON file per wallet) and results are
   checkpointed after every wallet, so an interrupted run resumes instead of re-fetching
   everything. All math uses `Decimal` to avoid float rounding errors, and ROI is calculated via the following formula:

   ```
   ROI = (total sale proceeds + value of leftover shares at resolution − total spent) / total spent
   ```

   Both numerator terms are vital. A wallet that bought shares of the winning outcome, but held them to resolution, shows zero sale proceeds yet still earned a return (because its leftover shares became worth
   $1 each at resolution). Per-wallet results are written to `Polymarket/roi_output.csv`.

4. **Statistical analysis.** The two-sample t-test and t-interval were performed in the report, but
   not by any script in this repo [stapplet.com](https://stapplet.com/) was used for this stage.

## Repository structure

```
Dune/
  dune_market_query.sql                             # Finds, classifies, and randomly samples wallets (with a fallback: daily count representing the amount of new wallets entering the market)
  fetch_query_results_and_save_as_csv.py            # Executes the Dune query and saves results as a CSV
  output_A.csv                                      # The 500 sampled wallets: address, first-trade time, group, pre-sample group size
  output_B.csv                                      # Example of the fallback output: daily new-trader counts
Polymarket/
  roi_calculator.py                                 # Rebuilds each wallet's trade history via Polymarket's Data API, computes ROI, saves results as a CSV
  roi_output.csv                                    # Final output for analysis
README.md
Statistical Report - Polymarket Wallet ROI.pdf      # Final report (containing analysis & commentary)
```

## How to run

1. Create a Dune account, save `Dune/dune_market_query.sql` as a query, edit only its `params`
   CTE (market condition ID, timestamps, minimum group size, sample size), and run it.
2. Set the query ID and your Dune API key in `Dune/fetch_query_results_and_save_as_csv.py`,
   then run it to download the results CSV.
3. Update the paths in the CONFIG section of `Polymarket/roi_calculator.py` (input CSV, cache
   and output locations), plus the market ID and resolution outcome if you changed markets.
4. Run `Polymarket/roi_calculator.py` to fetch, cache, and compute the ROI for every wallet.

The only third-party dependency is `requests` (`pip install requests`); everything else is
Python standard library. Warning: I built this as a one-off analysis... paths are hardcoded
and the scripts target one specific market.

## Data sources

- **Dune**: indexes Polymarket's on-chain activity and makes it queryable.
- **Polymarket public data API**: provides the per-wallet trade data used to reconstruct each
  sampled wallet's trade history within the market being analyzed.

## License

Released under the [MIT License](LICENSE).

Copyright &copy; 2026 Nick Sleptsov