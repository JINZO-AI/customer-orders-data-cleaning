# Customer Orders Data Cleaning

> **51,000 rows. 12 columns. 11,186 missing cells. 1,000 duplicates. One clean dataset.**  
> A full exploratory and cleaning pipeline on a real-world e-commerce orders dataset using Python, Pandas, and YData Profiling.

---

## Dataset Overview (Raw)

Profiled with **YData Profiling v4.19.1** before any cleaning.

| Metric | Value |
|---|---|
| Rows | 51,000 |
| Columns | 12 |
| Missing cells | 11,186 (1.8%) |
| Duplicate rows | 1,000 (2.0%) |
| Total size in memory | 28.6 MiB |

**Variable types:**

| Type | Count |
|---|---|
| Text | 3 |
| DateTime | 1 |
| Categorical | 6 |
| Numeric | 2 |

---

## Missing Values (Before Cleaning)

```
category        5.98%
country         4.98%
customer_id     3.99%
unit_price      3.99%
total_amount    3.00%
```

---

## Cleaning Steps

### 1. Automated EDA — YData Profiling
Generated a full HTML profile report before touching the data to understand distributions, missing patterns, correlations, and alerts.

```python
from ydata_profiling import ProfileReport
profile = ProfileReport(df, title="Profiling Report", explorative=True)
profile.to_file('report.html')
```

### 2. Standardize Column Names
```python
df.columns = (
    df.columns
    .str.replace(' ', '_')
    .str.lower()
    .str.strip()
)
```
Stripped whitespace, lowercased, replaced spaces with underscores for consistent programmatic access.

### 3. Remove Duplicates
```python
n_dupes = df.duplicated().sum()  # → 1,000
df = df.drop_duplicates()
df.shape  # → (50,000, 12)
```
Identified and removed **exactly 1,000 fully duplicated rows** (2.0% of the dataset), confirmed by `.duplicated(keep=False)` inspection before dropping.

### 4. Handle Missing Values

| Column | Missing % | Decision | Reasoning |
|---|---|---|---|
| `category` | 5.98% | Kept as `NaN` | Cannot infer category from other columns without a domain lookup table |
| `country` | 4.98% | Kept as `NaN` | No basis for imputation; flagging is more honest |
| `customer_id` | 3.99% | Kept as `NaN` | Cannot assign IDs — rows still contain valid transaction data |
| `unit_price` | 3.99% | Kept as `NaN` | Imputing price would corrupt any financial analysis |
| `total_amount` | 3.00% | Kept as `NaN` | Cannot calculate without `unit_price`; left intentionally missing |

**Deliberate decision:** No imputation was applied. Filling missing values with means or modes would introduce false precision into a financial dataset. Missing is marked as missing.

---

## Before vs. After

| Metric | Before | After |
|---|---|---|
| Rows | 51,000 | 50,000 |
| Duplicate rows | 1,000 | 0 |
| Missing cells | 11,186 | Preserved (not imputed) |

---

## Tools & Stack

| Tool | Purpose |
|---|---|
| Python 3 | Core language |
| Pandas | Data manipulation |
| NumPy | Numeric operations |
| YData Profiling | Automated EDA |
| Jupyter Notebook | Interactive environment |
| Google Colab | Cloud execution |

---

## Files

```
├── data_cleaning.ipynb           # Full cleaning notebook
├── messy_customer_orders.csv     # Raw dataset (original)
├── clean_customer_orders.csv     # Cleaned output
└── README.md
```

---

## Key Learnings

- **Profile before you clean.** YData Profiling revealed the full missing pattern and exact duplicate count before writing a single cleaning line — this prevents guessing.
- **Exact numbers matter.** There were exactly 1,000 duplicates. Precision in documentation reflects precision in the work.
- **Restraint is a cleaning decision.** Not imputing missing financial values (`unit_price`, `total_amount`) was deliberate — it preserves data integrity for any downstream analyst.
- **Column naming is cleaning too.** Standardizing column names before any transformation prevents silent bugs from whitespace or casing inconsistencies.

---

## Author

**Mohamed Jawad Touir** — Big Data Student @ EPI | Aspiring ML/AI Engineer  
[GitHub](https://github.com/JINZO-AI)
