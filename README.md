# Customer Orders Data Cleaning

> **51,000 rows → 49,700 clean rows. 12 columns → 15 enriched columns. 8 distinct data quality issues resolved in a single pipeline.**
>
> A production-grade data cleaning pipeline on a real-world e-commerce orders dataset, built with Python, Pandas, and automated EDA profiling.

---

## What Was Wrong With This Data

Before writing a single line of cleaning code, automated profiling exposed **8 distinct quality problems** across the 12-column dataset:

| Problem | Details |
|---|---|
| **Duplicate rows** | 1,000 exact duplicates (2.0% of dataset) |
| **Missing values** | 11,186 cells across 5 columns |
| **Inconsistent categories** | 16 variants of 5 category values (`ELECTRONICS`, `electronics`, `clothing`, `clothing `, etc.) |
| **Inconsistent status** | 9 variants of 4 status values (`shipped`, `DELIVERED`, `cancelled`, etc.) |
| **Inconsistent countries** | 15+ country representations (`US`, `U.S.A`, `USA`, `AUS`, `U.K.`, `DE`, etc.) |
| **Currency-contaminated prices** | `unit_price` stored as string with mixed `$`, `£`, and `,` formatting (e.g. `$1,920.61`, `£174.02`) |
| **Multi-format dates** | `order_date` written in 6+ formats (`2024-04-25`, `25 Oct 2022`, `February 20, 2022`, `04/05/2022`, `04-29-2022`) |
| **Boolean chaos** | `is_returned` stored as 12 string variants (`TRUE`, `True`, `1`, `Yes`, `Y`, `no`, `FALSE`, `0`, etc.) |

---

## Dataset Overview (Raw)

Profiled with **fg-data-profiling** before any cleaning to expose all quality issues upfront.

| Metric | Value |
|---|---|
| Rows | 51,000 |
| Columns | 12 |
| Missing cells | 11,186 (1.8%) |
| Duplicate rows | 1,000 (2.0%) |
| Total size in memory | 28.6 MiB |

**Variable types (raw):**

| Type | Count |
|---|---|
| Text / Object | 9 |
| Numeric | 2 |
| Date-like (stored as string) | 1 |

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

## The Full Cleaning Pipeline

### Step 1 — Automated EDA

Generated a full HTML profile report on the raw data before touching anything. This revealed the exact missing counts, duplicate count, value distributions, and all 8 quality issues listed above — preventing guesswork.

```python
from data_profiling import ProfileReport

profile = ProfileReport(df, title="Profiling Report", explorative=True)
profile.to_file('report.html')
```

### Step 2 — Standardize Column Names

```python
df.columns = (
    df.columns
    .str.replace(' ', '_')
    .str.lower()
    .str.strip()
)
```

Strips whitespace, lowercases, replaces spaces with underscores. This runs first — before any column access — to prevent silent bugs from casing or whitespace inconsistencies.

### Step 3 — Remove Duplicates

```python
n_dupes = df.duplicated().sum()          # → 1,000
df[df.duplicated(keep=False)].sort_values(by=list(df.columns)).head(10)  # inspect first
df = df.drop_duplicates()
# df.shape → (50,000, 12)
```

Verified duplicates visually before dropping. Exact count: **1,000 rows removed**.

### Step 4 — Handle Missing Values

| Column | Missing % | Decision | Reasoning |
|---|---|---|---|
| `category` | 5.98% | Filled → `'Unknown'` | Preserve rows; `'Unknown'` is a valid categorical label downstream |
| `country` | 4.98% | Filled → `'Unknown'` | Cannot infer from other fields; flagging is more honest than imputing |
| `customer_id` | 3.99% | Filled → `'Unknown'` | Transaction data still valid; rows would be lost otherwise |
| `unit_price` | 3.99% | Kept as `NaN` | Cannot safely impute financial data — left for downstream handling |
| `total_amount` | 3.00% | Filled → median | Temporary fill; will be fully recalculated in Step 10 |

```python
# Numeric
df['total_amount'] = df['total_amount'].fillna(df['total_amount'].median())

# Categorical
categorical_cols = ['customer_id', 'country', 'category']
df[categorical_cols] = df[categorical_cols].fillna('Unknown')
```

### Step 5 — Normalize Categorical Values

The `category`, `status`, and `country` columns contained inconsistent casing, abbreviations, and whitespace. All variants were mapped to canonical values.

```python
for col in ['category', 'status', 'country']:
    df[col] = df[col].str.strip().str.lower()

category_map = {
    'electronics': 'Electronics', 'clothing': 'Clothing',
    'home & garden': 'Home & Garden', 'home and garden': 'Home & Garden',
    'sports': 'Sports', 'books': 'Books', 'unknown': 'Unknown'
}

status_map = {
    'pending': 'Pending', 'shipped': 'Shipped',
    'delivered': 'Delivered', 'cancelled': 'Cancelled'
}

country_map = {
    'united states': 'United States', 'us': 'United States',
    'u.s.a': 'United States', 'usa': 'United States',
    'united kingdom': 'United Kingdom', 'uk': 'United Kingdom', 'u.k.': 'United Kingdom',
    'australia': 'Australia', 'aus': 'Australia',
    'canada': 'Canada', 'germany': 'Germany', 'de': 'Germany',
    'france': 'France', 'fr': 'France', 'unknown': 'Unknown'
}
```

Result: **16 category variants → 6 canonical · 9 status variants → 4 canonical · 15 country variants → 6 canonical**

### Step 6 — Fix Currency-Contaminated Prices

`unit_price` was stored as a mixed string field containing dollar signs, pound signs, and thousands-separator commas.

