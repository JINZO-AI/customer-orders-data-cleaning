# Customer Orders Data Cleaning

> **Python & Pandas pipeline that transformed a 50,000-row e-commerce dataset full of real-world quality issues into an analysis-ready asset.**

---

## Project Overview

This project simulates the kind of messy, real-world data analysts encounter in production environments. The raw dataset contained multiple overlapping issues across duplicates, date formats, inconsistent country names, financial calculations, and missing values. The goal was not just to clean the data, but to document every decision with a clear rationale — treating it as a production-grade audit trail.

---

## Dataset

| | Raw | Clean |
|---|---|---|
| **Rows** | ~50,000 | 49,700 |
| **Columns** | 15 | 15 |
| **Duplicates** | ~300 | 0 |
| **Zero-quantity rows** | ~300 | 0 |
| **Corrupted `total_amount`** | Yes | Recalculated |
| **NaT in `order_date`** | Yes | 0 |
| **Inconsistent country names** | Yes (`u.s.a`, `usa`, `uk`...) | Standardized |
| **Blank date feature columns** | Yes | Recalculated from source |

---

## Data Quality Issues & Decisions

### 1. Duplicate Removal
Detected and removed ~300 full duplicate rows. Retained the first occurrence of each, as there was no basis to prefer one over another.

### 2. Date Standardization
The `order_date` column had mixed formats (e.g. `2024-01-15`, `15/01/2024`, `January 15 2024`). Used `pd.to_datetime(utc=True, errors='coerce')` to parse all variants and standardize to UTC. Result: 0 NaT remaining.

### 3. Zero-Quantity Rows
Rows with `quantity == 0` represent no transaction — not returns, not sales. Dropped completely.

### 4. Financial Recalculation
Detected that the original `total_amount` column contained corrupted values (not equal to `quantity × unit_price`). Recalculated from scratch rather than patching individual outliers, ensuring full consistency across the dataset.

### 5. Return Transactions (Negative Quantity)
200 rows with `quantity < 0` were identified as returns. These were retained deliberately — removing them would skew revenue analysis and hide real business behaviour. Negative `total_amount` on returns is correct in accounting terms.

### 6. Missing `unit_price` (Deliberate Decision)
1,989 rows have no recorded unit price. These were kept as `NaN` intentionally. Imputing with median or mean would introduce bias into revenue calculations — the correct approach is to honour the missing data and let downstream analysis handle it explicitly.

### 7. Country Name Standardization
The `country` column contained 2,962 rows with `u.s.a` (and variants like `usa`, `u.s.a.`) that were not caught by a simple string match. Applied `.str.strip().str.lower()` before mapping to catch all edge cases. Result: `United States` count went from 8,933 → 11,895.

```
United States    11,895   ← merged 8,933 + 2,962 variants
Germany           9,011
Australia         8,796
Canada            8,779
United Kingdom    8,733
Unknown           2,486   ← retained as-is (genuinely missing)
```

### 8. Date Feature Columns Recalculated
`order_year`, `order_month`, and `order_day_of_week` were derived columns that contained blanks because they were computed before date standardization. Recalculated from the clean `order_date` column. Result: 0 NaN remaining.

---

## Tech Stack

- **Python 3.x**
- **Pandas** -- core data manipulation
- **NumPy** -- vectorised operations
- **Google Colab** -- development environment

---

## Files

```
.
├── data_cleaning.ipynb            # Full cleaning pipeline with documented rationale
├── messy_customer_orders.csv      # Original raw dataset
└── clean_customer_orders.csv      # Final cleaned output
```

---

## Final Data Quality State

```
Shape:                  49,700 rows × 15 columns
NaT in order_date:      0
NaN in unit_price:      1,989  (intentional -- see decision above)
NaN in order_year:      0
Negative quantity:      200    (returns -- retained deliberately)
Duplicates:             0
u.s.a variants:         0      (all merged into United States)
```

---

## Key Learnings

- **Corrupted derived columns** should always be recalculated from source columns, not patched
- **Missing data is not always a problem** -- imputing unknown prices would be worse than the missing values
- **Business context matters** -- negative quantities are valid returns, not errors
- **String matching is fragile** -- always normalise case and whitespace before applying a value map
- **Derived columns go stale** -- any column computed from another must be recalculated after the source is cleaned
- **Audit trails** -- every decision is documented inline in the notebook with a rationale

---

## Author

**Mohamed Jawad Touir** -- Big Data Student @ EPI | Aspiring ML/AI Engineer

[GitHub](https://github.com/JINZO-AI)