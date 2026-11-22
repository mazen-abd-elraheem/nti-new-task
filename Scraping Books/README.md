# Scraping Books — Concurrent Paginated Web Scraper

## Overview

`scraping.ipynb` crawls [books.toscrape.com](https://books.toscrape.com/) end-to-end across all **50 leaf categories**, follows every paginated listing page, and concurrently fetches every book's detail page to build a clean, validated catalog of all **1,000 books** in `books.csv` along with an execution audit in `scraping_summary.json`.

---

## Problems in Original Script & How They Were Solved

| Problem Found | Original Behavior | Upgraded Solution |
|---|---|---|
| **Incomplete Crawl (517 / 1,000 books)** | Read only page 1 of each category, skipping paginated books. | Follows `li.next a` across every category page until exhaustion, capturing all **1,000** books across **50** categories. |
| **UTF-8 Mojibake (`Â£`, `â€™`, `Ã©`, `â€œ`)** | Relied on `requests`' `ISO-8859-1` fallback and used `.replace("Â", "")` on prices, leaving book titles corrupted (`Noahâ€™s`, `CafÃ©`, `Pangur BÃ¡n`). | Parses raw `response.content` bytes with `BeautifulSoup(..., "lxml", from_encoding="utf-8")` and verifies `mojibake_hits == 0`. |
| **Missing Import & Brittle URLs** | Omitted `import csv` (raising `NameError` on a fresh run) and concatenated URLs with `url + cat["href"]`. | Self-contained imports and `urllib.parse.urljoin()` for category, pagination, detail, and cover image URLs. |
| **No HTTP Resilience or Concurrency** | Bare sequential `requests.get()` with no timeout, retries, or User-Agent. | Thread-local `requests.Session` pool (`MAX_WORKERS = 12`), `15s` timeout, `urllib3.Retry` (4 retries with exponential backoff on 429/5xx), and structured logging. |
| **Shallow Untyped Schema** | Saved 4 string columns (`Category, Name, Price, In_Stock`), missing stock counts, ratings, UPCs, and descriptions. | Crawls every book detail page for 13 typed columns (`UPC`, `Rating`, `Price_Excl_Tax`, `Price_Incl_Tax`, `Tax`, `In_Stock`, `Stock_Quantity`, `Num_Reviews`, `Description`, `Product_URL`, `Image_URL`) and deduplicates by `UPC`. |

---

## Output Files & Schema

### `books.csv` (1,000 rows, UTF-8)

| Column | Type | Description |
|---|---|---|
| `UPC` | `str` | Unique 16-character product identifier from detail table |
| `Title` | `str` | Full un-truncated book title (`h1` on detail page) |
| `Category` | `str` | Book category from left navigation tree (50 categories) |
| `Rating` | `int` | Star rating mapped to `1..5` |
| `Price_Excl_Tax` | `float` | Price excluding tax (GBP) |
| `Price_Incl_Tax` | `float` | Price including tax (GBP) |
| `Tax` | `float` | Tax amount (GBP) |
| `In_Stock` | `bool` | Availability flag (`True` / `False`) |
| `Stock_Quantity` | `int` | Exact number of available copies parsed from `"In stock (N available)"` |
| `Num_Reviews` | `int` | Number of customer reviews |
| `Description` | `str` | Product description text (empty string if absent on site) |
| `Product_URL` | `str` | Canonical book detail page URL |
| `Image_URL` | `str` | Absolute cover image URL |

### `scraping_summary.json`

Machine-readable summary of the latest crawl run, category/stock totals, price summary statistics, rating counts, and validation checks.

---

## Verified Crawl Results

- **Execution Runtime**: `27.62s` (`12` worker threads, `0` HTTP/parse errors)
- **Books Exported**: `1,000` unique `UPC`s across `50` categories
- **Stock**: `1,000` in stock (`8,585` total available units)
- **Price (incl. tax, GBP)**: Min `10.00`, Median `35.98`, Mean `35.07`, Max `59.99`
- **Rating Distribution**: `1★: 226`, `2★: 196`, `3★: 203`, `4★: 179`, `5★: 196`
- **Data Notes**:
  - `2` books genuinely have no product description block on the source site.
  - `2` records share the title *The Star-Touched Queen* in `Fantasy` with distinct `UPC`s and prices (`46.02` vs `32.30`), preserved cleanly via `UPC` deduplication.
- **Validation Checks**: `mojibake_hits: 0`, `missing_upc: 0`, `invalid_ratings: 0`, `invalid_prices: 0`, `scrape_error_count: 0`.

---

## How to Run

From either the repository root or `Scraping Books/`:

```bash
jupyter nbconvert --to notebook --execute --inplace "Scraping Books/scraping.ipynb"
```
