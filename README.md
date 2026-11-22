# Scraping & Data Cleaning Pipeline

End-to-end Python data engineering project covering two productionized workflows:

1. **Books Web Scraping (`Scraping Books/`)** — Concurrent, paginated, UTF-8-safe crawler collecting full product metadata for all **1,000 books** across **50 categories** from [books.toscrape.com](https://books.toscrape.com/).
2. **E-Commerce Data Cleaning (`Data After cleaning/`)** — Multi-table validation, sibling-order recovery, transparent numeric imputation, quarantine auditing, and KPI enrichment across **5 raw CSV datasets** in `Data Before cleaning/`.

---

## Repository Structure

```text
.
├── README.md
├── .gitignore
├── Scraping Books/
│   ├── README.md
│   ├── scraping.ipynb                  # Concurrent paginated + detail-page scraper
│   ├── books.csv                       # 1,000 validated books (13 typed columns)
│   └── scraping_summary.json           # Run audit, price/rating stats & validation checks
├── Data Before cleaning/
│   ├── customers_master.csv            # 150 raw customer records
│   ├── products_master.csv             # 30 raw product catalog records
│   ├── previous_days_summary.csv       # 30 daily summary rows (2025-09-01..2025-09-30)
│   ├── orders.csv                      # 1,020 raw October 2025 order lines
│   └── new_daily_batch.csv             # 333 raw November 2025 daily batch lines
└── Data After cleaning/
    ├── README.md
    ├── cleaning.ipynb                  # Modular cleaning, recovery, quarantine & KPI pipeline
    ├── customers_master_cleaned.csv    # 150 cleaned customer records
    ├── products_master_cleaned.csv     # 30 cleaned products + Stock_Alert & Units_Below_Reorder
    ├── previous_days_summary_cleaned.csv # 30 validated Sept 2025 summary records
    ├── orders_cleaned.csv              # 1,011 cleaned & KPI-enriched order lines
    ├── new_daily_batch_cleaned.csv     # 314 cleaned & KPI-enriched batch lines
    ├── rejected_records.csv            # 28 quarantined rows with Source_File & Rejection_Reason
    └── cleaning_summary.csv            # Dataset-level audit metrics & financial totals
```

---

## Key Results Summary

### 1. Book Scraping (`Scraping Books/`)
- **Coverage**: Upgraded from 517 books (first page per category only) to all **1,000 unique books** across **50 categories** (`0` scrape errors).
- **Encoding & Schema**: Fixed UTF-8 mojibake (`Â£`, `â€™`, `Ã©`, etc.) -> **`0` mojibake hits**; expanded from 4 raw string columns to **13 typed fields** (`UPC`, `Title`, `Category`, `Rating`, `Price_Excl_Tax`, `Price_Incl_Tax`, `Tax`, `In_Stock`, `Stock_Quantity`, `Num_Reviews`, `Description`, `Product_URL`, `Image_URL`).
- **Inventory & Pricing**: **8,585** total units in stock; prices range from **£10.00** to **£59.99** (median **£35.98**, mean **£35.07**).

### 2. Data Cleaning (`Data After cleaning/`)

| Dataset | Raw Rows | Duplicates Removed | Sibling Recoveries | Numeric Imputations | Ship Date Cleared | Ship Date Anomalies | Rejected | Clean Rows |
|---|---:|---:|---:|---:|---:|---:|---:|---:|
| `customers_master` | 150 | 0 | 0 | 0 | 0 | 0 | 0 | **150** |
| `products_master` | 30 | 0 | 0 | 0 | 0 | 0 | 0 | **30** |
| `previous_days_summary` | 30 | 0 | 0 | 0 | 0 | 0 | 0 | **30** |
| `orders.csv` | 1,020 | 2 | 4 | 2 | 0 | 1 | 9 | **1,011** |
| `new_daily_batch.csv` | 333 | 4 | 11 | 7 | 60 | 5 | 19 | **314** |

- **Referential Integrity**: Validates both `Product_ID` (`PX01`, `PX02`, `P555`, `P666`) and `Customer_ID` (`C5555`, `C6666`) against master tables after unambiguous sibling `Order_ID` recovery.
- **Quarantine Audit**: All **28** rejected records are preserved in `Data After cleaning/rejected_records.csv` with original row numbers and explicit rejection reasons rather than being silently dropped.

---

## Tech Stack & Running the Notebooks

- **Python**: 3.12+
- **Libraries**: `requests`, `urllib3`, `beautifulsoup4`, `lxml`, `pandas`, `numpy`, `jupyter` (`nbconvert` / `ipykernel`)

Both notebooks resolve relative paths automatically whether executed from the repository root or inside their respective folders:

```bash
# Run data cleaning pipeline
jupyter nbconvert --to notebook --execute --inplace "Data After cleaning/cleaning.ipynb"

# Run concurrent book scraper
jupyter nbconvert --to notebook --execute --inplace "Scraping Books/scraping.ipynb"
```

For deep-dive schemas, design decisions, and anomaly breakdowns, see:
- [Scraping Books/README.md](Scraping%20Books/README.md)
- [Data After cleaning/README.md](Data%20After%20cleaning/README.md)
