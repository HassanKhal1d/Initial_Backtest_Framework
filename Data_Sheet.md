# Datasheet for Datasets
**Dataset Name:** Financial Market OHLCV Dataset (2010–2025)
**Version:** 1.1
**Date:** 2026-07-05
**Author(s):** Hassan
**Change log (v1.0 → v1.1):** Corrected preprocessing description to match the Notebook 1 implementation exactly (duplicate dates are flagged, not dropped; the NYSE calendar index is built but not used to reindex the per-ticker frames); corrected the storage path; added the derived signals dataset produced by Notebook 2 as a related artifact.

This datasheet documents the composition, collection process, preprocessing steps, intended uses, distribution plan, and maintenance of the dataset described above.

---

### 1. Motivation

| Question | Answer |
| :--- | :--- |
| **What task does this dataset help to solve?** | Supports quantitative backtesting of trading strategies across AAPL, META, MSFT, and TSLA by providing high-fidelity, aligned OHLCV data. |
| **Who created it, and why?** | Created to establish a clean, robust data foundation for algorithmic trading backtests, ensuring market-calendar awareness and data integrity. |
| **Was it funded or supported by an organisation?** | No; developed for individual research purposes. |

### 2. Composition

| Question | Answer |
| :--- | :--- |
| **What types of data points are included?** | Daily Open, High, Low, Close, and Volume (OHLCV) values, plus a validity flag (`is_valid_day`). |
| **How many instances does the dataset contain?** | Spans 2010-01-01 to 2025-12-31; row count per ticker equals the number of NYSE trading dates over that period (with META additionally carrying pre-IPO rows set to NaN, see §4). |
| **Is the dataset complete or a sample?** | Represents a sample defined by `yfinance` availability, downloaded per calendar date range rather than reindexed against an independent trading calendar (see §4). |
| **What format are the instances in?** | Parquet files, one per ticker (`ohlcv_{TICKER}.parquet`), plus a `metadata.json` audit log. |
| **Are there labels or annotations?** | Includes an `is_valid_day` boolean mask: `True` where at least 3 of the 5 OHLCV fields are non-null for that row, `False` otherwise. |
| **Is there missing or incomplete data?** | Yes. Rows with ≤2 non-null OHLCV fields are flagged `is_valid_day = False` ("fatal" gaps, preserved but masked from trading logic). Rows with ≥3 non-null fields are forward-filled in place across all five OHLCV columns. |

### 3. Collection Process

| Question | Answer |
| :--- | :--- |
| **How was the data acquired?** | Automated pull per ticker using `yfinance.download()` with `auto_adjust=True` (splits and dividends applied). |
| **What sampling strategy was used?** | A candidate NYSE trading-date index is generated via `pandas_market_calendars` (`nyse.schedule()` → `mcal.date_range()`) for reference, but the per-ticker OHLCV frames are populated directly from the `yfinance` download rather than being reindexed onto this calendar index (see §4 for the implication). |
| **What was the time frame of data collection?** | 2010-01-01 to 2025-12-31. |

### 4. Preprocessing, Cleaning and Labelling

| Question | Answer |
| :--- | :--- |
| **What preprocessing steps were applied?** | (1) Flatten `yfinance` multi-index columns where present; (2) subset to `Open, High, Low, Close, Volume`; (3) normalise the index and strip timezone to produce tz-naive daily dates; (4) for META, set all rows before 2012-05-18 (pre-IPO) to NaN; (5) count null values and duplicate index entries per ticker for the audit log; (6) apply tiered validity masking (`is_valid_day`) and forward-fill OHLCV columns on rows with partial (≥3 field) data. |
| **Are duplicate dates removed?** | No — duplicate index entries are **detected and counted** (`duplicate_indexes` in `metadata.json`) but not dropped from the frame. Any downstream duplicate-handling is left to the consuming notebook. |
| **Is the NYSE calendar used to reindex the data?** | Partially. A master NYSE trading-date index is constructed in Notebook 1 but is not applied via `.reindex()` to the per-ticker frames in the current implementation — the shipped OHLCV frames follow whatever dates `yfinance` returned. This is a known gap flagged for hardening (see §7). |
| **Was the raw data preserved?** | The processed Parquet files retain the original OHLCV values plus the added `is_valid_day` mask and (for META) the pre-IPO NaN rows; no separate raw/unprocessed copy is persisted outside the user's local environment. |

### 5. Uses

| Question | Answer |
| :--- | :--- |
| **What are the intended uses of this dataset?** | High-integrity quantitative backtesting and signal generation research (e.g. the MA crossover strategy in Notebooks 2–9). |
| **What are the inappropriate uses?** | High-frequency trading models (data is daily granularity only); production-grade execution without further validation, especially given the reindexing gap noted in §4. |

### 6. Distribution

| Question | Answer |
| :--- | :--- |
| **How has the dataset been distributed?** | Stored locally at `Data/Processed/` within the project directory, as `ohlcv_{TICKER}.parquet` (one file per ticker) and `metadata.json`. |
| **Are there derived/related artifacts?** | Yes. Notebook 2 (Signal Construction) reads each `ohlcv_{TICKER}.parquet`, computes SMA/EMA short, long, and 200-day trend-filter series plus lagged position and return columns, and writes these out as a companion dataset, `signals_{TICKER}.parquet`, in the same directory. This derived dataset is not in scope of this datasheet but inherits all data-quality caveats from the underlying OHLCV files. |

### 7. Maintenance

| Question | Answer |
| :--- | :--- |
| **Who maintains the dataset?** | Maintained via the pipeline defined in Notebook 1 (`01_Data_Loading_&_Preprocessing.ipynb`). |
| **How is version control managed?** | Versioned via project-specific directory structure and the `metadata.json` audit log (date range, exclusions, adjustment method, and per-ticker quality-check counts). |
| **Known gaps / hardening recommendations** | (1) Apply the constructed NYSE calendar index via an explicit `.reindex()` step so every ticker frame is guaranteed to span the full common trading calendar; (2) drop or explicitly resolve counted duplicate index entries rather than leaving them in place; (3) persist a copy of the pre-preprocessing raw pull alongside the processed files for full auditability. |

---
*MA Crossover Backtesting Project | Hassan | Updated July 2026*
