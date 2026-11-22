# Data Cleaning & Quality Audit Pipeline

## Overview

`cleaning.ipynb` processes all five raw CSV files from `../Data Before cleaning/`, applies deterministic normalization, sibling `Order_ID` recovery, relational validation, transparent numeric imputation, status/date reconciliation, and KPI enrichment, and writes clean datasets alongside full audit outputs to `Data After cleaning/`.

---

## Data Problems Found & Solutions Implemented

| Issue Found in Raw Data | Previous Gap | Upgraded Pipeline Solution |
|---|---|---|
| **Broken Relative Paths** | Read/wrote from current working directory, failing inside `Data After cleaning/`. | `resolve_project_root()` locates `Data Before cleaning/` and `Data After cleaning/` cleanly from root or subfolder. |
| **Inconsistent Casing & Whitespace** | `Region` (`mansoura`, ` Cairo`), `Category` (`home`, `beauty`), `Order_Status` (` delivered `, `SHIPPED`), and `Payment_Method` (`wallet`, `ONLINE`, `Card `) had mixed formats. | Standardized with `.str.strip().str.title()` and validated against explicit allowed enum sets. |
| **Dirty Units & Percentages** | `Quantity` values like `1 pcs`, `6 pcs`; `Discount` values like `15%`, `5%`, `10%`, `0%`. | Regex-strips `pcs` (case-insensitive) to integer `Quantity`; detects `%` strings and converts them to decimal fractions. |
| **Exact & Key-Level Duplicates** | Duplicate lines in `orders.csv` (`ORD10009`) and `new_daily_batch.csv` (`NB000075`, `NB000090`, `NB000104`, `NB000125`). | Normalizes strings/numbers before deduplicating exact rows and `(Order_ID, Product_ID)` keys; logs dropped duplicates to `rejected_records.csv`. |
| **Unvalidated `Customer_ID` Foreign Keys** | Original script checked `Product_ID` (`PX01`, `PX02`, `P555`, `P666`) but ignored invalid `Customer_ID`s (`C5555`, `C6666`) in `new_daily_batch.csv`. | Validates both `Product_ID` and `Customer_ID` against `products_master` (`P001`–`P030`) and `customers_master` (`C0001`–`C0150`). |
| **Recoverable Multi-Line Order Fields** | Missing `Customer_ID` (`ORD10003`, `ORD10059`, `ORD10160`, `NB000063`, `NB000156`), fake `Customer_ID` on partial lines (`NB000206`, `NB000110`, `NB000211`), and `ORD10052` `not-a-date` were either filled with `"Unknown"` or dropped. | Unambiguous sibling `Order_ID` recovery restores `Customer_ID`, `Order_Date`, and `Ship_Date` when valid sibling rows agree on a single value, recorded in `Recovered_Fields`. Unrecoverable rows (`NB000052` `C6666`, `NB000088` blank) are quarantined. |
| **Status vs. `Ship_Date` Contradictions** | `60` `Pending`/`Cancelled` rows in `new_daily_batch.csv` carried bogus `Ship_Date`s; some `Delivered`/`Shipped` rows lacked `Ship_Date`; `ORD10085` mixed `Cancelled` and `Shipped`. | Rejects chronological errors (`Ship_Date < Order_Date`) first, clears `Ship_Date` to null on `Pending`/`Cancelled` orders (`Ship_Date_Cleared = True`), and flags remaining `Shipped`/`Delivered` lines without dates (`Ship_Date_Anomaly`) plus mixed-status orders (`Order_Status_Conflict`). |
| **Missing `Unit_Price` & `Shipping_Cost`** | `ORD10038` (`unknown`) and `ORD10180` (blank) lacked `Unit_Price`; `new_daily_batch.csv` had missing `Shipping_Cost`. | Imputes `Unit_Price` via product median valid price (fallback to `Unit_Cost` markup) and `Shipping_Cost` via unambiguous sibling value or dataset median, tracked with `Is_Imputed` and `Imputed_Fields`. |
| **Silent Row Drops & Unchecked Summary** | Dropped bad rows with zero audit trail and skipped validation on `previous_days_summary.csv`. | Quarantines all rejected rows to `rejected_records.csv` with original row numbers and reasons; validates `previous_days_summary.csv` dates, counts, and `[0, 1]` rate bounds. |

---

## Output Artifacts & Enriched Columns

### Cleaned Tables (`Data After cleaning/`)
- `customers_master_cleaned.csv` (`150` rows) — Normalized & validated customer directory.
- `products_master_cleaned.csv` (`30` rows) — Added `Stock_Alert` (`Healthy`, `Reorder Needed`, `Out of Stock`) and `Units_Below_Reorder`.
- `previous_days_summary_cleaned.csv` (`30` rows) — Chronologically sorted, type-checked September 2025 daily summary (`Total_Sales = 8,418,231.21`).
- `orders_cleaned.csv` (`1,011` rows) & `new_daily_batch_cleaned.csv` (`314` rows) — Enriched with:
  - **Audit & Quality Flags**: `Recovered_Fields`, `Ship_Date_Cleared`, `Ship_Date_Anomaly`, `Order_Status_Conflict`, `Imputed_Fields`, `Is_Imputed`
  - **Business KPIs**: `Gross_Revenue`, `Net_Revenue`, `Line_Cost`, `Line_Profit`, `Delivery_Days`

### Audit & Reporting Files
- `rejected_records.csv` (`28` rows total: `9` from `orders.csv`, `19` from `new_daily_batch.csv`) — Includes `Source_File`, `Original_Row_Number`, and `Rejection_Reason`.
- `cleaning_summary.csv` — Per-dataset before/after counts, recoveries, imputations, anomalies, net revenue, and line profit.

---

## Exact Run Results

| Dataset | Raw | Duplicates | Recovered | Imputed | Ship Date Cleared | Ship Date Anomaly | Rejected | Clean | Net Revenue | Line Profit |
|---|---:|---:|---:|---:|---:|---:|---:|---:|---:|---:|
| `customers_master` | 150 | 0 | 0 | 0 | 0 | 0 | 0 | **150** | — | — |
| `products_master` | 30 | 0 | 0 | 0 | 0 | 0 | 0 | **30** | — | — |
| `previous_days_summary` | 30 | 0 | 0 | 0 | 0 | 0 | 0 | **30** | 8,418,231.21 | — |
| `orders.csv` | 1,020 | 2 | 4 | 2 | 0 | 1 | 9 | **1,011** | 1,055,953.26 | 273,418.26 |
| `new_daily_batch.csv` | 333 | 4 | 11 | 7 | 60 | 5 | 19 | **314** | 317,174.08 | 106,624.08 |

### Rejection Breakdown (`rejected_records.csv`)
- **`orders.csv` (`9` rows)**: `Exact duplicate row` (`2`), `Invalid Product_ID` (`2`), `Quantity <= 0 or non-numeric` (`2`), `Discount outside [0, 1]` (`2`), `Ship_Date before Order_Date` (`1`).
- **`new_daily_batch.csv` (`19` rows)**: `Exact duplicate row` (`4`), `Invalid Product_ID` (`4`), `Ship_Date before Order_Date` (`4`), `Discount outside [0, 1]` (`3`), `Invalid or missing Customer_ID` (`2`), `Quantity <= 0 or non-numeric` (`2`).

---

## How to Run

From either the repository root or `Data After cleaning/`:

```bash
jupyter nbconvert --to notebook --execute --inplace "Data After cleaning/cleaning.ipynb"
```
