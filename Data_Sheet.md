# Datasheet for Datasets

**Dataset Name:** Financial Market OHLCV Dataset (2010–2025)
**Version:** 1.0
**Date:** 2026-06-28
**Author(s):** User

This datasheet documents the composition, collection process, preprocessing steps, intended uses, distribution plan, and maintenance of the dataset described above.

---

### 1. Motivation

| Question | Answer |
| :--- | :--- |
| **What task does this dataset help to solve?** | Supports quantitative backtesting of trading strategies across AAPL, META, MSFT, and TSLA by providing high-fidelity, aligned OHLCV data. |
| **Who created it, and why?** | Created to establish a clean, robust data foundation for algorithmic trading backtests, ensuring market-calendar alignment and data integrity. |
| **Was it funded or supported by an organisation?** | No; developed for individual research purposes. |

### 2. Composition

| Question | Answer |
| :--- | :--- |
| **What types of data points are included?** | Daily Open, High, Low, Close, and Volume (OHLCV) values, plus a validity flag (`is_valid_day`). |
| **How many instances does the dataset contain?** | Spans 2010–2025; count reflects all valid trading days per the NYSE market calendar. |
| **Is the dataset complete or a sample?** | Represents a sample defined by `yfinance` availability constrained by the NYSE market calendar. |
| **What format are the instances in?** | Parquet files per ticker. |
| **Are there labels or annotations?** | Includes an `is_valid_day` boolean mask for data integrity tracking. |
| **Is there missing or incomplete data?** | Missing data is handled via forward-filling for partial integrity or masking for fatal gaps. |

### 3. Collection Process

| Question | Answer |
| :--- | :--- |
| **How was the data acquired?** | Automated pull using `yfinance` API with auto-adjust enabled. |
| **What sampling strategy was used?** | Systematic acquisition aligned with NYSE business days via `pandas_market_calendars`. |
| **What was the time frame of data collection?** | 2010-01-01 to 2025-12-31. |

### 4. Preprocessing, Cleaning and Labelling

| Question | Answer |
| :--- | :--- |
| **What preprocessing steps were applied?** | tz-naive normalization, reindexing to NYSE calendar, removal of duplicates, and tiered quality flagging. |
| **Was the raw data preserved?** | The processed files retain the original state plus the validity mask; external raw storage depends on user environment. |

### 5. Uses

| Question | Answer |
| :--- | :--- |
| **What are the intended uses of this dataset?** | High-integrity quantitative backtesting and signal generation research. |
| **What are the inappropriate uses?** | High-frequency trading models (due to daily granularity) or production-grade execution without further validation. |

### 6. Distribution

| Question | Answer |
| :--- | :--- |
| **How has the dataset been distributed?** | Stored locally in `data/processed/` within the project directory. |

### 7. Maintenance

| Question | Answer |
| :--- | :--- |
| **Who maintains the dataset?** | Maintained via the defined pipeline in Notebook 1. |
| **How is version control managed?** | Versioned via project-specific directory structures and metadata JSON logs. |