```python
df['unit_price'] = (
    df['unit_price']
    .astype(str)
    .str.replace(r'[$£,]', '', regex=True)
    .str.strip()
)
df['unit_price'] = pd.to_numeric(df['unit_price'], errors='coerce')
```

Examples fixed: `$1,920.61` → `1920.61` · `£174.02` → `174.02` · `$64.81` → `64.81`

### Step 7 — Standardize Boolean Field

`is_returned` contained 12 distinct string representations of True/False.

```python
bool_map = {
    'Yes': True, 'yes': True, 'Y': True, 'y': True, 'TRUE': True,
    'True': True, 'true': True, '1': True, 1: True, True: True,
    'No': False, 'no': False, 'N': False, 'n': False, 'FALSE': False,
    'False': False, 'false': False, '0': False, 0: False, False: False
}
df['is_returned'] = df['is_returned'].map(bool_map).fillna(False)
```

Result: **12 variants → Python native `bool` dtype**

### Step 8 — Multi-Format Date Parsing

`order_date` was written in at least 6 different formats. A three-pass strategy resolved all of them.

```python
# Pass 1: standard ISO / slash formats
df['order_date'] = pd.to_datetime(df['order_date'], dayfirst=False, errors='coerce')

# Pass 2: MM-DD-YYYY format (e.g. 04-29-2022)
mask = df['order_date'].isna()
df.loc[mask, 'order_date'] = pd.to_datetime(
    df_raw_dates.loc[mask, 'order_date_raw'], format='%m-%d-%Y', errors='coerce'
)

# Pass 3: text month formats (e.g. "February 20, 2022", "25 Oct 2022")
mask2 = df['order_date'].isna()
df.loc[mask2, 'order_date'] = pd.to_datetime(
    df_raw_dates.loc[mask2, 'order_date_raw'], format='mixed', dayfirst=True, errors='coerce'
)
```

Formats resolved: `2024-04-25` · `25 Oct 2022` · `February 20, 2022` · `04/05/2022` · `04-29-2022` · `26/05/2023`

### Step 9 — Date Feature Engineering

After parsing, three new analytical columns were derived directly from `order_date`.

```python
df['order_year']        = df['order_date'].dt.year
df['order_month']       = df['order_date'].dt.month
df['order_day_of_week'] = df['order_date'].dt.day_name()
```

These columns enable time-series aggregation and day-of-week analysis without requiring downstream consumers to re-parse dates.

### Step 10 — Remove Zero-Quantity Rows & Recalculate Totals

```python
# Drop zero-quantity rows (no valid transaction)
df = df[df['quantity'] != 0].copy()

# Recalculate total_amount from source columns
df['total_amount'] = df['quantity'] * df['unit_price']
```

Zero-quantity rows represent data entry errors with no business meaning. Recalculating `total_amount` from `quantity × unit_price` overwrites the median-filled placeholder from Step 4 with ground-truth values wherever both source columns are non-null.

---

## Before vs. After

| Metric | Before | After |
|---|---|---|
| Rows | 51,000 | 49,700 |
| Columns | 12 | 15 (3 date features added) |
| Duplicate rows | 1,000 | 0 |
| Zero-quantity rows | ~300 | 0 |
| Missing cells | 11,186 | 3,978 (`unit_price` + `total_amount` only) |
| `unit_price` dtype | `str` (with `$`, `£`, `,`) | `float64` |
| `is_returned` dtype | `str` (12 variants) | `bool` |
| Category variants | 16 | 6 canonical |
| Country variants | 15+ | 6 canonical |
| Status variants | 9 | 4 canonical |

---

## Tools & Stack

| Tool | Version | Purpose |
|---|---|---|
| Python | 3.x | Core language |
| Pandas | latest | Data manipulation and cleaning |
| NumPy | latest | Numeric operations |
| fg-data-profiling | latest | Automated EDA — HTML profile report |
| Jupyter Notebook | — | Interactive development |
| Google Colab | — | Cloud execution environment |

---

## Repository Structure

```
├── data_cleaning.ipynb           # Full cleaning notebook — all 10 steps
├── messy_customer_orders.csv     # Raw dataset (51,000 rows, 12 columns)
├── clean_customer_orders.csv     # Clean output (49,700 rows, 15 columns)
├── report.html                   # Profiling report (pre-cleaning)
└── README.md
```

---

## Key Takeaways

- **Profile before you clean.** Running the profiler on raw data before writing any code exposed all 8 quality issues at once — including currency symbols in `unit_price` and 12 boolean variants in `is_returned` that wouldn't show up in a quick `.head()` scan.
- **Data types are part of data quality.** Fixing dtypes (`str → float`, `str → bool`, `str → datetime`) is not just cosmetic — it prevents downstream aggregation bugs and enables correct pandas operations.
- **Multi-pass date parsing beats single-format assumptions.** A single `pd.to_datetime()` call failed on 3+ formats silently. Three targeted passes resolved all date variants without data loss.
- **Feature engineering is part of the cleaning deliverable.** Adding `order_year`, `order_month`, and `order_day_of_week` directly in the cleaning step reduces friction for every downstream consumer of this dataset.
- **Recalculate derived columns from source columns, not placeholders.** `total_amount` was first filled with a median to avoid losing rows, then fully recomputed from `quantity × unit_price` once both source columns were clean.

---

## Author

**Mohamed Jawad Touir** — Big Data Student @ EPI Digital School | Aspiring Data Engineer / ML Engineer

[GitHub](https://github.com/JINZO-AI) · [LinkedIn](https://www.linkedin.com/in/jawad-touir)
